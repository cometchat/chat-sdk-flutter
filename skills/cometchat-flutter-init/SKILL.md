---
name: cometchat-flutter-init
description: Use when initializing the CometChat Flutter SDK v5 in any Flutter app. Triggers on any mention of CometChat.init(), AppSettingsBuilder, 'set up CometChat', 'integrate CometChat', or questions about why CometChat methods are failing before init completes. Also use when adding CometChat to a new project or troubleshooting 'SDK not initialized' errors. Make sure to use this skill whenever the user mentions CometChat setup, SDK initialization, AppSettings configuration, or encounters ERR_NOT_INITIALIZED errors, even if they don't explicitly say 'init'.
---

# CometChat Flutter SDK v5 Initialization

Initialization is the gateway to every CometChat operation. Every failure mode downstream traces back to one of these init mistakes.

## Why This Matters

CometChat.init() is a `Future<void>` that uses callbacks (onSuccess/onError), not a direct return value. The SDK is not ready until onSuccess fires. Use a Completer to bridge to async/await.

**Failure modes this skill prevents:**
1. Calling SDK methods before init() completes
2. Building AppSettings with wrong region (must be lowercase: 'us', 'eu', 'in')
3. Running init() in a widget's build() method (re-inits on every rebuild)
4. Not checking getLoggedInUser() before calling login() on app resume
5. Ignoring the onError callback and silently swallowing init failures

## Initialization Sequence

```
App starts
  → WidgetsFlutterBinding.ensureInitialized()
  → CometChat.init(appId, appSettings, onSuccess:, onError:)
    → onError: Show error UI, do NOT proceed
    → onSuccess: CometChat.getLoggedInUser()
      → user != null: Session restored → go to chat
      → user == null: No session → go to login
```

Note: The SDK handles session restoration internally during init(). If a previous session exists, it restores it automatically. You still need to call getLoggedInUser() after init to check.

## Step 1: Build AppSettings

appId is NOT a property of AppSettingsBuilder — it's passed directly to CometChat.init(). The builder configures region, subscription, and connection settings.

```dart
String appId = "YOUR_APP_ID";
String region = "us"; // lowercase only: 'us', 'eu', or 'in'

AppSettings appSettings = (AppSettingsBuilder()
  ..subscriptionType = CometChatSubscriptionType.allUsers
  ..region = region
  ..autoEstablishSocketConnection = true  // default: true
).build();
```

Common mistakes:
- Region must be lowercase: 'us' not 'US' — the SDK validates against ['us', 'eu', 'in'] and throws ERR_INVALID_REGION
- Omitting subscriptionType silently disables presence events — no error thrown
- Setting autoEstablishSocketConnection to false means you must manually call CometChat.connect()

## Step 2: Call init() Once Using a Completer

CometChat.init() returns Future<void> but communicates results via callbacks. Use a Completer to bridge:

```dart
Future<void> initCometChat(String appId, AppSettings appSettings) async {
  final completer = Completer<void>();

  CometChat.init(appId, appSettings,
    onSuccess: (String successMessage) {
      debugPrint("CometChat init succeeded: $successMessage");
      completer.complete();
    },
    onError: (CometChatException e) {
      debugPrint("CometChat init failed — code: ${e.code}, message: ${e.message}");
      completer.completeError(e);
    },
  );

  return completer.future;
}
```

main.dart integration:

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  String appId = "YOUR_APP_ID";
  AppSettings appSettings = (AppSettingsBuilder()
    ..subscriptionType = CometChatSubscriptionType.allUsers
    ..region = "us"
    ..autoEstablishSocketConnection = true
  ).build();

  try {
    await initCometChat(appId, appSettings);
    runApp(const MyApp());
  } on CometChatException catch (e) {
    debugPrint("Fatal: CometChat could not initialize. Code: ${e.code}");
    runApp(InitErrorApp(error: e));
  }
}
```

WidgetsFlutterBinding.ensureInitialized() is required because Flutter's platform channel must be set up before runApp(). Without it: 'ServicesBinding was accessed before the binding was initialized'.

## Step 3: Check Existing Session Before Login

getLoggedInUser() returns Future<User?> and calls onSuccess only when user is non-null. You can await it directly or use callbacks:

```dart
// Callback style (matches SDK pattern)
User? existingUser;
await CometChat.getLoggedInUser(
  onSuccess: (user) => existingUser = user,
  onError: (_) {},
);

if (existingUser != null) {
  navigateToChatScreen(existingUser!);
} else {
  navigateToLoginScreen();
}
```

Or with a Completer for cleaner async:

```dart
Future<User?> getLoggedInUserOrNull() async {
  final completer = Completer<User?>();
  CometChat.getLoggedInUser(
    onSuccess: (User user) => completer.complete(user),
    onError: (CometChatException e) => completer.complete(null),
  );
  return completer.future;
}
```

Note: onSuccess receives `User` (non-nullable). The method returns `Future<User?>` — null means no session.

The SDK restores sessions automatically during init(). Calling login() on an already-authenticated session is unnecessary and wasteful.

## Placement Rules

| Location | OK? | Why |
|----------|-----|-----|
| main() before runApp() | ✅ | Runs once, blocks until ready |
| initState() of root widget | ✅ | Runs once per app lifecycle |
| build() of any widget | ❌ | Called on every rebuild — re-inits repeatedly |
| Inside FutureBuilder without guarding | ❌ | Rebuilds re-call init |
| After calling login() | ❌ | Login requires init to have completed |

Note: If you call init() with the same appId and region when already initialized, the SDK returns success immediately without re-initializing. If credentials change, it disposes the old instance and re-initializes.

## AppSettings Reference

| Property | Type | Required | Notes |
|----------|------|----------|-------|
| region | String | Yes | "us" / "eu" / "in". Validated against exact list. Lowercase only. |
| subscriptionType | String | No | CometChatSubscriptionType.allUsers / .none / .roles / .friends. Omitting disables presence. |
| autoEstablishSocketConnection | bool | No | Default true. Set false for manual WebSocket management via CometChat.connect(). |
| adminHost | String | No | Only for on-premise/dedicated deployments. |
| clientHost | String | No | Only for on-premise/dedicated deployments. |
| webSocketHost | String | No | Custom WebSocket host (advanced). |
| webSocketPort | String | No | Custom WebSocket port (advanced). |
| webSocketTimeout | int | No | WebSocket connection timeout in seconds (default: 5). |
| maxReconnectionAttempts | int | No | null = unlimited (default). Set positive number to limit. |
| loggerConfig | LoggerConfig | No | Enable SDK debug logging. Disabled by default. |

## Error Codes (Init-specific)

| Code | Meaning | Fix |
|------|---------|-----|
| ERR_INVALID_APP_ID | appId is empty or invalid | Check Dashboard credentials, ensure non-empty string |
| ERR_INVALID_REGION | Region not in ['us', 'eu', 'in'] | Use lowercase region matching your Dashboard |
| ERR_ALREADY_INITIALIZED | init() called again with same credentials | Safe to ignore — SDK returns success |
| ERR_INITIALIZATION_FAILED | General init failure | Check logs, verify network connectivity |
| ERR_NOT_INITIALIZED | SDK method called before init completes | Gate all SDK calls behind init onSuccess |

## Anti-Patterns

**init in build()** — build() is called on every setState(). This re-inits on every rebuild:
```dart
// ❌ WRONG
Widget build(BuildContext context) {
  CometChat.init(appId, appSettings, onSuccess: (_) {}, onError: (_) {});
  return Scaffold(...);
}
```

**login() before init() completes** — race condition:
```dart
// ❌ WRONG
CometChat.init(appId, appSettings, onSuccess: (_) {}, onError: (_) {});
CometChat.login(uid, authKey, onSuccess: (_) {}, onError: (_) {});
```

**login() without checking existing session:**
```dart
// First launch: init → login ✅
// Second launch: init → login again — wasteful, session already restored
// Always check getLoggedInUser() after init before attempting login.
```

**Putting appId in AppSettingsBuilder:**
```dart
// ❌ WRONG — appId is NOT a property of AppSettingsBuilder
AppSettingsBuilder()..appId = "APP_ID"  // This doesn't exist

// ✅ CORRECT — appId goes to CometChat.init()
CometChat.init("APP_ID", appSettings, onSuccess: ..., onError: ...);
```

## Retry Pattern for Network Failures

```dart
Future<void> initWithRetry(String appId, AppSettings appSettings, {int maxAttempts = 3}) async {
  for (int attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      await initCometChat(appId, appSettings);
      return;
    } on CometChatException catch (e) {
      if (e.code == "ERR_INITIALIZATION_FAILED" && attempt < maxAttempts) {
        final delay = Duration(seconds: attempt * 2);
        debugPrint("Init attempt $attempt failed. Retrying in ${delay.inSeconds}s...");
        await Future.delayed(delay);
      } else {
        rethrow;
      }
    }
  }
}
```

## Checklist

- WidgetsFlutterBinding.ensureInitialized() called before init()
- init() called exactly once per app lifecycle (not in build())
- appId passed to CometChat.init(), NOT to AppSettingsBuilder
- region is lowercase and matches Dashboard exactly ('us', 'eu', or 'in')
- onError callback is handled — not a no-op
- All downstream SDK calls gated behind init completion
- getLoggedInUser() checked after init() before attempting login()
- subscriptionType set if you need presence events
