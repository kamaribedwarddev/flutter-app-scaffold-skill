# Notifications Module

Firebase Cloud Messaging (FCM) for push notifications + flutter_local_notifications for scheduled local notifications.

## Files to Create

```
lib/features/notifications/
├── data/
│   ├── repositories/
│   │   └── notifications_repository.dart
│   └── services/
│       ├── fcm_service.dart
│       └── local_notification_service.dart
├── domain/
│   └── models/
│       └── notification_settings.dart    # + generated files
└── presentation/
    ├── providers/
    │   └── notification_providers.dart
    ├── screens/
    │   └── notification_settings_screen.dart
    └── widgets/
        └── notification_toggle.dart
```

## Database Tables

```sql
-- Store FCM tokens per device
CREATE TABLE public.fcm_tokens (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    token TEXT NOT NULL,
    platform TEXT NOT NULL,        -- 'ios' or 'android'
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    UNIQUE(user_id, token)
);

-- Notification preferences
CREATE TABLE public.notification_settings (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE UNIQUE NOT NULL,
    push_enabled BOOLEAN DEFAULT true,
    quiet_hours_start TIME,       -- e.g., '22:00'
    quiet_hours_end TIME,         -- e.g., '08:00'
    -- Add app-specific notification type toggles:
    -- reminders_enabled BOOLEAN DEFAULT true,
    -- social_enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);
```

## Manual Steps Required

Before implementing, the user must complete:
1. **Firebase project setup** — see `manual-configs.md` § Firebase Project Setup
2. **APNs key upload** — see `manual-configs.md` § Firebase Cloud Messaging (APNs)

## FCM Service

```dart
class FCMService {
  final FirebaseMessaging _messaging = FirebaseMessaging.instance;

  Future<void> initialize() async {
    // Request permission (iOS)
    final settings = await _messaging.requestPermission(
      alert: true,
      badge: true,
      sound: true,
    );

    if (settings.authorizationStatus == AuthorizationStatus.authorized) {
      // Get and save FCM token
      final token = await _messaging.getToken();
      if (token != null) await _saveToken(token);

      // Listen for token refresh
      _messaging.onTokenRefresh.listen(_saveToken);

      // Handle foreground messages
      FirebaseMessaging.onMessage.listen(_handleForegroundMessage);

      // Handle background/terminated taps
      FirebaseMessaging.onMessageOpenedApp.listen(_handleNotificationTap);

      // Check if app was opened from notification
      final initialMessage = await _messaging.getInitialMessage();
      if (initialMessage != null) _handleNotificationTap(initialMessage);
    }
  }

  Future<void> _saveToken(String token) async {
    final userId = Supabase.instance.client.auth.currentUser?.id;
    if (userId == null) return;

    await Supabase.instance.client.rpc('upsert_fcm_token', params: {
      'p_token': token,
      'p_platform': Platform.isIOS ? 'ios' : 'android',
    });
  }

  void _handleForegroundMessage(RemoteMessage message) {
    // Show local notification when app is in foreground
    LocalNotificationService.showFromRemoteMessage(message);
  }

  void _handleNotificationTap(RemoteMessage message) {
    // Navigate based on notification data payload
    final route = message.data['route'];
    if (route != null) {
      // Use router to navigate — implementation depends on how router is accessible
    }
  }
}
```

## Local Notification Service

```dart
class LocalNotificationService {
  static final FlutterLocalNotificationsPlugin _plugin =
      FlutterLocalNotificationsPlugin();

  static Future<void> initialize() async {
    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    const iosSettings = DarwinInitializationSettings(
      requestAlertPermission: false,  // FCM handles this
      requestBadgePermission: false,
      requestSoundPermission: false,
    );

    await _plugin.initialize(
      const InitializationSettings(android: androidSettings, iOS: iosSettings),
      onDidReceiveNotificationResponse: _onNotificationTap,
    );
  }

  static Future<void> showFromRemoteMessage(RemoteMessage message) async {
    final notification = message.notification;
    if (notification == null) return;

    await _plugin.show(
      notification.hashCode,
      notification.title,
      notification.body,
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'default_channel',
          'Default',
          importance: Importance.high,
          priority: Priority.high,
        ),
        iOS: DarwinNotificationDetails(),
      ),
      payload: jsonEncode(message.data),
    );
  }

  static Future<void> scheduleNotification({
    required int id,
    required String title,
    required String body,
    required DateTime scheduledDate,
    String? payload,
  }) async {
    await _plugin.zonedSchedule(
      id,
      title,
      body,
      tz.TZDateTime.from(scheduledDate, tz.local),
      const NotificationDetails(
        android: AndroidNotificationDetails('scheduled', 'Scheduled'),
        iOS: DarwinNotificationDetails(),
      ),
      androidScheduleMode: AndroidScheduleMode.inexactAllowWhileIdle,
      payload: payload,
    );
  }

  static void _onNotificationTap(NotificationResponse response) {
    // Handle local notification tap
    if (response.payload != null) {
      final data = jsonDecode(response.payload!);
      // Navigate based on payload
    }
  }
}
```

## Edge Function: Send Push Notification

```typescript
// supabase/functions/send-push-notification/index.ts
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

serve(async (req) => {
  const { user_id, title, body, data } = await req.json()

  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!,
  )

  // Get user's FCM tokens
  const { data: tokens } = await supabase
    .from('fcm_tokens')
    .select('token')
    .eq('user_id', user_id)

  if (!tokens?.length) return new Response(JSON.stringify({ sent: 0 }))

  // Send via FCM HTTP v1 API
  const projectId = Deno.env.get('FIREBASE_PROJECT_ID')!
  const serviceAccount = JSON.parse(Deno.env.get('FIREBASE_SERVICE_ACCOUNT')!)

  // Get OAuth2 token for FCM
  const accessToken = await getFirebaseAccessToken(serviceAccount)

  let sent = 0
  for (const { token } of tokens) {
    const res = await fetch(
      `https://fcm.googleapis.com/v1/projects/${projectId}/messages:send`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          message: {
            token,
            notification: { title, body },
            data: data || {},
          },
        }),
      }
    )
    if (res.ok) sent++
  }

  return new Response(JSON.stringify({ sent }))
})
```

## Quiet Hours

Respect user's quiet hours when scheduling local notifications:

```dart
bool _isInQuietHours(NotificationSettings settings) {
  if (settings.quietHoursStart == null || settings.quietHoursEnd == null) return false;

  final now = TimeOfDay.now();
  final start = settings.quietHoursStart!;
  final end = settings.quietHoursEnd!;

  if (start.hour < end.hour) {
    // Same day range (e.g., 14:00 - 18:00)
    return now.hour >= start.hour && now.hour < end.hour;
  } else {
    // Overnight range (e.g., 22:00 - 08:00)
    return now.hour >= start.hour || now.hour < end.hour;
  }
}
```

## main.dart Integration

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  await LocalNotificationService.initialize();

  // ... other init

  runApp(const ProviderScope(child: MyApp()));
}

// In the app's initial authenticated state:
// await FCMService().initialize();
```
