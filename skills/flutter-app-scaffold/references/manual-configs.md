# Manual Configuration Steps

This file contains detailed instructions for every manual step the user may encounter during scaffolding. When a phase requires one of these steps, present the instructions clearly and wait for the user to confirm completion.

## Table of Contents
1. [Apple Developer Account & App ID](#apple-developer)
2. [Apple Sign-In Configuration](#apple-sign-in)
3. [Google Cloud Console & OAuth](#google-oauth)
4. [Firebase Project Setup](#firebase-setup)
5. [Firebase Cloud Messaging (APNs)](#fcm-apns)
6. [RevenueCat Setup](#revenuecat)
7. [App Store Connect Setup](#app-store-connect)
8. [Google Play Console Setup](#google-play-console)

---

## Apple Developer Account & App ID {#apple-developer}

**When needed:** Before first iOS build, Apple Sign-In, push notifications, or App Store submission.

```
⏸️ MANUAL STEP REQUIRED: Apple Developer Setup

You'll need an Apple Developer account ($99/year) and need to register your app.

1. Go to https://developer.apple.com/account
2. Sign up or log in
3. Go to "Certificates, Identifiers & Profiles" → "Identifiers"
4. Click the "+" button to register a new App ID
5. Select "App IDs" → Continue
6. Select "App" → Continue
7. Fill in:
   - Description: [Your App Name]
   - Bundle ID: Select "Explicit" and enter: com.[yourorg].[appname]
8. Under Capabilities, enable any you need:
   - ☑️ Sign In with Apple (if using Apple auth)
   - ☑️ Push Notifications (if using notifications)
   - ☑️ App Groups (if using home widgets)
9. Click "Continue" → "Register"

Tell me the Bundle ID you registered (e.g., com.mycompany.myapp).
```

---

## Apple Sign-In Configuration {#apple-sign-in}

**When needed:** During auth phase if Apple Sign-In is selected.

```
⏸️ MANUAL STEP REQUIRED: Configure Sign In with Apple

Part 1 — App ID (skip if already done above):
Ensure "Sign In with Apple" is enabled for your App ID in Apple Developer Portal.

Part 2 — Service ID (needed for Supabase):
1. Go to "Certificates, Identifiers & Profiles" → "Identifiers"
2. Click "+" → Select "Services IDs" → Continue
3. Fill in:
   - Description: [App Name] Sign In
   - Identifier: com.[yourorg].[appname].signin (note: different from bundle ID)
4. Click "Continue" → "Register"
5. Click on the newly created Service ID
6. Enable "Sign In with Apple" checkbox
7. Click "Configure" next to it
8. Set:
   - Primary App ID: Select your app
   - Domains: [your-supabase-project-ref].supabase.co
   - Return URLs: https://[your-supabase-project-ref].supabase.co/auth/v1/callback
9. Save

Part 3 — Configure in Supabase Dashboard:
1. Go to your Supabase project → Authentication → Providers
2. Enable "Apple"
3. Enter your Service ID (com.[yourorg].[appname].signin)
4. Save

Let me know when all three parts are done.
```

---

## Google Cloud Console & OAuth {#google-oauth}

**When needed:** During auth phase if Google Sign-In is selected.

```
⏸️ MANUAL STEP REQUIRED: Configure Google Sign-In

Part 1 — Google Cloud Project:
1. Go to https://console.cloud.google.com
2. Create a new project (or select existing one)
3. Name it after your app

Part 2 — OAuth Consent Screen:
1. Go to "APIs & Services" → "OAuth consent screen"
2. Select "External" → Create
3. Fill in:
   - App name: [Your App Name]
   - User support email: [your email]
   - Developer contact email: [your email]
4. Add scopes: email, profile, openid
5. Save and continue through the remaining steps

Part 3 — Create OAuth Credentials:
You need up to 3 client IDs depending on your platforms:

**iOS Client ID:**
1. Go to "APIs & Services" → "Credentials" → "Create Credentials" → "OAuth client ID"
2. Application type: iOS
3. Bundle ID: [your bundle ID from Apple setup]
4. Create and save the Client ID

**Web Client ID (required for Supabase):**
1. Create another OAuth client ID
2. Application type: Web application
3. Name: [App Name] Web
4. Authorized redirect URIs: https://[your-supabase-ref].supabase.co/auth/v1/callback
5. Create and save the Client ID and Client Secret

**Android Client ID:**
1. Create another OAuth client ID
2. Application type: Android
3. Package name: com.[yourorg].[appname]
4. SHA-1 fingerprint: Run this command and paste the result:
   keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android
5. Create

Part 4 — Configure in Supabase:
1. Go to Supabase Dashboard → Authentication → Providers
2. Enable "Google"
3. Enter the Web Client ID and Web Client Secret
4. Save

I need these values from you:
- iOS Client ID
- Web Client ID
- Web Client Secret (for .env only, never commit this)
```

---

## Firebase Project Setup {#firebase-setup}

**When needed:** Before notifications, crashlytics, or analytics setup.

```
⏸️ MANUAL STEP REQUIRED: Create Firebase Project

1. Go to https://console.firebase.google.com
2. Click "Add project"
3. Enter your project name
4. Enable/disable Google Analytics (recommended: enable)
5. If enabling Analytics, select or create a Google Analytics account
6. Click "Create project" and wait

Now register your app platforms:

**iOS:**
1. Click the iOS icon to add an iOS app
2. Bundle ID: [your bundle ID]
3. App nickname: [Your App Name]
4. Download `GoogleService-Info.plist`
5. I'll tell you where to place it

**Android:**
1. Click "Add app" → Android
2. Package name: com.[yourorg].[appname]
3. App nickname: [Your App Name]
4. SHA-1: (same as Google Sign-In step above)
5. Download `google-services.json`
6. I'll tell you where to place it

Place the files here:
- iOS: ios/Runner/GoogleService-Info.plist
- Android: android/app/google-services.json

Let me know when you've downloaded and placed both files.
```

After confirmation, run:
```bash
flutterfire configure
```
to generate `firebase_options.dart`.

---

## Firebase Cloud Messaging (APNs) {#fcm-apns}

**When needed:** During notifications phase for iOS push notifications.

```
⏸️ MANUAL STEP REQUIRED: Configure APNs for iOS Push Notifications

Push notifications on iOS require an APNs authentication key.

Part 1 — Create APNs Key:
1. Go to Apple Developer Portal → "Keys"
2. Click "+" to create a new key
3. Name: [App Name] Push Key
4. Enable "Apple Push Notifications service (APNs)"
5. Click "Continue" → "Register"
6. Download the .p8 key file — you can only download it ONCE
7. Note the Key ID shown on the page
8. Note your Team ID (visible in top right of developer portal, or Account → Membership)

Part 2 — Upload to Firebase:
1. Go to Firebase Console → Project Settings → Cloud Messaging tab
2. Under "Apple app configuration", click "Upload" next to APNs Authentication Key
3. Upload the .p8 file
4. Enter Key ID and Team ID
5. Save

Part 3 — Enable Push Notifications Capability:
1. Open your project in Xcode (ios/Runner.xcworkspace)
2. Select the Runner target → "Signing & Capabilities" tab
3. Click "+ Capability" → Add "Push Notifications"
4. Also add "Background Modes" → check "Remote notifications"

Let me know when all three parts are done.
```

---

## RevenueCat Setup {#revenuecat}

**When needed:** During subscriptions/monetization phase.

```
⏸️ MANUAL STEP REQUIRED: Set Up RevenueCat

Part 1 — Create RevenueCat Account:
1. Go to https://www.revenuecat.com and sign up
2. Create a new project with your app name

Part 2 — Connect App Store (iOS):
1. In RevenueCat, go to Project Settings → Apps → "New App"
2. Select "Apple App Store"
3. Enter your app's Bundle ID
4. You'll need an App Store Connect Shared Secret:
   a. Go to App Store Connect → your app → General → App Information
   b. Under "App-Specific Shared Secret", click "Manage"
   c. Generate and copy the secret
5. Paste into RevenueCat
6. Save

Part 3 — Connect Google Play (Android):
1. In RevenueCat, click "New App" → "Google Play Store"
2. Enter your package name
3. You'll need a Google Play Service Account JSON key:
   a. Go to Google Play Console → Setup → API access
   b. Create or link a Google Cloud project
   c. Create a Service Account with "Finance" permissions
   d. Download the JSON key
5. Upload to RevenueCat
6. Save

Part 4 — Create Products:
1. In App Store Connect: Go to your app → Subscriptions → create subscription group and products
2. In Google Play Console: Go to your app → Monetize → Products → Subscriptions → create products
3. In RevenueCat: Go to Products → add your products, mapping them to store product IDs
4. Create an Offering (the set of products shown to users) and add your products as Packages

Part 5 — Get API Key:
1. In RevenueCat → Project Settings → API Keys
2. Copy the public API key for each platform

I need from you:
- RevenueCat iOS API key
- RevenueCat Android API key (if targeting Android)
- Product identifiers you created
```

---

## App Store Connect Setup {#app-store-connect}

**When needed:** Before first iOS release or TestFlight beta.

```
⏸️ MANUAL STEP REQUIRED: Set Up App Store Connect

1. Go to https://appstoreconnect.apple.com
2. Click "My Apps" → "+" → "New App"
3. Fill in:
   - Platform: iOS
   - Name: [Your App Name]
   - Primary Language: English (or your language)
   - Bundle ID: Select the one you registered
   - SKU: [your-app-name] (unique identifier, can be anything)
4. Click "Create"

For TestFlight (beta testing):
1. Go to your app → TestFlight tab
2. You can upload builds once the app is built and signed
3. Add internal testers (your Apple ID) or create an external test group

You don't need to fill in all the App Store listing details yet — 
that's only needed for public release. TestFlight just needs the build.

Let me know when the app is created in App Store Connect.
```

---

## Google Play Console Setup {#google-play-console}

**When needed:** Before first Android release.

```
⏸️ MANUAL STEP REQUIRED: Set Up Google Play Console

1. Go to https://play.google.com/console
2. Sign up for a developer account ($25 one-time fee) if you haven't
3. Click "Create app"
4. Fill in:
   - App name: [Your App Name]
   - Default language: English
   - App or Game: App
   - Free or Paid: Free (you can still have in-app purchases)
5. Complete the declarations (content rating, target audience, etc.)
6. Click "Create app"

For internal testing:
1. Go to Testing → Internal testing
2. Create a new release
3. Upload your APK or AAB file
4. Create a testers list and add email addresses
5. Copy the test link to share with testers

Let me know when the app is created in Google Play Console.
```
