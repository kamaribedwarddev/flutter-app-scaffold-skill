# Supabase Setup Reference

Complete guide for setting up Supabase as the backend for a scaffolded Flutter app.

## Table of Contents
1. [Initial Setup (Manual Steps)](#initial-setup)
2. [Local Development](#local-development)
3. [Migration Patterns](#migration-patterns)
4. [Row Level Security](#row-level-security)
5. [RPC Functions](#rpc-functions)
6. [Edge Functions](#edge-functions)
7. [Realtime Subscriptions](#realtime-subscriptions)
8. [Flutter Client Configuration](#flutter-client-configuration)

---

## Initial Setup

### ⏸️ MANUAL STEP: Create Supabase Project

Tell the user:

```
⏸️ MANUAL STEP REQUIRED: Create Supabase Project

1. Go to https://supabase.com and sign up / log in
2. Click "New Project"
3. Choose an organization (or create one)
4. Set a project name (use your app name)
5. Set a strong database password — save it somewhere safe
6. Choose a region close to your users
7. Click "Create new project" and wait for it to provision (~2 minutes)

Once ready, I need two values from Project Settings → API:
- **Project URL** (looks like https://abcdefgh.supabase.co)
- **anon/public key** (starts with eyJ...)

Paste them here when you have them.
```

After receiving the values, update `.env.prod`:
```
SUPABASE_URL=<their-project-url>
SUPABASE_ANON_KEY=<their-anon-key>
```

### ⏸️ MANUAL STEP: Local Development Setup

Tell the user:

```
⏸️ MANUAL STEP REQUIRED: Set Up Local Supabase

Local development requires Docker. Here's what to do:

1. Install Docker Desktop if you don't have it: https://docker.com/products/docker-desktop
2. Make sure Docker is running (you should see the whale icon in your menu bar/taskbar)
3. Let me know when Docker is running, and I'll initialize Supabase locally.
```

After confirmation, run:
```bash
supabase init    # Creates supabase/ directory with config.toml
supabase start   # Starts local Supabase (first run downloads images, takes a few minutes)
```

Capture the output — it contains the local `API URL` and `anon key`. Update `.env.dev` with these values.

### Optional: Supabase MCP Server Setup

If the user opted for the Supabase MCP server during discovery, set it up after local Supabase is running. The MCP server gives Claude direct access to the Supabase project — it can run SQL, apply migrations, list tables, deploy Edge Functions, and more, without the user needing to copy-paste between the dashboard and the terminal.

**Step 1: Check prerequisites**

The MCP server requires Node.js (for `npx`). Check:
```bash
node --version  # Should be 18+
```

If not installed, tell the user:
```
You'll need Node.js installed to run the Supabase MCP server.
Install it from https://nodejs.org (LTS version recommended).
Let me know when it's installed.
```

**Step 2: Create `.mcp.json`**

Create a `.mcp.json` file in the project root with the Supabase MCP server configuration.

For **local development** (connects to local Supabase started via Docker):
```json
{
  "mcpServers": {
    "supabase": {
      "command": "npx",
      "args": [
        "-y",
        "@supabase/mcp-server-supabase@latest",
        "--read-only=false",
        "--project-ref=local",
        "--api-url=http://127.0.0.1:54331/mcp",
        "--db-url=postgresql://postgres:postgres@127.0.0.1:54332/postgres"
      ]
    }
  }
}
```

For **production** (connects to cloud Supabase project — use the user's project ref):
```json
{
  "mcpServers": {
    "supabase": {
      "command": "npx",
      "args": [
        "-y",
        "@supabase/mcp-server-supabase@latest",
        "--read-only=false",
        "--project-ref=<their-project-ref>"
      ]
    }
  }
}
```

The project ref is the subdomain from the Supabase URL (e.g., `abcdefgh` from `https://abcdefgh.supabase.co`). For production, the MCP server authenticates via the Supabase CLI login (`supabase login`).

**Step 3: Enable in Claude Code settings**

Add to `.claude/settings.local.json` in the project:
```json
{
  "enableAllProjectMcpServers": true,
  "enabledMcpjsonServers": ["supabase"]
}
```

**Step 4: Verify**

After restarting Claude Code (or starting a new conversation), the Supabase MCP tools should be available. Test by listing tables:
- Use the `mcp__supabase__list_tables` tool to verify connectivity.

**What the MCP server enables:**

Once set up, you can use MCP tools throughout the build instead of writing raw SQL files and running them manually:
- `execute_sql` — Run queries directly against the database
- `apply_migration` — Create and apply numbered migration files
- `list_tables` — Inspect the current schema
- `list_migrations` — See which migrations have been applied
- `deploy_edge_function` — Deploy Edge Functions directly
- `get_logs` — Check Edge Function and API logs
- `list_extensions` — See installed Postgres extensions

This is especially useful during the database schema phase (Phase 2) and any phase that involves writing migrations or Edge Functions — Claude can apply changes directly and verify them without the user running CLI commands.

**Note:** Start with the local configuration during development. Switch to the production configuration when deploying to cloud Supabase.

---

## Local Development

### Key Commands

```bash
supabase start          # Start local Supabase (Docker must be running)
supabase stop           # Stop local Supabase
supabase db reset       # Reset DB: drops everything, replays all migrations + seed
supabase status         # Show local URLs and keys
```

### Important Rules

- **Never run `supabase db reset` on production** — it deletes all data
- **Never make database changes directly** — always create migration files
- **Apply migrations individually** during development — don't reset unless starting fresh
- Local Supabase Studio is available at `http://localhost:54323` for visual DB management

---

## Migration Patterns

### Naming Convention

Migrations are numbered by date and sequence:
```
YYYYMMDD000XXX_description.sql
```

Example sequence:
```
20250415000001_initial_schema.sql
20250415000002_rls_policies.sql
20250415000003_functions_triggers.sql
20250415000004_seed_reference_data.sql
20250416000001_add_user_preferences.sql
```

### Initial Schema Migration

The first migration creates all core tables. Design the schema based on the app's data model identified during discovery.

```sql
-- Example pattern for a core table
CREATE TABLE IF NOT EXISTS public.people (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    name TEXT NOT NULL,
    relationship TEXT,
    notes TEXT,
    avatar_url TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);

-- Enable RLS on every table
ALTER TABLE public.people ENABLE ROW LEVEL SECURITY;
```

### Common Patterns

**Auto-update timestamps:**
```sql
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_people_updated_at
    BEFORE UPDATE ON public.people
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

**Auto-create profile on signup:**
```sql
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO public.profiles (id, email, display_name)
    VALUES (
        NEW.id,
        NEW.email,
        COALESCE(NEW.raw_user_meta_data->>'name', split_part(NEW.email, '@', 1))
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
    AFTER INSERT ON auth.users
    FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

**Generate unique codes (for invites, sharing):**
```sql
CREATE OR REPLACE FUNCTION generate_unique_code(table_name TEXT, column_name TEXT, code_length INT DEFAULT 8)
RETURNS TEXT AS $$
DECLARE
    new_code TEXT;
    code_exists BOOLEAN;
BEGIN
    LOOP
        new_code := upper(substr(md5(random()::text), 1, code_length));
        EXECUTE format('SELECT EXISTS(SELECT 1 FROM %I WHERE %I = $1)', table_name, column_name)
            INTO code_exists USING new_code;
        EXIT WHEN NOT code_exists;
    END LOOP;
    RETURN new_code;
END;
$$ LANGUAGE plpgsql;
```

### Applying Migrations

```bash
# Apply a new migration to local DB
supabase db push

# Apply to production (via Supabase CLI linked to project)
supabase db push --linked
```

---

## Row Level Security

Every table must have RLS enabled. Common policy patterns:

### User-Owned Data (most common)
```sql
-- Users can only see their own data
CREATE POLICY "Users can view own data"
    ON public.people FOR SELECT
    USING (user_id = auth.uid());

CREATE POLICY "Users can insert own data"
    ON public.people FOR INSERT
    WITH CHECK (user_id = auth.uid());

CREATE POLICY "Users can update own data"
    ON public.people FOR UPDATE
    USING (user_id = auth.uid())
    WITH CHECK (user_id = auth.uid());

CREATE POLICY "Users can delete own data"
    ON public.people FOR DELETE
    USING (user_id = auth.uid());
```

### Shared Data (e.g., group/team content)
```sql
-- Members can view group data
CREATE POLICY "Members can view group items"
    ON public.group_items FOR SELECT
    USING (
        EXISTS (
            SELECT 1 FROM public.group_members
            WHERE group_id = group_items.group_id
            AND user_id = auth.uid()
        )
    );
```

### Public Read, Authenticated Write
```sql
CREATE POLICY "Anyone can read"
    ON public.categories FOR SELECT
    USING (true);

CREATE POLICY "Authenticated users can insert"
    ON public.categories FOR INSERT
    WITH CHECK (auth.uid() IS NOT NULL);
```

---

## RPC Functions

All client-side data access should go through RPC functions rather than direct table queries. This provides a clean API layer and allows complex queries that combine multiple tables.

### Pattern

```sql
CREATE OR REPLACE FUNCTION get_people_for_user()
RETURNS JSONB AS $$
BEGIN
    RETURN COALESCE(
        (SELECT jsonb_agg(
            jsonb_build_object(
                'id', p.id,
                'name', p.name,
                'relationship', p.relationship,
                'interests', (
                    SELECT COALESCE(jsonb_agg(i.name), '[]'::jsonb)
                    FROM people_interests pi
                    JOIN interests i ON i.id = pi.interest_id
                    WHERE pi.person_id = p.id
                )
            )
        )
        FROM people p
        WHERE p.user_id = auth.uid()
        ORDER BY p.name),
        '[]'::jsonb
    );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### When to Use SECURITY DEFINER

- **SECURITY DEFINER**: Function runs as the function creator (bypasses RLS). Use for:
  - Cross-user queries (checking group membership)
  - Operations that need to read/write across user boundaries
  - Complex aggregations that RLS would block

- **SECURITY INVOKER** (default): Function runs as the calling user. Use for:
  - Simple queries where RLS already handles access correctly
  - Operations that should respect the user's permissions

### Calling from Flutter

```dart
// Simple call
final response = await supabase.rpc('get_people_for_user');

// With parameters
final response = await supabase.rpc('create_person', params: {
  'p_name': name,
  'p_relationship': relationship,
});
```

---

## Edge Functions

Edge Functions are TypeScript/Deno functions hosted by Supabase for server-side logic.

### Structure

```
supabase/functions/
├── _shared/             # Shared utilities
│   └── cors.ts          # CORS headers
├── function-name/
│   └── index.ts         # Function entry point
```

### Template

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
}

serve(async (req) => {
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders })
  }

  try {
    // Verify JWT
    const authHeader = req.headers.get('Authorization')
    if (!authHeader) throw new Error('No authorization header')

    const supabase = createClient(
      Deno.env.get('SUPABASE_URL')!,
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!,
    )

    const token = authHeader.replace('Bearer ', '')
    const { data: { user }, error } = await supabase.auth.getUser(token)
    if (error || !user) throw new Error('Invalid token')

    // Your logic here
    const body = await req.json()

    return new Response(JSON.stringify({ success: true }), {
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
    })
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 400,
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
    })
  }
})
```

### Deployment

```bash
# Deploy a single function (always use verify_jwt=false — functions handle their own auth)
supabase functions deploy function-name --no-verify-jwt

# Deploy all functions
supabase functions deploy --no-verify-jwt
```

### Calling from Flutter

```dart
final response = await supabase.functions.invoke(
  'function-name',
  body: {'key': 'value'},
);
```

---

## Realtime Subscriptions

For features that need live updates (chat, shared lists, collaboration):

### Stream Pattern

```dart
// In repository
Stream<List<Item>> getItemsStream(String groupId) {
  return _supabase
      .from('items')
      .stream(primaryKey: ['id'])
      .eq('group_id', groupId)
      .map((data) => data.map((json) => Item.fromJson(json)).toList());
}
```

### Channel Pattern (for presence, broadcast)

```dart
final channel = _supabase.channel('room:$roomId');
channel.onPostgresChanges(
  event: PostgresChangeEvent.all,
  schema: 'public',
  table: 'messages',
  filter: PostgresChangeFilter(
    type: PostgresChangeFilterType.eq,
    column: 'room_id',
    value: roomId,
  ),
  callback: (payload) {
    // Handle insert, update, delete
  },
).subscribe();
```

---

## Flutter Client Configuration

### supabase_config.dart

```dart
import 'package:supabase_flutter/supabase_flutter.dart';

class SupabaseConfig {
  static Future<void> initialize({
    required String url,
    required String anonKey,
  }) async {
    await Supabase.initialize(
      url: url,
      anonKey: anonKey,
      authOptions: const FlutterAuthClientOptions(
        authFlowType: AuthFlowType.pkce,
      ),
    );
  }

  static SupabaseClient get client => Supabase.instance.client;
}
```

### In main.dart

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Env.load();

  await SupabaseConfig.initialize(
    url: Env.supabaseUrl,
    anonKey: Env.supabaseAnonKey,
  );

  runApp(const ProviderScope(child: MyApp()));
}
```
