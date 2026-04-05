# Onboarding Module

A guided first-run experience tailored to the app's purpose. The onboarding flow collects essential information from the user and introduces them to core features.

## Important: This Is NOT a Template

Every onboarding flow should be unique to the app being built. The structure below is a framework — the actual steps, questions, and illustrations must be designed around what the app needs to know about the user on day one.

## Files to Create

```
lib/features/onboarding/
├── data/
│   └── repositories/
│       └── onboarding_repository.dart
├── domain/
│   └── models/
│       └── onboarding_state.dart     # + .freezed.dart, .g.dart
└── presentation/
    ├── providers/
    │   └── onboarding_providers.dart
    ├── screens/
    │   └── onboarding_screen.dart    # PageView-based flow
    └── widgets/
        ├── onboarding_step_*.dart    # One widget per step
        └── onboarding_progress.dart  # Progress indicator
```

## Design Principles

1. **Conversational, not form-like** — each step should feel like a friendly question, not a form field. Use illustrations or icons to make it visual.
2. **3-6 steps maximum** — any more and users abandon the flow. Prioritize what you absolutely need.
3. **Allow skipping** — always provide a "Skip" option. Users can fill in details later.
4. **Show progress** — a step indicator (dots or progress bar) so users know how far along they are.
5. **Celebrate completion** — the final step should feel rewarding: a welcome message, a summary of what they set up, or a preview of the app.

## How to Design Steps

During discovery, identify what the app needs from the user:

| App Type | Possible Onboarding Steps |
|----------|--------------------------|
| Social/community | Display name → Avatar → Interests → Follow suggestions |
| Fitness/health | Name → Goals (lose weight, build muscle) → Activity level → Schedule |
| Productivity | Name → What do you want to track? → Import existing data? → Notification preferences |
| E-commerce | Name → Interests/categories → Size preferences → Notification preferences |
| Finance | Name → Financial goals → Connect accounts? → Budget preferences |
| Education | Name → What to learn → Skill level → Daily time commitment |

The steps should collect data that will meaningfully personalize the app experience.

## Implementation Pattern

### Onboarding State

```dart
@freezed
class OnboardingState with _$OnboardingState {
  const factory OnboardingState({
    @Default(0) int currentStep,
    @Default(false) bool isComplete,
    String? name,
    // ... app-specific fields collected during onboarding
  }) = _OnboardingState;

  factory OnboardingState.fromJson(Map<String, dynamic> json) =>
      _$OnboardingStateFromJson(json);
}
```

### Screen Structure

```dart
class OnboardingScreen extends ConsumerStatefulWidget {
  @override
  ConsumerState<OnboardingScreen> createState() => _OnboardingScreenState();
}

class _OnboardingScreenState extends ConsumerState<OnboardingScreen> {
  final PageController _pageController = PageController();
  int _currentStep = 0;

  // Define the steps based on the app
  List<Widget> get _steps => [
    OnboardingStepWelcome(onNext: _nextStep),
    OnboardingStepName(onNext: _nextStep, onSkip: _nextStep),
    // ... more steps
    OnboardingStepComplete(onFinish: _completeOnboarding),
  ];

  void _nextStep() {
    if (_currentStep < _steps.length - 1) {
      _pageController.nextPage(
        duration: AppAnimations.normalDuration,
        curve: AppAnimations.defaultCurve,
      );
      setState(() => _currentStep++);
    }
  }

  void _completeOnboarding() {
    ref.read(onboardingControllerProvider.notifier).completeOnboarding();
    context.go('/home');
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: Column(
          children: [
            // Progress indicator
            OnboardingProgress(
              currentStep: _currentStep,
              totalSteps: _steps.length,
            ),
            // Skip button (except on last step)
            if (_currentStep < _steps.length - 1)
              Align(
                alignment: Alignment.topRight,
                child: TextButton(
                  onPressed: _completeOnboarding,
                  child: const Text('Skip'),
                ),
              ),
            // Step content
            Expanded(
              child: PageView(
                controller: _pageController,
                physics: const NeverScrollableScrollPhysics(),
                children: _steps,
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

### Progress Indicator Widget

```dart
class OnboardingProgress extends StatelessWidget {
  final int currentStep;
  final int totalSteps;

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.center,
      children: List.generate(totalSteps, (index) {
        return AnimatedContainer(
          duration: AppAnimations.fastDuration,
          margin: const EdgeInsets.symmetric(horizontal: 4),
          width: index == currentStep ? 24 : 8,
          height: 8,
          decoration: BoxDecoration(
            color: index <= currentStep
                ? AppColors.primary
                : AppColors.primary.withOpacity(0.2),
            borderRadius: BorderRadius.circular(4),
          ),
        );
      }),
    );
  }
}
```

## Database Integration

If the app has a database, save onboarding data to the user's profile:

```dart
class OnboardingRepository {
  final SupabaseClient _supabase;

  Future<void> saveOnboardingData(Map<String, dynamic> data) async {
    await _supabase.rpc('update_onboarding', params: {
      'p_data': data,
    });
  }

  Future<void> markOnboardingComplete() async {
    await _supabase.rpc('complete_onboarding');
  }
}
```

Add an `onboarding_completed` boolean to the profiles table, and check it in the router to redirect new users to onboarding.

## Router Integration

```dart
redirect: (context, state) {
  final user = ref.read(currentUserIdProvider);
  final hasCompletedOnboarding = ref.read(onboardingCompleteProvider).valueOrNull ?? false;

  if (user != null && !hasCompletedOnboarding && state.matchedLocation != '/onboarding') {
    return '/onboarding';
  }
  // ... other redirects
},
```
