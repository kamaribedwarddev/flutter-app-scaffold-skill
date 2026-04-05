# Home Screen Widgets Module

Native home screen widgets for iOS (WidgetKit) and Android (AppWidget) using the `home_widget` Flutter package.

## Files to Create

```
lib/features/home_widget/
├── data/
│   └── services/
│       └── home_widget_service.dart
└── presentation/
    └── providers/
        └── home_widget_providers.dart
```

Plus native platform files (see Platform Setup below).

## What Home Widgets Can Display

Home widgets are read-only views that show data at a glance. They work well for:
- Summary stats (daily count, streak, progress)
- Upcoming items (events, deadlines, reminders)
- Quick status (last sync, current state)
- Motivational content (daily quote, achievement progress)

They do NOT support:
- Complex interactivity (no scrolling, no text input)
- Real-time updates (they refresh on a schedule, not live)
- Large data sets

During the discovery phase, decide what data is most valuable at a glance for the user.

## Flutter Integration

### Home Widget Service

```dart
class HomeWidgetService {
  static const String appGroupId = 'group.com.[org].[appname]';

  /// Save data that the widget will display
  static Future<void> updateWidgetData({
    required Map<String, dynamic> data,
  }) async {
    for (final entry in data.entries) {
      await HomeWidget.saveWidgetData<String>(
        entry.key,
        entry.value is String ? entry.value : jsonEncode(entry.value),
      );
    }
    // Tell the OS to refresh the widget
    await HomeWidget.updateWidget(
      iOSName: 'AppWidget',           // Must match iOS widget extension name
      androidName: 'AppWidgetProvider', // Must match Android provider class
    );
  }

  /// Register callback for widget tap
  static Future<void> registerInteractivity() async {
    await HomeWidget.registerInteractivityCallback(widgetTapCallback);
  }

  /// Called when user taps the widget
  @pragma('vm:entry-point')
  static Future<void> widgetTapCallback(Uri? uri) async {
    if (uri != null) {
      // Handle deep link from widget tap
      // e.g., navigate to specific screen
    }
  }
}
```

### Update Widget on App State Changes

```dart
// In your app's main shell or wherever state changes:
ref.listen(someDataProvider, (prev, next) {
  next.whenData((data) {
    HomeWidgetService.updateWidgetData(data: {
      'title': data.title,
      'subtitle': data.subtitle,
      'count': data.items.length.toString(),
      'lastUpdated': DateTime.now().toIso8601String(),
    });
  });
});
```

### Refresh on App Resume

```dart
// In your app's main widget
@override
void didChangeAppLifecycleState(AppLifecycleState state) {
  if (state == AppLifecycleState.resumed) {
    // Refresh widget data when app comes to foreground
    _refreshWidgetData();
  }
}
```

## Platform Setup

### ⏸️ MANUAL STEP: iOS Widget Extension

```
⏸️ MANUAL STEP REQUIRED: Create iOS Widget Extension

1. Open ios/Runner.xcworkspace in Xcode
2. File → New → Target
3. Search for "Widget Extension"
4. Name it: [AppName]Widget
5. Uncheck "Include Live Activity" and "Include Configuration App Intent"
6. Click "Finish" → "Activate" when prompted

Now configure App Groups (needed for data sharing):
7. Select the Runner target → Signing & Capabilities
8. Click "+ Capability" → "App Groups"
9. Add: group.com.[org].[appname]
10. Select the Widget extension target → Signing & Capabilities
11. Add the SAME App Group: group.com.[org].[appname]

CRITICAL BUILD FIX:
12. Select the Widget extension target → Build Settings
13. Search for "User Script Sandboxing"
14. Set it to "No" — this fixes the build cycle error with Flutter

Let me know when the widget extension is created.
```

After confirmation, create the widget Swift code:

```swift
// ios/[AppName]Widget/[AppName]Widget.swift
import WidgetKit
import SwiftUI

struct AppEntry: TimelineEntry {
    let date: Date
    let title: String
    let subtitle: String
    let count: Int
}

struct AppWidgetProvider: TimelineProvider {
    let appGroupId = "group.com.[org].[appname]"

    func placeholder(in context: Context) -> AppEntry {
        AppEntry(date: Date(), title: "Loading...", subtitle: "", count: 0)
    }

    func getSnapshot(in context: Context, completion: @escaping (AppEntry) -> Void) {
        let entry = getEntryFromUserDefaults()
        completion(entry)
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<AppEntry>) -> Void) {
        let entry = getEntryFromUserDefaults()
        let nextUpdate = Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
        let timeline = Timeline(entries: [entry], policy: .after(nextUpdate))
        completion(timeline)
    }

    private func getEntryFromUserDefaults() -> AppEntry {
        let defaults = UserDefaults(suiteName: appGroupId)
        let title = defaults?.string(forKey: "title") ?? "No data"
        let subtitle = defaults?.string(forKey: "subtitle") ?? ""
        let count = Int(defaults?.string(forKey: "count") ?? "0") ?? 0
        return AppEntry(date: Date(), title: title, subtitle: subtitle, count: count)
    }
}

struct AppWidgetEntryView: View {
    var entry: AppEntry

    var body: some View {
        VStack(alignment: .leading, spacing: 4) {
            Text(entry.title)
                .font(.headline)
            Text(entry.subtitle)
                .font(.caption)
                .foregroundColor(.secondary)
            Spacer()
            Text("\(entry.count)")
                .font(.title)
                .bold()
        }
        .padding()
    }
}

@main
struct AppWidget: Widget {
    let kind: String = "AppWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: AppWidgetProvider()) { entry in
            AppWidgetEntryView(entry: entry)
        }
        .configurationDisplayName("[App Name]")
        .description("Shows your latest data at a glance.")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}
```

### Android Widget Setup

Create these files:

**android/app/src/main/res/layout/widget_layout.xml:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp"
    android:background="@drawable/widget_background">

    <TextView
        android:id="@+id/widget_title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="16sp"
        android:textStyle="bold" />

    <TextView
        android:id="@+id/widget_subtitle"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="12sp" />

    <TextView
        android:id="@+id/widget_count"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="32sp"
        android:textStyle="bold" />
</LinearLayout>
```

**android/app/src/main/res/xml/widget_info.xml:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="180dp"
    android:minHeight="110dp"
    android:updatePeriodMillis="3600000"
    android:initialLayout="@layout/widget_layout"
    android:resizeMode="horizontal|vertical"
    android:widgetCategory="home_screen" />
```

**Kotlin Widget Provider:**
```kotlin
// android/app/src/main/kotlin/.../AppWidgetProvider.kt
class AppWidgetProvider : HomeWidgetProvider() {
    override fun onUpdate(
        context: Context,
        appWidgetManager: AppWidgetManager,
        appWidgetIds: IntArray,
        widgetData: SharedPreferences
    ) {
        appWidgetIds.forEach { widgetId ->
            val views = RemoteViews(context.packageName, R.layout.widget_layout).apply {
                setTextViewText(R.id.widget_title, widgetData.getString("title", "No data"))
                setTextViewText(R.id.widget_subtitle, widgetData.getString("subtitle", ""))
                setTextViewText(R.id.widget_count, widgetData.getString("count", "0"))
            }
            appWidgetManager.updateAppWidget(widgetId, views)
        }
    }
}
```

Register in AndroidManifest.xml:
```xml
<receiver android:name=".AppWidgetProvider" android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/widget_info" />
</receiver>
```
