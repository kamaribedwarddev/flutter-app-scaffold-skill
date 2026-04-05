# Subscriptions & Monetization Module

RevenueCat-powered subscription management with free/premium tiers and upgrade prompts.

## Files to Create

```
lib/features/subscriptions/
├── data/
│   └── repositories/
│       └── subscription_repository.dart
├── domain/
│   └── models/
│       ├── subscription_tier.dart
│       └── user_subscription.dart     # + generated files
└── presentation/
    ├── providers/
    │   └── subscription_providers.dart
    ├── screens/
    │   ├── paywall_screen.dart
    │   └── subscription_management_screen.dart
    └── widgets/
        ├── upgrade_prompt.dart
        └── tier_badge.dart
```

## Database Tables

```sql
CREATE TABLE public.user_subscriptions (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE UNIQUE NOT NULL,
    tier TEXT DEFAULT 'free' NOT NULL,          -- 'free', 'premium', 'pro', etc.
    revenuecat_customer_id TEXT,
    product_id TEXT,                            -- RevenueCat product identifier
    expires_at TIMESTAMPTZ,
    is_active BOOLEAN DEFAULT false NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);
```

## Tier Design

During discovery, define what's free vs. premium. Common patterns:

| Pattern | Free Tier | Premium Tier |
|---------|-----------|-------------|
| **Usage limits** | 5 items, 3 AI calls/month | Unlimited |
| **Feature gating** | Core features only | Advanced features unlocked |
| **Ads** | Shows ads | Ad-free |
| **Storage** | 100MB | 10GB |
| **Export** | No export | CSV/PDF export |

### Tier Enum

```dart
enum SubscriptionTier {
  free,
  premium;

  bool get isPremium => this == SubscriptionTier.premium;

  // Define limits per tier
  int get maxItems => switch (this) {
    free => 10,
    premium => -1, // unlimited
  };

  int get monthlyAiCalls => switch (this) {
    free => 3,
    premium => -1,
  };
}
```

## Manual Steps Required

See `manual-configs.md` § RevenueCat Setup for the full setup process.

## RevenueCat Integration

### Initialization

```dart
class SubscriptionRepository {
  Future<void> initialize() async {
    await Purchases.setLogLevel(LogLevel.debug); // Remove in production
    
    PurchasesConfiguration configuration;
    if (Platform.isIOS) {
      configuration = PurchasesConfiguration(Env.revenueCatIosApiKey);
    } else {
      configuration = PurchasesConfiguration(Env.revenueCatAndroidApiKey);
    }

    await Purchases.configure(configuration);
  }

  /// Set the user ID after authentication
  Future<void> login(String userId) async {
    await Purchases.logIn(userId);
  }

  Future<void> logout() async {
    await Purchases.logOut();
  }

  /// Get available packages to show on paywall
  Future<List<Package>> getOfferings() async {
    final offerings = await Purchases.getOfferings();
    return offerings.current?.availablePackages ?? [];
  }

  /// Purchase a package
  Future<CustomerInfo> purchase(Package package) async {
    final result = await Purchases.purchasePackage(package);
    return result.customerInfo;
  }

  /// Restore purchases (required by App Store)
  Future<CustomerInfo> restorePurchases() async {
    return await Purchases.restorePurchases();
  }

  /// Check current entitlements
  Future<SubscriptionTier> getCurrentTier() async {
    final customerInfo = await Purchases.getCustomerInfo();
    if (customerInfo.entitlements.active.containsKey('premium')) {
      return SubscriptionTier.premium;
    }
    return SubscriptionTier.free;
  }

  /// Listen for subscription changes
  Stream<CustomerInfo> get customerInfoStream {
    return Purchases.customerInfoStream;
  }
}
```

### Providers

```dart
final subscriptionRepositoryProvider = Provider<SubscriptionRepository>((ref) {
  return SubscriptionRepository();
});

final currentTierProvider = FutureProvider<SubscriptionTier>((ref) async {
  return ref.watch(subscriptionRepositoryProvider).getCurrentTier();
});

final offeringsProvider = FutureProvider<List<Package>>((ref) async {
  return ref.watch(subscriptionRepositoryProvider).getOfferings();
});
```

### Tier Enforcement

Create a utility to check access:

```dart
extension TierCheck on WidgetRef {
  bool canAccess(SubscriptionTier requiredTier) {
    final currentTier = watch(currentTierProvider).valueOrNull ?? SubscriptionTier.free;
    return currentTier.index >= requiredTier.index;
  }
}
```

Use in features:

```dart
// In a screen or widget
if (!ref.canAccess(SubscriptionTier.premium)) {
  showUpgradePrompt(context);
  return;
}
// ... premium feature code
```

### Upgrade Prompt Widget

```dart
class UpgradePrompt extends StatelessWidget {
  final String feature;
  final String description;

  @override
  Widget build(BuildContext context) {
    return GWCard(
      variant: GWCardVariant.accent,
      child: Column(
        children: [
          Icon(Icons.lock_outline, size: 48, color: AppColors.accent),
          const SizedBox(height: 12),
          Text('Unlock $feature', style: AppTypography.titleMedium),
          const SizedBox(height: 8),
          Text(description, textAlign: TextAlign.center),
          const SizedBox(height: 16),
          GWButton(
            label: 'Upgrade to Premium',
            onPressed: () => context.push('/paywall'),
          ),
        ],
      ),
    );
  }
}
```

## Paywall Screen

RevenueCat provides a pre-built paywall UI, or you can build custom:

### Using RevenueCat's Paywall UI (simpler)

```dart
class PaywallScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return PaywallView(
      offering: ref.watch(offeringsProvider).valueOrNull,
      onPurchaseCompleted: (customerInfo) {
        ref.invalidate(currentTierProvider);
        context.pop();
      },
      onRestoreCompleted: (customerInfo) {
        ref.invalidate(currentTierProvider);
        context.pop();
      },
    );
  }
}
```

### Custom Paywall (more control)

Build a custom screen showing packages with pricing, features comparison, and purchase buttons. Include a "Restore Purchases" button (required for App Store approval).

## Webhook Integration

Create an Edge Function to handle RevenueCat webhooks — this keeps your database in sync with subscription state:

```typescript
// supabase/functions/revenuecat-webhook/index.ts
serve(async (req) => {
  const body = await req.json()
  const event = body.event

  // Verify webhook (check authorization header matches your webhook secret)
  const authHeader = req.headers.get('Authorization')
  if (authHeader !== `Bearer ${Deno.env.get('REVENUECAT_WEBHOOK_SECRET')}`) {
    return new Response('Unauthorized', { status: 401 })
  }

  const userId = event.app_user_id
  const productId = event.product_id

  switch (event.type) {
    case 'INITIAL_PURCHASE':
    case 'RENEWAL':
      await supabase.from('user_subscriptions').upsert({
        user_id: userId,
        tier: 'premium',
        product_id: productId,
        is_active: true,
        expires_at: event.expiration_at_ms
          ? new Date(event.expiration_at_ms).toISOString()
          : null,
      })
      break

    case 'CANCELLATION':
    case 'EXPIRATION':
      await supabase.from('user_subscriptions').update({
        is_active: false,
        tier: 'free',
      }).eq('user_id', userId)
      break
  }

  return new Response(JSON.stringify({ received: true }))
})
```

Configure the webhook URL in RevenueCat: `https://[project-ref].supabase.co/functions/v1/revenuecat-webhook`

## Beta Mode Support

For TestFlight/internal testing, support a beta mode that hides subscription UI:

```dart
// In Env class
static bool get isBetaMode =>
    const String.fromEnvironment('BETA_MODE', defaultValue: 'false') == 'true';

// In subscription UI
if (Env.isBetaMode) {
  // Show "Beta Limit Reached" instead of upgrade prompt
  // Hide subscription menu item in profile
  // Show "BETA" badge
}
```

Build with: `flutter build ios --dart-define=FLAVOR=prod --dart-define=BETA_MODE=true`
