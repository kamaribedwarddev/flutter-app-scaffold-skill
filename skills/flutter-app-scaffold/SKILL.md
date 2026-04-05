---
name: flutter-app-scaffold
description: Use this skill when the user wants to build a new mobile app from scratch, create a new Flutter project, start a new app from an idea, or asks "where do I start" with a mobile app concept. Guides them from idea to production-ready Flutter app through a phased implementation plan — interviewing about features, setting up clean architecture (Riverpod, GoRouter, Freezed), optional Supabase database with MCP server, authentication, and modular add-ons (gamification, notifications, home widgets, subscriptions). Trigger this skill for any request that involves creating a NEW Flutter app or turning an app idea into code — including vague requests like "I want to build an app for X" or "help me make a mobile app". Do NOT trigger for requests to fix, modify, refactor, or deploy an existing app.
---

# Flutter App Scaffold

You are guiding a user from app idea to production-ready Flutter mobile app. The process is conversational and phased — you interview, plan, build, and iterate together.

## The Big Picture

This skill produces a Flutter app with:
- **Feature-based clean architecture** (data / domain / presentation layers per feature)
- **Riverpod** for state management
- **GoRouter** for navigation with auth guards
- **Freezed** for immutable models with JSON serialization
- **Optional Supabase** backend (Postgres + RLS + Edge Functions)
- **Build flavors** (dev / prod) with environment-specific config
- **A living implementation plan** (`docs/implementation-plan.md`) that tracks every phase

The user's app idea drives everything — features, database schema, theme, onboarding flow. Nothing is copy-pasted from a template; every piece is tailored to their app.

---

## Phase 0: Discovery Interview

Start here every time. Your goal is to gather enough information to generate a complete implementation plan. Ask these questions conversationally — don't dump them all at once. Group related questions naturally.

### Required Information

1. **App idea**: What does the app do? Who is it for? What problem does it solve?
2. **Core features**: What are the 3-5 main things a user can do in the app?
3. **Database**: Does the app need a backend/database? (user accounts, synced data, shared content = yes)
4. **Authentication**: If database — email/password? Social login (Apple, Google)? Both?
5. **Target platforms**: iOS, Android, or both?

### Optional Information

6. **Design inspiration**: Ask if the user has design inspiration — screenshots, mockups, color preferences, apps they admire. If yes, ask them to specify a folder path where they've placed inspiration images, OR describe the aesthetic they want (e.g., "minimal and dark", "playful with bright colors", "professional and clean").
7. **Optional modules** — present these as choices and briefly explain each:
   - **Onboarding flow**: Guided first-run experience (tailored to their app)
   - **Gamification**: Points, achievements, streaks to drive engagement
   - **Push notifications**: FCM-based notifications (requires Firebase setup)
   - **Home screen widgets**: iOS WidgetKit + Android AppWidget
   - **Subscriptions/monetization**: RevenueCat-based subscription tiers

8. **Supabase MCP server** (if database selected): Ask if the user wants to install the Supabase MCP server. This gives Claude direct access to their Supabase project during development — it can run SQL queries, apply migrations, manage Edge Functions, and inspect tables without the user needing to copy-paste between the Supabase dashboard and the terminal. It's optional but significantly speeds up database-related phases. If they say yes, follow the setup steps in `references/supabase-setup.md` § Supabase MCP Server Setup.

### What You're Building Toward

By the end of this interview you should know:
- The app's name (or working name)
- What every screen does
- What data needs to be stored and how it relates
- Which optional modules to include
- Where design inspiration lives (folder path or description)
- Whether to set up the Supabase MCP server (if database selected)

If the user is vague, help them think through it. Ask "what happens when a user opens the app for the first time?" or "what does the main screen show?" to draw out specifics.

---

## Phase 1: Generate the Implementation Plan

Once discovery is complete, generate `docs/implementation-plan.md` in the new project directory. This is the single source of truth for the entire build.

### Plan Structure

```markdown
# [App Name] Implementation Plan

## Current State Summary

**Completed:**
- (none yet)

**In Progress:**
- Phase 1: Project Setup & Core Infrastructure

**Not Started:**
- (list remaining phases)

---

## Implementation Phases

### Phase 1: Project Setup & Core Infrastructure
(details)

### Phase 2: Database Schema & Supabase Setup
(if database selected — otherwise skip)

### Phase 3: Authentication System
(if database selected)

### Phase 4: [First Core Feature]
...

### Phase N: [Optional Modules]
...
```

### How to Prioritize Phases

Always follow this order:
1. **Project setup & core infrastructure** (always first)
2. **Database schema & Supabase setup** (if needed — everything else depends on this)
3. **Authentication** (if database — most features depend on knowing who the user is)
4. **Onboarding flow** (if selected — runs once on first launch, good to build early)
5. **Core features** (the 3-5 main features, ordered by dependency — build the ones others depend on first)
6. **Optional modules** in this order: notifications → gamification → home widgets → subscriptions (subscriptions last because it gates other features)

### Phase Detail Format

Each phase should specify:
- What's being built and why
- Key files to create (using the architecture pattern from `references/architecture.md`)
- Database tables/migrations (if applicable)
- Manual steps required (clearly marked with ⏸️)
- How to verify it works

Present the complete plan to the user. Wait for their approval or modifications before proceeding.

---

## Phase 2: Project Creation & Core Infrastructure

This is always the first phase you execute. Read `references/architecture.md` for the full project structure and patterns.

### Steps

1. **Create the Flutter project**
   ```bash
   flutter create --org com.[user_org] [app_name]
   cd [app_name]
   ```

2. **Set up directory structure** — create the full `lib/` tree:
   ```
   lib/
   ├── main.dart
   ├── core/
   │   ├── config/
   │   │   ├── env.dart
   │   │   └── supabase_config.dart  (if database)
   │   ├── router/
   │   │   └── app_router.dart
   │   ├── theme/
   │   │   ├── app_colors.dart
   │   │   ├── app_typography.dart
   │   │   └── app_theme.dart
   │   ├── animations/
   │   │   └── app_animations.dart
   │   ├── services/
   │   └── utils/
   │       └── logger.dart
   ├── shared/
   │   ├── widgets/
   │   ├── models/
   │   ├── data/
   │   └── providers/
   └── features/
       └── (created per-feature in later phases)
   ```

3. **Install dependencies** — update `pubspec.yaml` based on selected features. Read `references/architecture.md` § Dependencies for the full list organized by category.

4. **Generate the design system** — this is where design inspiration matters:
   - If the user provided an **inspiration folder**: read the images, extract a cohesive color palette, typography direction, and component style. Generate `app_colors.dart`, `app_typography.dart`, and `app_theme.dart` that reflect the inspiration.
   - If the user provided a **description**: translate it into concrete design tokens. "Minimal and dark" → dark surfaces, high contrast, tight spacing. "Playful" → rounded corners, saturated accent colors, bouncy animations.
   - If **no inspiration**: ask the user for at least a primary color preference and overall mood (modern, playful, professional, minimal) before generating the theme.
   - Always use **Google Fonts** — pick a font that matches the aesthetic.
   - Generate `AppColors` with: primary, secondary, accent, background, surface, text, error, and semantic colors (success, warning, info).
   - Generate `AppTypography` with Material 3 text theme using the chosen font.
   - Generate `AppTheme` combining colors + typography into a full `ThemeData`.

5. **Set up build flavors** — create `.env.dev` and `.env.prod` files (with placeholder values) and the `Env` class that loads them based on `--dart-define=FLAVOR`.

6. **Set up GoRouter** — create `app_router.dart` with:
   - Auth guard redirects (if auth selected)
   - Bottom navigation shell (if app has tab-based nav)
   - Initial routes for the main screens identified in discovery
   - Use `context.go()` for tab switches, `context.push()` for stack navigation

7. **Create shared widgets** — scaffold the reusable widget library using the `GW` prefix convention:
   - `GWButton` (primary, secondary, accent, text, icon variants)
   - `GWCard` (standard, hero, listItem variants)
   - `GWTextField` (with label, hint, error, prefix/suffix support)
   - `GWAvatar` (with image URL fallback to initials)
   - Style all widgets using the generated theme — no hardcoded colors.

8. **Set up animations** — create `AppAnimations` with standard durations and curves.

9. **Set up logging** — create `AppLogger` with debug/info/warning/error levels and console output.

10. **Generate CLAUDE.md** — create a project-specific CLAUDE.md following the same structure as a working Flutter project guide: overview, commands, architecture explanation, key patterns, environment setup.

11. **Run the app** — verify it builds and shows the home screen with the generated theme.

After this phase, update `docs/implementation-plan.md`: mark Phase 1 as completed, move the next phase to "In Progress."

---

## Executing Subsequent Phases

For each remaining phase in the implementation plan:

### Before Starting a Phase

1. Read the phase details from `docs/implementation-plan.md`
2. Tell the user what you're about to build and what they can expect
3. If the phase has manual steps (marked with ⏸️), warn them upfront

### During a Phase

- Follow the feature-based clean architecture pattern (see `references/architecture.md`)
- For database features, read `references/supabase-setup.md` for migration patterns
- For optional modules, read the relevant file in `references/modules/`
- After creating Freezed models, run `dart run build_runner build --delete-conflicting-outputs`

### Manual Step Protocol

When you encounter a step that requires the user to do something outside the codebase (console configurations, app store setup, etc.), follow this exact protocol:

1. **Announce** the manual step clearly:
   ```
   ⏸️ MANUAL STEP REQUIRED: [Title]
   ```
2. **Explain** what needs to be done with specific, numbered steps. Read `references/manual-configs.md` for the detailed instructions for each manual configuration.
3. **State what you need back** from them (API key, bundle ID, config file, etc.)
4. **Wait** — do not proceed until the user confirms completion
5. **Verify** if possible (e.g., test an API key, check a config file exists)

Common manual steps and when they occur:
- **Supabase project creation** → Phase 2 (database setup)
- **Apple Developer account / App ID** → Phase 3 (auth with Apple Sign-In) or build phase
- **Google Cloud Console / OAuth credentials** → Phase 3 (auth with Google Sign-In)
- **Firebase project setup** → Notifications phase
- **RevenueCat account setup** → Subscriptions phase
- **App Store Connect / Google Play Console** → Final deployment phase

### After Completing a Phase

1. Verify the feature works (run the app, test the flow)
2. Update `docs/implementation-plan.md`:
   - Check off the completed phase: `- [x] Phase N: ...`
   - Move next phase to "In Progress"
3. Tell the user what was built and what's next
4. Ask: "Ready to move to the next phase, or do you want to adjust anything?"

---

## Module Reference Guide

When implementing optional modules, read the corresponding reference file for detailed patterns:

| Module | Reference File | Key Manual Steps |
|--------|---------------|-----------------|
| Supabase database | `references/supabase-setup.md` | Supabase project creation, Docker for local dev, optional MCP server |
| Authentication | `references/modules/auth.md` | Apple Developer, Google Cloud Console |
| Onboarding | `references/modules/onboarding.md` | None |
| Gamification | `references/modules/gamification.md` | None |
| Notifications | `references/modules/notifications.md` | Firebase project, APNs key, FCM setup |
| Home widgets | `references/modules/home-widgets.md` | Xcode widget extension, App Groups |
| Subscriptions | `references/modules/subscriptions.md` | RevenueCat account, App Store Connect products |

---

## Key Conventions

These conventions apply to every file you generate:

### Naming
- **Shared widgets**: `GW` prefix (e.g., `GWButton`, `GWCard`)
- **Feature widgets**: No prefix
- **Providers**: Suffix with `Provider` (e.g., `peopleProvider`)
- **Controllers**: Suffix with `ControllerProvider` (e.g., `peopleControllerProvider`)
- **Repositories**: Suffix with `Repository` (e.g., `PeopleRepository`)
- **Dart files**: snake_case (e.g., `people_repository.dart`)
- **Supabase tables/functions**: snake_case (e.g., `get_people_for_user`)

### State Management (Riverpod)
- `FutureProvider` for one-time async fetches
- `StreamProvider` for real-time data streams
- `StateNotifierProvider` for mutable state with mutation methods
- `Provider` for computed values and dependency injection
- Always use `ref.watch()` in build methods, `ref.read()` for one-off actions

### Database (Supabase)
- All client-side data access goes through RPC functions — never direct table queries
- RLS on every table — users only see their own data
- `SECURITY DEFINER` on RPC functions that need cross-user access (e.g., shared circles)
- Migration files named: `YYYYMMDD000XXX_description.sql`
- Edge Functions handle their own JWT verification (deploy with `verify_jwt: false`)

### Navigation (GoRouter)
- `context.go('/path')` for full navigation (replaces stack)
- `context.push('/path')` for stack push (back button returns)
- Auth routes outside the shell, main app routes inside a `ShellRoute` with bottom nav

---

## Design Inspiration Handling

When the user provides a folder of inspiration images:

1. Read each image in the folder
2. Analyze across all images for:
   - **Color palette**: Dominant colors, accent colors, background tones
   - **Typography feel**: Rounded vs geometric, weight preferences, serif vs sans-serif
   - **Component style**: Card shapes (rounded? shadowed?), button styles, spacing density
   - **Overall mood**: Minimal, playful, professional, bold, soft
3. Synthesize into a cohesive design system that captures the *spirit* of the inspiration without copying any specific design
4. Generate the theme files and show the user a summary of your design decisions
5. Ask for feedback before proceeding — the theme touches every screen

If the user provides new inspiration images later, regenerate the theme and update all screens accordingly.
