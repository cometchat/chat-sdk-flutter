---
name: cometchat-flutter-core
description: Shared rules and patterns for any CometChat Flutter SDK v5 integration. Use as foundational context alongside a feature-specific skill. Do not use alone — pair with cometchat-flutter-init, cometchat-flutter-auth, cometchat-flutter-messaging, etc. Provides the Completer pattern, listener lifecycle rules, error handling conventions, and hard constraints that all feature skills inherit.
license: "MIT"
compatibility: "Flutter >=3.x; Dart >=3.x; cometchat_sdk ^5.0.0"
metadata:
  author: "CometChat"
  version: "1.0.0"
  tags: "chat cometchat flutter dart core patterns"
---

# CometChat Flutter SDK — Core Rules

These rules apply to every CometChat Flutter integration. They are hard constraints, not suggestions. Feature-specific skills must not contradict them.

---

## HARD RULES

```
INIT_FIRST
  CometChat.init() MUST resolve before login() or any other SDK call.
  Breaking this → ERR_NOT_INITIALIZED.

SESSION_CHECK
  Always getLoggedInUser() after init. Only login if null.
  Breaking this → wasteful redundant login, potential errors.

LISTENER_LIFECYCLE
  addListener in initState(), removeListener in dispose(). Always.
  Unique IDs per screen. Duplicate IDs silently replace previous listener.

COMPLETER_PATTERN
  All CometChat methods use callbacks. Wrap with Completer for async/await.

NEVER_EMPTY_ONERROR
  Every onError MUST log and surface the error. Never swallow.

APPID_GOES_TO_INIT
  appId is passed to CometChat.init(), NOT set on AppSettingsBuilder.

REGION_LOWERCASE
  Region: 'us', 'eu', 'in'. Validated against exact list. Case-sensitive.

ASYNC_SAFETY
  After any await on an SDK method, verify calling context is still valid
  before using the result. In Flutter: check `mounted` after EVERY await.
  This is the most commonly violated rule — even experienced developers miss it.
  Breaking this → setState on disposed widget, crash.

CACHE_SDK_REFS
  Cache SDK references (datasource, repositories) before entering async flows.
  Do not re-lookup via InheritedWidget after await points or in dispose().
  Breaking this → "Looking up a deactivated widget's ancestor" crash.

PUSH_CLEANUP_ON_LOGOUT
  CometChatNotifications.unregisterPushToken() before CometChat.logout().
```

---

## Completer Pattern (Callback → Async/Await Bridge)

CometChat methods use `onSuccess`/`onError` callbacks. Wrap them for clean async:

```dart
import 'dart:async';

// Generic wrapper — reuse for any CometChat callback method
Future<T> wrapCallback<T>(void Function(Completer<T> c) operation) {
  final completer = Completer<T>();
  operation(completer);
  return completer.future;
}

// Usage examples:
Future<void> initSDK(String appId, AppSettings settings) {
  return wrapCallback<void>((c) {
    CometChat.init(appId, settings,
      onSuccess: (_) => c.complete(),
      onError: (e) => c.completeError(e),
    );
  });
}

Future<User?> getLoggedInUser() {
  return wrapCallback<User?>((c) {
    CometChat.getLoggedInUser(
      onSuccess: (user) => c.complete(user),
      onError: (_) => c.complete(null),
    );
  });
}

Future<User> loginWithToken(String token) {
  return wrapCallback<User>((c) {
    CometChat.loginWithAuthToken(token,
      onSuccess: (user) => c.complete(user),
      onError: (e) => c.completeError(e),
    );
  });
}
```

### Defensive Completer: isCompleted guard and timeout

Some SDK methods (notably `getLoggedInUser`) may resolve both the callback and the Future, or may not call either callback in edge cases. Use an `isCompleted` guard and a timeout for safety:

```dart
Future<User?> getLoggedInUserSafe() async {
  try {
    final c = Completer<User?>();
    CometChat.getLoggedInUser(
      onSuccess: (u) {
        if (!c.isCompleted) c.complete(u);
      },
      onError: (_) {
        if (!c.isCompleted) c.complete(null);
      },
    );
    return c.future.timeout(
      const Duration(seconds: 5),
      onTimeout: () => null,
    );
  } catch (_) {
    return null;
  }
}
```

Use the `isCompleted` guard whenever a Completer might be completed more than once. Use the timeout pattern for session checks at app startup where hanging indefinitely is unacceptable.

---

## Listener Lifecycle Pattern

Every screen that receives real-time events follows this pattern:

```dart
class _ChatScreenState extends State<ChatScreen> with MessageListener {
  // 1. Unique ID — prevents collisions with other screens
  //    Use a widget property to make it unique per INSTANCE, not just per screen type.
  //    A static const works only if one instance exists at a time.
  late final _listenerId = "chat_screen_${widget.receiverUid}";
  // Or for guaranteed uniqueness: "chat_screen_${identityHashCode(this)}"

  @override
  void initState() {
    super.initState();
    // 2. Register in initState
    CometChat.addMessageListener(_listenerId, this);
  }

  @override
  void dispose() {
    // 3. Remove in dispose — prevents leaks and setState-after-dispose
    CometChat.removeMessageListener(_listenerId);
    super.dispose();
  }

  // 4. Override only the callbacks you need
  @override
  void onTextMessageReceived(TextMessage msg) {
    setState(() { _messages.add(msg); });
  }
}
```

### When you need BuildContext for listener registration

If listener registration requires `context` (e.g., to access an InheritedWidget like a RepositoryProvider), `initState` won't work because `context` isn't fully available yet. Use `didChangeDependencies` with an init guard, or `addPostFrameCallback`:

```dart
// Option A: didChangeDependencies with guard
bool _initialized = false;

@override
void didChangeDependencies() {
  super.didChangeDependencies();
  if (!_initialized) {
    _initialized = true;
    RepositoryProvider.of(context).datasource.addGroupListener(_lid, _listener);
    _loadData();
  }
}

// Option B: addPostFrameCallback in initState
@override
void initState() {
  super.initState();
  WidgetsBinding.instance.addPostFrameCallback((_) {
    RepositoryProvider.of(context).datasource.addMessageListener(_lid, this);
  });
}
```

Both are valid. The key rule remains: always remove in `dispose()`.

Listener types and their add/remove methods:

| Listener | Add | Remove |
|----------|-----|--------|
| MessageListener | addMessageListener | removeMessageListener |
| UserListener | addUserListener | removeUserListener |
| GroupListener | addGroupListener | removeGroupListener |
| ConnectionListener | addConnectionListener | removeConnectionListener |
| LoginListener | addloginListener (lowercase 'l') | removeLoginListener |
| CallListener | addCallListener | removeCallListener |

---

## Async Safety Pattern

SDK methods are async. Between `await` and response, the calling widget can be disposed. Always guard with `mounted` checks and cache references before async gaps.

```dart
// ✅ CORRECT — cache ref, check mounted after every await
Future<void> _init() async {
  final datasource = RepositoryProvider.of(context).datasource; // cache before async
  final user = await datasource.getLoggedInUser();
  if (!mounted) return;  // guard after await

  await fetchMessages();
  if (!mounted) return;  // guard again

  datasource.addMessageListener(_id, this); // use cached ref
}

@override
void dispose() {
  _datasource.removeListener(_id); // use cached ref, not context lookup
  super.dispose();
}
```

```dart
// ❌ WRONG — re-lookup after await, no mounted check
Future<void> _init() async {
  final user = await RepositoryProvider.of(context).datasource.getLoggedInUser();
  setState(() => _user = user); // widget may be disposed — crash

  // dispose uses context lookup — crashes on deactivated widget
}
```

---

## Error Handling Convention

```dart
// Every onError callback follows this pattern:
onError: (CometChatException e) {
  // 1. Log for debugging
  debugPrint("Operation failed: ${e.code} - ${e.message}");

  // 2. Surface to user (never swallow)
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(content: Text(mapErrorToUserMessage(e.code))),
  );
}
```

Error categories for retry decisions:

| Category | Retry? | Examples |
|----------|--------|---------|
| Network | Yes (backoff) | ERR_INITIALIZATION_FAILED |
| Auth | No (refresh token) | ERR_AUTH_TOKEN_EXPIRED |
| Validation | No (fix input) | ERROR_INVALID_RECEIVER_ID |
| State | No (fix call order) | ERR_NOT_INITIALIZED |
| Concurrency | Wait | ERROR_REQUEST_IN_PROGRESS |

---

## App Startup Flow (Canonical)

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // 1. Init
  final appSettings = (AppSettingsBuilder()
    ..subscriptionType = CometChatSubscriptionType.allUsers
    ..region = "us"
    ..autoEstablishSocketConnection = true
  ).build();

  await initSDK("APP_ID", appSettings);

  // 2. Session check
  final existingUser = await getLoggedInUser();

  // 3. Route
  runApp(MyApp(initialUser: existingUser));
}
```

---

## BaseMessage Subtype Checking

`BaseMessage` has no `.text` property. Always check the concrete type. Message lists from `fetchPrevious()`/`fetchNext()` contain ALL subtypes — not just chat messages:

```dart
for (var msg in messages) {
  // System events — not chat bubbles
  if (msg is Action) {
    // actionOn is BaseMessage → message-level (delete/edit) — typically skip
    // actionOn is User/GroupMember → group-level (join/leave/kick) — show as banner
    if (msg.actionOn is BaseMessage) continue;
    displaySystemBanner(msg.message ?? msg.action ?? 'Action');
    continue;
  }
  if (msg is Call) {
    displayCallBanner(msg);
    continue;
  }

  // Chat messages — render as bubbles
  if (msg is TextMessage) print(msg.text);
  else if (msg is MediaMessage) print(msg.attachment?.fileUrl);
  else if (msg is CustomMessage) print(msg.customData);
  else if (msg is InteractiveMessage) print(msg.interactiveData);
  else print("[${msg.type} message]");
}
```

The full subtype hierarchy of `BaseMessage`:
- `TextMessage` — text content (`.text`)
- `MediaMessage` — file/image/video/audio (`.attachment`, `.caption`)
- `CustomMessage` — app-defined payload (`.customData`, `.subType`)
- `InteractiveMessage` — forms, cards (`.interactiveData`)
- `Action` — system events (`.action`, `.actionOn`, `.actionBy`)
- `Call` — call events (`.callInitiator`, `.callReceiver`, `.sessionId`)

---

## Constants Quick Reference

```dart
// Receiver types
CometChatConversationType.user   // "user"
CometChatConversationType.group  // "group"

// Message types
CometChatMessageType.text    // "text"
CometChatMessageType.image   // "image"
CometChatMessageType.video   // "video"
CometChatMessageType.audio   // "audio"
CometChatMessageType.file    // "file"
CometChatMessageType.custom  // "custom"

// Group types
CometChatGroupType.public    // "public"
CometChatGroupType.private   // "private"
CometChatGroupType.password  // "password"

// Member scopes
CometChatMemberScope.admin       // "admin"
CometChatMemberScope.moderator   // "moderator"
CometChatMemberScope.participant  // "participant"

// Subscription types (for presence)
CometChatSubscriptionType.allUsers  // "allUsers"
CometChatSubscriptionType.roles     // "roles"
CometChatSubscriptionType.friends   // "friends"
CometChatSubscriptionType.none      // "none"

// Connection states
CometChatWSState.connected        // "connected"
CometChatWSState.connecting       // "connecting"
CometChatWSState.disconnected     // "disconnected"
CometChatWSState.featureThrottled // "featureThrottled"
```

---

## Pagination Pattern

```dart
// History (older messages) — no cursor needed
MessagesRequest request = (MessagesRequestBuilder()
  ..uid = "receiver-uid"  // or ..guid for groups
  ..limit = 30
).build();

request.fetchPrevious(onSuccess: ..., onError: ...);
// Call fetchPrevious() again on SAME object for next page

// Missed messages (newer) — cursor REQUIRED
MessagesRequest request = (MessagesRequestBuilder()
  ..uid = "receiver-uid"
  ..limit = 30
  ..messageId = lastKnownMessageId  // cursor
).build();

request.fetchNext(onSuccess: ..., onError: ...);
```

---

## API Surface Reference

For the complete list of all models, methods, listeners, and constants, see:
`doc/api-surface.json`

For error codes and fixes, see:
`doc/error-catalog.md`
