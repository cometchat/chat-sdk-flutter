---
name: cometchat-flutter-auth
description: Use when implementing login, logout, or session management with CometChat Flutter SDK v5. Triggers on mentions of CometChat.login(), loginWithAuthToken, getLoggedInUser, logout, auth tokens, createUser, or errors like ERR_ALREADY_LOGGED_IN, ERROR_USER_NOT_LOGGED_IN. Also use when the user is setting up user authentication flows, handling app resume sessions, or creating new CometChat users before login. Make sure to use this skill whenever the user mentions CometChat authentication, even if they just say 'connect the user', 'sign in to chat', or 'how do I log out'.
---

# CometChat Flutter SDK v5 Authentication

CometChat does not handle user management. Your app handles registration and login. Once your user is authenticated in your app, you log them into CometChat programmatically.

## Auth Flow Overview

```
SDK initialized (via CometChat.init)
  → CometChat.getLoggedInUser()
    → user != null: Session already active → go to chat
    → user == null: No session
      → Your server generates auth token
      → CometChat.loginWithAuthToken(authToken)
        → onSuccess: User logged in → go to chat
        → onError: Handle failure
```

The SDK restores sessions automatically during init(). Always check getLoggedInUser() before calling login. Calling login on an active session is wasteful.

## Two Login Methods

### loginWithAuthToken (Production — recommended)

Your server creates an auth token via the CometChat REST API, passes it to the client. The auth key never touches client code.

```dart
String authToken = "AUTH_TOKEN_FROM_YOUR_SERVER";

final existingUser = await CometChat.getLoggedInUser();
if (existingUser == null) {
  await CometChat.loginWithAuthToken(authToken,
    onSuccess: (User user) {
      debugPrint("Login Successful: ${user.uid}");
    },
    onError: (CometChatException e) {
      debugPrint("Login failed: ${e.code} - ${e.message}");
    },
  );
}
```

**Signature:** `static Future<User?> loginWithAuthToken(String authToken, {required Function(User)? onSuccess, required Function(CometChatException)? onError})`

### login with Auth Key (Testing only — deprecated)

Uses the auth key directly in client code. Only for POC/development.

```dart
String uid = "cometchat-uid-1";
String authKey = "AUTH_KEY";

final existingUser = await CometChat.getLoggedInUser();
if (existingUser == null) {
  await CometChat.login(uid, authKey,
    onSuccess: (User user) {
      debugPrint("Login Successful: ${user.uid}");
    },
    onError: (CometChatException e) {
      debugPrint("Login failed: ${e.code} - ${e.message}");
    },
  );
}
```

**Signature:** `@Deprecated static Future<User?> login(String uid, String authKey, {required Function(User)? onSuccess, required Function(CometChatException)? onError})`

## Session Check: getLoggedInUser

Returns `Future<User?>`. onSuccess only fires when user is non-null. onSuccess receives `User` (non-nullable).

```dart
// Simple pattern — await the Future directly
final user = await CometChat.getLoggedInUser(
  onSuccess: (user) => debugPrint("Session found: ${user.uid}"),
  onError: (_) {},
);

if (user != null) {
  // Session restored — go to chat
} else {
  // No session — show login screen
}
```

Or wrap with a Completer for cleaner async:

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

**Signature:** `static Future<User?> getLoggedInUser({Function(User)? onSuccess, Function(CometChatException)? onError})`

Note: onSuccess and onError are optional here (not required), unlike login/logout.

## Create User

Users must exist in CometChat before they can log in. For production, create users server-side via the REST API. The client-side method exists for testing.

```dart
User newUser = User(uid: "user_123", name: "Jane Doe");
String authKey = "AUTH_KEY";

await CometChat.createUser(newUser, authKey,
  onSuccess: (User user) {
    debugPrint("User created: ${user.uid}");
    // Now login the user
  },
  onError: (CometChatException e) {
    debugPrint("Create user failed: ${e.code} - ${e.message}");
  },
);
```

**Signature:** `static Future<User?> createUser(User user, String authKey, {required Function(User)? onSuccess, required Function(CometChatException)? onError})`

## Logout

Cancels all stream subscriptions automatically and notifies LoginListeners. Call this when your user logs out of your app. Always unregister push notification tokens before logout.

```dart
// 1. Unregister push token first (prevents notifications after logout)
CometChatNotifications.unregisterPushToken(
  onSuccess: (String response) {
    debugPrint("Push token unregistered: $response");
  },
  onError: (CometChatException e) {
    debugPrint("Push token unregister failed: ${e.message}");
    // Continue with logout even if this fails
  },
);

// 2. Then logout
CometChat.logout(
  onSuccess: (String message) {
    debugPrint("Logout successful: $message");
    // Navigate to login screen, clear local state
  },
  onError: (CometChatException e) {
    debugPrint("Logout failed: ${e.code} - ${e.message}");
  },
);
```

**Signature:** `static Future<void> logout({required Function(String)? onSuccess, required Function(CometChatException)? onError})`

What logout does internally:
1. Cancels all real-time stream subscriptions (messages, typing, receipts, presence, etc.)
2. Calls the auth repository logout (clears session on server)
3. Notifies all LoginListeners (logoutSuccess or logoutFailure)

## LoginListener

Get real-time notifications for login/logout events across the app.

```dart
class MyAuthHandler with LoginListener {
  @override
  void loginSuccess(User user) {
    debugPrint("User logged in: ${user.uid}");
  }

  @override
  void loginFailure(CometChatException e) {
    debugPrint("Login failed: ${e.message}");
  }

  @override
  void logoutSuccess() {
    debugPrint("User logged out");
  }

  @override
  void logoutFailure(CometChatException e) {
    debugPrint("Logout failed: ${e.message}");
  }
}

// Register
CometChat.addloginListener("auth_listener", MyAuthHandler());

// Remove when no longer needed
CometChat.removeLoginListener("auth_listener");
```

Note: The add method is `addloginListener` (lowercase 'l') — this is the actual SDK API.

## Completer-Based Auth Service Pattern

For production apps, wrap the callback-based SDK in a clean async service:

```dart
class AuthService {
  Future<User> login(String uid, String authKey) async {
    final completer = Completer<User>();
    CometChat.login(uid, authKey,
      onSuccess: (user) => completer.complete(user),
      onError: (e) => completer.completeError(e),
    );
    return completer.future;
  }

  Future<User> loginWithAuthToken(String authToken) async {
    final completer = Completer<User>();
    CometChat.loginWithAuthToken(authToken,
      onSuccess: (user) => completer.complete(user),
      onError: (e) => completer.completeError(e),
    );
    return completer.future;
  }

  Future<void> logout() async {
    // Unregister push token before logout
    final tokenCompleter = Completer<void>();
    CometChatNotifications.unregisterPushToken(
      onSuccess: (_) => tokenCompleter.complete(),
      onError: (_) => tokenCompleter.complete(), // Continue even if fails
    );
    await tokenCompleter.future;

    final completer = Completer<void>();
    CometChat.logout(
      onSuccess: (_) => completer.complete(),
      onError: (e) => completer.completeError(e),
    );
    return completer.future;
  }

  Future<User?> getLoggedInUser() async {
    final completer = Completer<User?>();
    CometChat.getLoggedInUser(
      onSuccess: (user) => completer.complete(user),
      onError: (e) => completer.complete(null),
    );
    return completer.future;
  }
}
```

## Complete App Startup Flow

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // 1. Init SDK
  final initCompleter = Completer<void>();
  CometChat.init(appId, appSettings,
    onSuccess: (_) => initCompleter.complete(),
    onError: (e) => initCompleter.completeError(e),
  );
  await initCompleter.future;

  // 2. Check existing session
  User? existingUser;
  await CometChat.getLoggedInUser(
    onSuccess: (user) => existingUser = user,
    onError: (_) {},
  );

  // 3. Route based on session state
  runApp(MyApp(initialUser: existingUser));
}
```

## Anti-Patterns

**login() before init() completes:**
```dart
// ❌ WRONG — race condition, SDK not ready
CometChat.init(appId, appSettings, onSuccess: (_) {}, onError: (_) {});
CometChat.login(uid, authKey, onSuccess: (_) {}, onError: (_) {});
```

**Skipping session check:**
```dart
// ❌ WRONG — calls login every app start, wasteful
await initCometChat();
await CometChat.login(uid, authKey, ...); // Session may already exist!
```

**Auth key in production code:**
```dart
// ❌ WRONG — auth key exposed in client
CometChat.login(uid, "YOUR_AUTH_KEY_HERE", ...);

// ✅ CORRECT — auth token from your server
CometChat.loginWithAuthToken(tokenFromServer, ...);
```

**Not handling logout cleanup:**
```dart
// ❌ WRONG — just navigating away without logout
Navigator.pushReplacementNamed(context, '/login');

// ❌ WRONG — logout without unregistering push token
CometChat.logout(
  onSuccess: (_) => Navigator.pushReplacementNamed(context, '/login'),
  onError: (e) => showError(e.message),
);
// User still receives push notifications after logout!

// ✅ CORRECT — unregister push token, then logout, then navigate
CometChatNotifications.unregisterPushToken(
  onSuccess: (_) {},
  onError: (_) {},
);
CometChat.logout(
  onSuccess: (_) => Navigator.pushReplacementNamed(context, '/login'),
  onError: (e) => showError(e.message),
);
```

**Empty onError callbacks:**
```dart
// ❌ WRONG — silently swallows errors
CometChat.login(uid, authKey, onSuccess: (_) {}, onError: (_) {});

// ✅ CORRECT — handle and surface errors
CometChat.login(uid, authKey,
  onSuccess: (user) => navigateToChat(user),
  onError: (e) {
    debugPrint("Login failed: ${e.code}");
    showUserFriendlyError(e);
  },
);
```

## Error Codes (Auth-specific)

| Code | Meaning | Fix |
|------|---------|-----|
| ERR_NOT_INITIALIZED | SDK not initialized before auth call | Ensure init() completes before login/logout |
| ERR_AUTH_TOKEN_EXPIRED | Auth token has expired | Generate new token from your server |
| ERROR_USER_NOT_LOGGED_IN | Calling methods that require auth | Login first |
| ERROR_UNHANDLED_EXCEPTION | Unexpected error during auth | Check stack trace, verify network |

## Checklist

- getLoggedInUser() checked after init() before attempting login()
- Using loginWithAuthToken for production (not login with auth key)
- Auth key never hardcoded in client code for production
- onError callbacks handle and surface errors (not empty)
- CometChatNotifications.unregisterPushToken() called before logout()
- logout() called before navigating to login screen
- LoginListener registered if you need cross-app auth event notifications
- All auth calls gated behind init() completion
