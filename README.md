# Flutter App Scaffold

A Claude Code skill that scaffolds complete Flutter mobile apps from an idea.

## Features

- **Feature-based clean architecture** (data / domain / presentation layers)
- **Riverpod** for state management
- **GoRouter** for navigation with auth guards
- **Freezed** for immutable models with JSON serialization
- **Optional Supabase** backend (Postgres + RLS + Edge Functions)
- **Build flavors** (dev / prod) with environment-specific config
- **Modular features**: Auth, Gamification, Notifications, Home Widgets, Subscriptions

## Installation

```
/plugin install your-username/flutter-app-scaffold-skill
```

## Usage

Once installed, Claude will automatically use this skill when you:
- Want to create a new Flutter app
- Start a mobile project from scratch
- Scaffold an app from an idea
- Set up a Flutter project with clean architecture

Or invoke directly:
```
/flutter-app-scaffold
```

## License

MIT
