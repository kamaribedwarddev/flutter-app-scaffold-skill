# Architecture Reference

This document defines the project structure, patterns, and conventions for every Flutter app scaffolded by this skill.

## Table of Contents
1. [Directory Structure](#directory-structure)
2. [Dependencies](#dependencies)
3. [Feature Architecture Pattern](#feature-architecture-pattern)
4. [State Management](#state-management)
5. [Navigation](#navigation)
6. [Design System Structure](#design-system-structure)
7. [Environment Configuration](#environment-configuration)
8. [Shared Widget Library](#shared-widget-library)
9. [Logging](#logging)
10. [Code Generation](#code-generation)

---

## Directory Structure

Every project follows this exact structure. Create all directories upfront during Phase 1.

```
lib/
├── main.dart
├── core/
│   ├── config/
│   │   ├── env.dart                    # Environment variable loading
│   │   └── supabase_config.dart        # Supabase client init (if database)
│   ├── router/
│   │   └── app_router.dart             # GoRouter config with guards
│   ├── theme/
│   │   ├── app_colors.dart             # Color palette + gradients
│   │   ├── app_typography.dart         # Text styles + font config
│   │   └── app_theme.dart              # ThemeData combining colors + type
│   ├── animations/
│   │   └── app_animations.dart         # Shared durations, curves, widgets
│   ├── services/                       # App-wide services (timezone, cache, etc.)
│   └── utils/
│       └── logger.dart                 # Centralized logging
├── shared/
│   ├── widgets/                        # GW-prefixed reusable components
│   ├── models/                         # Shared domain models
│   ├── data/                           # Shared repositories
│   └── providers/                      # Global providers
└── features/
    └── {feature_name}/
        ├── data/
        │   ├── repositories/           # Data access (Supabase RPC calls)
        │   └── services/               # Realtime streams, specialized logic
        ├── domain/
        │   └── models/                 # Freezed immutable models
        └── presentation/
            ├── providers/              # Riverpod providers + controllers
            ├── screens/                # Full-page widgets
            └── widgets/                # Feature-specific UI components
```

Additional project-level files:
```
project_root/
├── docs/
│   └── implementation-plan.md          # Living roadmap (always create)
├── supabase/                           # (if database)
│   ├── migrations/                     # Numbered SQL migrations
│   ├── functions/                      # Edge Functions (TypeScript/Deno)
│   ├── config.toml                     # Local Supabase config
│   └── seed.sql                        # Development seed data
├── .env.dev                            # Local development environment
├── .env.prod                           # Production environment
├── CLAUDE.md                           # Developer guide for Claude Code
└── pubspec.yaml
```

---

## Dependencies

Install these based on which features are selected. Always include the "Core" group.

### Core (always)
```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.6.1
  riverpod_annotation: ^2.6.1
  go_router: ^14.6.2
  freezed_annotation: ^2.4.4
  json_annotation: ^4.9.0
  google_fonts: ^6.2.1
  flutter_dotenv: ^5.2.1
  intl: ^0.19.0
  uuid: ^4.5.1
  cached_network_image: ^3.4.1
  flutter_svg: ^2.0.16
  share_plus: ^10.1.4
  url_launcher: ^6.3.1

dev_dependencies:
  flutter_lints: ^5.0.0
  build_runner: ^2.4.13
  freezed: ^2.5.7
  json_serializable: ^6.8.0
  riverpod_generator: ^2.6.3
```

### Database (Supabase)
```yaml
dependencies:
  supabase_flutter: ^2.8.3
  flutter_secure_storage: ^9.2.3
```

### Authentication
```yaml
dependencies:
  sign_in_with_apple: ^6.1.3
  google_sign_in: ^6.2.2
  crypto: ^3.0.6
```

### Notifications (FCM)
```yaml
dependencies:
  firebase_core: ^3.8.1
  firebase_messaging: ^15.1.6
  flutter_local_notifications: ^18.0.1
  timezone: ^0.10.0
```

### Home Widgets
```yaml
dependencies:
  home_widget: ^0.7.0
```

### Subscriptions (RevenueCat)
```yaml
dependencies:
  purchases_flutter: ^8.1.0
  purchases_ui_flutter: ^8.1.0
```

### Monitoring (recommended for prod)
```yaml
dependencies:
  sentry_flutter: ^8.12.0
  firebase_crashlytics: ^4.1.6
  firebase_analytics: ^11.3.6
```

### Media & Files
```yaml
dependencies:
  image_picker: ^1.1.2
  file_picker: ^8.1.6
  permission_handler: ^11.3.1
```

---

## Feature Architecture Pattern

Every feature follows three layers. This is non-negotiable — it keeps the codebase navigable as it grows.

### Data Layer (`data/`)
Handles communication with external data sources (Supabase, APIs, local storage).

```dart
// data/repositories/people_repository.dart
class PeopleRepository {
  final SupabaseClient _supabase;

  PeopleRepository(this._supabase);

  Future<List<Person>> getPeople() async {
    final response = await _supabase.rpc('get_people_for_user');
    return (response as List).map((json) => Person.fromJson(json)).toList();
  }

  Stream<List<Person>> getPeopleStream() {
    // Use Supabase Realtime for live updates
    return _supabase
        .from('people')
        .stream(primaryKey: ['id'])
        .map((data) => data.map((json) => Person.fromJson(json)).toList());
  }

  Future<void> createPerson(Person person) async {
    await _supabase.rpc('create_person', params: person.toJson());
  }
}
```

### Domain Layer (`domain/`)
Pure Dart models with no framework dependencies. Always use Freezed.

```dart
// domain/models/person.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'person.freezed.dart';
part 'person.g.dart';

@freezed
class Person with _$Person {
  const factory Person({
    required String id,
    required String name,
    String? relationship,
    String? notes,
    String? avatarUrl,
    required DateTime createdAt,
  }) = _Person;

  factory Person.fromJson(Map<String, dynamic> json) => _$PersonFromJson(json);
}
```

### Presentation Layer (`presentation/`)
Providers expose data to widgets. Controllers handle mutations.

```dart
// presentation/providers/people_providers.dart

// Repository provider (dependency injection)
final peopleRepositoryProvider = Provider<PeopleRepository>((ref) {
  final supabase = Supabase.instance.client;
  return PeopleRepository(supabase);
});

// Data provider (read-only stream)
final peopleProvider = StreamProvider<List<Person>>((ref) {
  return ref.watch(peopleRepositoryProvider).getPeopleStream();
});

// Controller provider (mutations)
final peopleControllerProvider =
    StateNotifierProvider<PeopleController, AsyncValue<void>>((ref) {
  return PeopleController(ref.watch(peopleRepositoryProvider));
});

class PeopleController extends StateNotifier<AsyncValue<void>> {
  final PeopleRepository _repository;

  PeopleController(this._repository) : super(const AsyncData(null));

  Future<void> createPerson(Person person) async {
    state = const AsyncLoading();
    try {
      await _repository.createPerson(person);
      state = const AsyncData(null);
    } catch (e, st) {
      state = AsyncError(e, st);
    }
  }
}
```

---

## State Management

### Provider Selection Guide

| Use Case | Provider Type | Example |
|-----------|--------------|---------|
| One-time async fetch | `FutureProvider` | Loading user profile |
| Real-time data stream | `StreamProvider` | Live chat messages |
| Mutable state + mutations | `StateNotifierProvider` | Form state, CRUD operations |
| Computed/derived values | `Provider` | Repository instances, filtered lists |
| Simple reactive value | `StateProvider` | Toggle, selected tab index |

### Key Patterns

**Avoid token-refresh cascades** — when you need the current user ID in multiple providers, select just the ID rather than watching the full auth state:

```dart
final currentUserIdProvider = Provider<String?>((ref) {
  return ref.watch(authStateProvider).valueOrNull?.id;
});
```

**AsyncValue in the UI** — always handle all three states:

```dart
ref.watch(peopleProvider).when(
  data: (people) => PeopleList(people: people),
  loading: () => const Center(child: CircularProgressIndicator()),
  error: (e, st) => ErrorWidget(message: e.toString()),
);
```

---

## Navigation

### GoRouter Setup

```dart
final routerProvider = Provider<GoRouter>((ref) {
  final authState = ref.watch(authStateProvider);

  return GoRouter(
    initialLocation: '/home',
    redirect: (context, state) {
      final isAuthenticated = authState.valueOrNull != null;
      final isAuthRoute = state.matchedLocation.startsWith('/login') ||
          state.matchedLocation.startsWith('/signup');

      if (!isAuthenticated && !isAuthRoute) return '/login';
      if (isAuthenticated && isAuthRoute) return '/home';
      return null;
    },
    routes: [
      // Auth routes (outside shell)
      GoRoute(path: '/login', builder: (_, __) => const LoginScreen()),
      GoRoute(path: '/signup', builder: (_, __) => const SignupScreen()),

      // Main app (inside shell with bottom nav)
      ShellRoute(
        builder: (_, __, child) => MainShell(child: child),
        routes: [
          GoRoute(path: '/home', builder: (_, __) => const HomeScreen()),
          // ... other tab routes
        ],
      ),

      // Full-screen routes (outside shell, with back button)
      GoRoute(path: '/items/add', builder: (_, __) => const AddItemScreen()),
    ],
  );
});
```

### Navigation Rules
- **Tab switches**: `context.go('/tab-path')` — replaces the route
- **Push onto stack**: `context.push('/detail-path')` — adds to navigation stack
- **Go back**: `context.pop()` — pops the stack
- Auth routes and onboarding live outside the `ShellRoute`
- Feature detail screens can be inside or outside the shell depending on whether the bottom nav should remain visible

---

## Design System Structure

### AppColors

```dart
class AppColors {
  // Primary palette (generated from inspiration)
  static const Color primary = Color(0xFF...);
  static const Color primaryLight = Color(0xFF...);
  static const Color primaryDark = Color(0xFF...);

  // Secondary palette
  static const Color secondary = Color(0xFF...);
  static const Color accent = Color(0xFF...);

  // Backgrounds
  static const Color background = Color(0xFF...);
  static const Color surface = Color(0xFF...);
  static const Color surfaceVariant = Color(0xFF...);

  // Text
  static const Color textPrimary = Color(0xFF...);
  static const Color textSecondary = Color(0xFF...);
  static const Color textTertiary = Color(0xFF...);

  // Semantic
  static const Color success = Color(0xFF4CAF50);
  static const Color warning = Color(0xFFFF9800);
  static const Color error = Color(0xFFF44336);
  static const Color info = Color(0xFF2196F3);

  // Gradients (optional)
  static const LinearGradient primaryGradient = LinearGradient(...);
}
```

### AppTypography

Use Google Fonts. Pick a font that matches the aesthetic.

```dart
class AppTypography {
  static String get _fontFamily => GoogleFonts.nunito().fontFamily!;

  static TextTheme get textTheme => TextTheme(
    displayLarge: TextStyle(fontFamily: _fontFamily, fontSize: 48, fontWeight: FontWeight.bold),
    headlineLarge: TextStyle(fontFamily: _fontFamily, fontSize: 32, fontWeight: FontWeight.bold),
    titleLarge: TextStyle(fontFamily: _fontFamily, fontSize: 20, fontWeight: FontWeight.w600),
    bodyLarge: TextStyle(fontFamily: _fontFamily, fontSize: 16),
    bodyMedium: TextStyle(fontFamily: _fontFamily, fontSize: 14),
    labelLarge: TextStyle(fontFamily: _fontFamily, fontSize: 14, fontWeight: FontWeight.w600),
  );
}
```

### AppTheme

Combines everything into a `ThemeData`:

```dart
class AppTheme {
  static ThemeData get lightTheme => ThemeData(
    useMaterial3: true,
    colorScheme: ColorScheme.light(
      primary: AppColors.primary,
      secondary: AppColors.secondary,
      surface: AppColors.surface,
      error: AppColors.error,
    ),
    textTheme: AppTypography.textTheme,
    // Component themes...
  );
}
```

---

## Environment Configuration

### Env Class

```dart
class Env {
  static late String supabaseUrl;
  static late String supabaseAnonKey;
  // ... other keys based on selected features

  static bool get isBetaMode =>
      const String.fromEnvironment('BETA_MODE', defaultValue: 'false') == 'true';

  static Future<void> load() async {
    const flavor = String.fromEnvironment('FLAVOR', defaultValue: 'dev');
    await dotenv.load(fileName: '.env.$flavor');

    supabaseUrl = dotenv.env['SUPABASE_URL']!;
    supabaseAnonKey = dotenv.env['SUPABASE_ANON_KEY']!;
  }
}
```

### .env Files

```bash
# .env.dev (local development)
SUPABASE_URL=http://127.0.0.1:54331
SUPABASE_ANON_KEY=<local-anon-key>

# .env.prod (production)
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=<prod-anon-key>
```

### Build Flavors

```bash
# Development (default)
flutter run

# Production
flutter run --dart-define=FLAVOR=prod

# Beta (production backend, beta UI)
flutter run --dart-define=FLAVOR=prod --dart-define=BETA_MODE=true
```

---

## Shared Widget Library

All shared widgets use the `GW` prefix (standing for the app's initials — adjust per project). They must:
- Accept all styling via the theme — no hardcoded colors
- Support multiple variants via an enum parameter
- Include loading/disabled states where appropriate

### GWButton

```dart
enum GWButtonVariant { primary, secondary, accent, text, icon }

class GWButton extends StatelessWidget {
  final String? label;
  final IconData? icon;
  final VoidCallback? onPressed;
  final GWButtonVariant variant;
  final bool isLoading;
  // ...
}
```

### GWCard

```dart
enum GWCardVariant { standard, hero, listItem, accent }

class GWCard extends StatelessWidget {
  final Widget child;
  final GWCardVariant variant;
  final VoidCallback? onTap;
  // ...
}
```

### GWTextField

```dart
class GWTextField extends StatelessWidget {
  final String? label;
  final String? hint;
  final String? errorText;
  final TextEditingController? controller;
  final Widget? prefix;
  final Widget? suffix;
  final bool obscureText;
  // ...
}
```

### GWAvatar

```dart
class GWAvatar extends StatelessWidget {
  final String? imageUrl;
  final String? initials;
  final double size;
  // Falls back to initials circle if no image
}
```

---

## Logging

```dart
class AppLogger {
  static void debug(String message) => _log('🔍', message);
  static void info(String message) => _log('ℹ️', message);
  static void warning(String message) => _log('⚠️', message);
  static void error(String message, [Object? error, StackTrace? stackTrace]) {
    _log('❌', message);
    if (error != null) debugPrint('  Error: $error');
    if (stackTrace != null) debugPrint('  Stack: $stackTrace');
  }

  static void _log(String emoji, String message) {
    final timestamp = DateTime.now().toIso8601String().substring(11, 23);
    debugPrint('$emoji [$timestamp] $message');
  }
}
```

---

## Code Generation

After creating or modifying any Freezed model, always run:

```bash
dart run build_runner build --delete-conflicting-outputs
```

This generates:
- `.freezed.dart` — immutability, copyWith, equality
- `.g.dart` — JSON serialization (fromJson / toJson)

For active development with frequent model changes:

```bash
dart run build_runner watch --delete-conflicting-outputs
```
