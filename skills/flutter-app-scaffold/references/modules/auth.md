# Authentication Module

Implements email/password auth with optional Apple and Google social sign-in, powered by Supabase Auth.

## Files to Create

```
lib/features/auth/
├── data/
│   └── repositories/
│       └── auth_repository.dart
├── domain/
│   └── models/
│       └── app_user.dart            # + .freezed.dart, .g.dart (generated)
└── presentation/
    ├── providers/
    │   └── auth_providers.dart
    ├── screens/
    │   ├── login_screen.dart
    │   ├── signup_screen.dart
    │   └── forgot_password_screen.dart
    └── widgets/
        └── social_auth_buttons.dart
```

## Database Prerequisites

The `profiles` table must exist (created in the database schema phase) with an `on_auth_user_created` trigger that auto-creates a profile row when a user signs up.

## Auth Repository

The repository wraps Supabase Auth methods:

```dart
class AuthRepository {
  final SupabaseClient _supabase;
  AuthRepository(this._supabase);

  // Email auth
  Future<AuthResponse> signUpWithEmail(String email, String password, String name) async {
    return await _supabase.auth.signUp(
      email: email,
      password: password,
      data: {'name': name},
    );
  }

  Future<AuthResponse> signInWithEmail(String email, String password) async {
    return await _supabase.auth.signInWithPassword(email: email, password: password);
  }

  Future<void> signOut() async => await _supabase.auth.signOut();

  Future<void> resetPassword(String email) async {
    await _supabase.auth.resetPasswordForEmail(email);
  }

  // Auth state
  Stream<AuthState> onAuthStateChange() => _supabase.auth.onAuthStateChange;
  User? get currentUser => _supabase.auth.currentUser;
}
```

### Apple Sign-In Addition

```dart
Future<AuthResponse> signInWithApple() async {
  final rawNonce = _supabase.auth.generateRawNonce();
  final hashedNonce = sha256.convert(utf8.encode(rawNonce)).toString();

  final credential = await SignInWithApple.getAppleIDCredential(
    scopes: [AppleIDAuthorizationScopes.email, AppleIDAuthorizationScopes.fullName],
    nonce: hashedNonce,
  );

  return await _supabase.auth.signInWithIdToken(
    provider: OAuthProvider.apple,
    idToken: credential.identityToken!,
    nonce: rawNonce,
  );
}
```

### Google Sign-In Addition

```dart
Future<AuthResponse> signInWithGoogle() async {
  final googleSignIn = GoogleSignIn(
    clientId: Platform.isIOS ? Env.googleIosClientId : null,
    serverClientId: Env.googleWebClientId,
  );

  final googleUser = await googleSignIn.signIn();
  if (googleUser == null) throw Exception('Google sign-in cancelled');

  final googleAuth = await googleUser.authentication;

  return await _supabase.auth.signInWithIdToken(
    provider: OAuthProvider.google,
    idToken: googleAuth.idToken!,
    accessToken: googleAuth.accessToken,
  );
}
```

## AppUser Model (Freezed)

```dart
@freezed
class AppUser with _$AppUser {
  const factory AppUser({
    required String id,
    required String email,
    String? name,
    String? avatarUrl,
    DateTime? createdAt,
  }) = _AppUser;

  factory AppUser.fromJson(Map<String, dynamic> json) => _$AppUserFromJson(json);

  factory AppUser.fromSupabaseUser(User user) => AppUser(
    id: user.id,
    email: user.email ?? '',
    name: user.userMetadata?['name'] as String?,
    avatarUrl: user.userMetadata?['avatar_url'] as String?,
    createdAt: DateTime.tryParse(user.createdAt),
  );
}
```

## Providers

```dart
// Auth state stream
final authStateProvider = StreamProvider<AuthState>((ref) {
  return ref.watch(authRepositoryProvider).onAuthStateChange();
});

// Current user ID selector (prevents rebuild cascades)
final currentUserIdProvider = Provider<String?>((ref) {
  final session = ref.watch(authStateProvider).valueOrNull;
  return session?.session?.user.id;
});

// Auth repository
final authRepositoryProvider = Provider<AuthRepository>((ref) {
  return AuthRepository(Supabase.instance.client);
});

// Auth controller for actions
final authControllerProvider =
    StateNotifierProvider<AuthController, AsyncValue<void>>((ref) {
  return AuthController(ref.watch(authRepositoryProvider));
});
```

## Screen Design Guidelines

**Login Screen:**
- App logo or illustrated header at the top
- Email and password fields using GWTextField
- "Forgot password?" text link
- Primary "Log In" button using GWButton
- Social auth divider ("or continue with")
- Social auth buttons (Apple, Google) if enabled
- "Don't have an account? Sign Up" link at bottom

**Signup Screen:**
- Minimal fields: name, email, password
- Password strength indicator (optional)
- Primary "Create Account" button
- Social auth options
- "Already have an account? Log In" link

**Forgot Password Screen:**
- Simple email input
- "Send Reset Link" button
- Back to login link
- Success state: show confirmation message

## Router Integration

Update `app_router.dart` to add auth guards:

```dart
redirect: (context, state) {
  final isAuthenticated = ref.read(currentUserIdProvider) != null;
  final isAuthRoute = ['/login', '/signup', '/forgot-password']
      .contains(state.matchedLocation);

  if (!isAuthenticated && !isAuthRoute) return '/login';
  if (isAuthenticated && isAuthRoute) return '/home';
  return null;
},
```

## Manual Steps Required

- **Apple Sign-In**: See `manual-configs.md` § Apple Sign-In Configuration
- **Google Sign-In**: See `manual-configs.md` § Google Cloud Console & OAuth
