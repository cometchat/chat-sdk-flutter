---
name: cometchat-flutter-realtime
description: Use when subscribing to real-time events with CometChat Flutter SDK v5. Triggers on mentions of MessageListener, UserListener, GroupListener, ConnectionListener, typing indicators, read receipts, delivery receipts, presence status, online/offline detection, startTyping, endTyping, markAsRead, markAsDelivered, or WebSocket connection state. Also use when the user asks about real-time updates, live message delivery, why presence events aren't working, or listener lifecycle management. Make sure to use this skill whenever the user mentions real-time chat features, listener registration, or event subscriptions, even if they just say 'show when user is typing' or 'detect online users'.
---

# CometChat Flutter SDK v5 Real-Time Events

Five listener types handle all real-time events. Each follows the same lifecycle: register in initState, remove in dispose, use unique IDs.

## Listener Types Quick Reference

| Listener | Register | Remove | Events |
|----------|----------|--------|--------|
| MessageListener | addMessageListener(id, this) | removeMessageListener(id) | Messages, typing, receipts, reactions, edits, deletes, moderation, AI messages |
| UserListener | addUserListener(id, this) | removeUserListener(id) | onUserOnline, onUserOffline |
| GroupListener | addGroupListener(id, this) | removeGroupListener(id) | Member joined/left/kicked/banned/unbanned, scope changed |
| ConnectionListener | addConnectionListener(id, this) | removeConnectionListener(id) | onConnected, onConnecting, onDisconnected, onFeatureThrottled, onConnectionError |
| LoginListener | addloginListener(id, this) | removeLoginListener(id) | loginSuccess, loginFailure, logoutSuccess, logoutFailure |
| AIAssistantListener | addAIAssistantListener(id, this) | removeAIAssistantListener(id) | onAIAssistantEventReceived |

Note: `addloginListener` has lowercase 'l' — this is the actual SDK API.

## Presence (Online/Offline)

Requires `subscriptionType` set during init. Without it, no presence events fire — no error thrown.

```dart
// During init — REQUIRED for presence to work
AppSettings appSettings = (AppSettingsBuilder()
  ..subscriptionType = CometChatSubscriptionType.allUsers
  ..region = "us"
).build();
```

Three subscription options:
- `CometChatSubscriptionType.allUsers` — all users (use unless >10k users)
- `CometChatSubscriptionType.roles` — specific roles only
- `CometChatSubscriptionType.friends` — friends only

```dart
class _ContactsScreenState extends State<ContactsScreen> with UserListener {
  static const _listenerId = "contacts_presence_listener";

  @override
  void initState() {
    super.initState();
    CometChat.addUserListener(_listenerId, this);
  }

  @override
  void dispose() {
    CometChat.removeUserListener(_listenerId);
    super.dispose();
  }

  @override
  void onUserOnline(User user) {
    // user.status == 'online', user.lastActiveAt updated
    setState(() { _updateUserStatus(user); });
  }

  @override
  void onUserOffline(User user) {
    // user.status == 'offline', user.lastActiveAt holds last seen time
    setState(() { _updateUserStatus(user); });
  }
}
```

## Typing Indicators

Send typing status via CometChat, receive via MessageListener.

```dart
// Send — start typing
CometChat.startTyping(
  receiverUid: "cometchat-uid-1",
  receiverType: CometChatReceiverType.user,  // or .group with GUID
);

// Send — stop typing
CometChat.endTyping(
  receiverUid: "cometchat-uid-1",
  receiverType: CometChatReceiverType.user,
);
```

Receive via MessageListener:

```dart
@override
void onTypingStarted(TypingIndicator typingIndicator) {
  // typingIndicator.sender — User who is typing
  // typingIndicator.receiverId — UID or GUID
  // typingIndicator.receiverType — "user" or "group"
  setState(() { _isTyping = true; });
}

@override
void onTypingEnded(TypingIndicator typingIndicator) {
  setState(() { _isTyping = false; });
}
```

The SDK also exposes `CometChat.onTypingIndicator()` which returns a `Stream<TypingIndicator>` for stream-based consumption.

### Typing on a Conversation List (Multi-Conversation)

The single-chat example above uses a boolean `_isTyping`. On a conversation list screen, you need to map typing events to specific conversations using a keyed map.

Key: build a conversation key from `TypingIndicator` fields to match against your conversation list.

```dart
// For 1:1 chats, the key is the sender's UID.
// For groups, the key is the receiverId (the GUID).
String _typingKey(TypingIndicator indicator) {
  return indicator.receiverType == 'group'
      ? indicator.receiverId
      : indicator.sender.uid;
}
```

Track typing state as a map of conversation key → typer's name:

```dart
// State: conversationKey → name of user typing
final Map<String, String> _typingMap = {};

@override
void onTypingStarted(TypingIndicator indicator) {
  final key = _typingKey(indicator);
  setState(() => _typingMap[key] = indicator.sender.name);
}

@override
void onTypingEnded(TypingIndicator indicator) {
  final key = _typingKey(indicator);
  setState(() => _typingMap.remove(key));
}
```

Then when rendering a conversation row, check `_typingMap` using the conversation partner's UID (for users) or GUID (for groups):

```dart
// Look up typing status for this conversation
final entity = conversation.conversationWith;
final String convKey;
if (entity is User) {
  convKey = entity.uid;
} else if (entity is Group) {
  convKey = entity.guid;
}
final typingName = _typingMap[convKey]; // null if nobody is typing
```

The SDK does not send `onTypingEnded` if the user disconnects or goes idle without explicitly stopping. Add a safety-net timer (5-8 seconds) that auto-clears stale typing entries from the map. This prevents "ghost" typing indicators if `onTypingEnded` is never received.

```dart
final Map<String, String> _typingMap = {};
final Map<String, Timer> _typingTimers = {};

@override
void onTypingStarted(TypingIndicator indicator) {
  final key = _typingKey(indicator);

  // Cancel any existing safety-net timer for this key
  _typingTimers[key]?.cancel();

  setState(() => _typingMap[key] = indicator.sender.name);

  // Safety-net: auto-clear after 6s if onTypingEnded never arrives
  _typingTimers[key] = Timer(const Duration(seconds: 6), () {
    if (mounted) setState(() => _typingMap.remove(key));
    _typingTimers.remove(key);
  });
}

@override
void onTypingEnded(TypingIndicator indicator) {
  final key = _typingKey(indicator);
  _typingTimers[key]?.cancel();
  _typingTimers.remove(key);
  setState(() => _typingMap.remove(key));
}

// Cancel all timers in dispose:
@override
void dispose() {
  for (final timer in _typingTimers.values) { timer.cancel(); }
  super.dispose();
}
```

## Delivery & Read Receipts

Mark messages as delivered/read, receive receipt events via MessageListener.

```dart
// Mark as read — pass the BaseMessage object
CometChat.markAsRead(message,
  onSuccess: (String unused) { debugPrint("Marked as read"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);

// Mark as delivered
CometChat.markAsDelivered(message,
  onSuccess: (String unused) { debugPrint("Marked as delivered"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);
```

Receive receipt events:

```dart
@override
void onMessagesDelivered(MessageReceipt messageReceipt) {
  // messageReceipt.messageId, .sender, .receiverId, .deliveredAt
}

@override
void onMessagesRead(MessageReceipt messageReceipt) {
  // messageReceipt.messageId, .sender, .receiverId, .readAt
}

// Group-only (requires Enhanced Messaging Status feature):
@override
void onMessagesDeliveredToAll(MessageReceipt messageReceipt) { }

@override
void onMessagesReadByAll(MessageReceipt messageReceipt) { }
```

When to call markAsRead:
1. When message list is fetched (mark the last message)
2. When a real-time message arrives while the chat window is open

## Connection Status

Monitor WebSocket connection state:

```dart
class _AppState extends State<App> with ConnectionListener {
  static const _listenerId = "app_connection_listener";

  @override
  void initState() {
    super.initState();
    CometChat.addConnectionListener(_listenerId, this);
  }

  @override
  void dispose() {
    CometChat.removeConnectionListener(_listenerId);
    super.dispose();
  }

  @override
  void onConnected() { debugPrint("WebSocket connected"); }

  @override
  void onConnecting() { debugPrint("WebSocket connecting..."); }

  @override
  void onDisconnected() { debugPrint("WebSocket disconnected"); }

  @override
  void onFeatureThrottled() { debugPrint("Features throttled"); }

  @override
  void onConnectionError(CometChatException error) {
    debugPrint("Connection error: ${error.message}");
  }
}

// Get current status synchronously
String status = CometChat.getConnectionStatus();
// Returns: CometChatWSState.connected / .connecting / .disconnected / .featureThrottled
```

The SDK auto-reconnects on disconnect. In auto mode: connected in foreground, disconnected in background.

### Web-Specific Behavior

The SDK adapts WebSocket handling for web browsers:

- **Lifecycle:** `AppLifecycleState.inactive` is ignored on web (tab focus changes briefly trigger it, causing false disconnects on native). Only `paused`/`detached`/`hidden` trigger background disconnect on web.
- **Pong timeout:** 10 seconds on web (vs 3s on native). Browser timer throttling in background tabs can delay pong responses — the longer timeout prevents false reconnections.
- **Tab resume:** When a browser tab returns to foreground, the SDK resets any queued reconnection attempts before reconnecting. This prevents a storm of simultaneous reconnect attempts from timers that were throttled while the tab was in the background.

## Anti-Patterns

**No subscriptionType → no presence events:**
```dart
// ❌ WRONG — presence silently disabled, no error
AppSettings appSettings = (AppSettingsBuilder()..region = "us").build();
// UserListener.onUserOnline/onUserOffline will NEVER fire

// ✅ CORRECT
AppSettings appSettings = (AppSettingsBuilder()
  ..subscriptionType = CometChatSubscriptionType.allUsers
  ..region = "us"
).build();
```

**Listener leak — not removing in dispose():**
```dart
// ❌ WRONG — events fire on disposed widget, causes setState errors
@override void initState() { CometChat.addUserListener("id", this); }
// Missing dispose cleanup
```

The SDK logs a warning when any listener map exceeds 10 entries — this strongly suggests a leak from missing `removeListener()` calls in `dispose()`.

**Duplicate listener IDs across screens:**
```dart
// ❌ WRONG — second replaces first silently
CometChat.addMessageListener("listener", screenA);
CometChat.addMessageListener("listener", screenB);  // screenA stops receiving
```

**Calling markAsRead without checking featureThrottled:**
The SDK validates connection state internally. If featureThrottled, markAsRead throws ERROR_RECEIPTS_TEMPORARILY_BLOCKED.

**Re-fetching full conversation list on every new message:**
```dart
// ❌ WRONG — nukes list, causes full screen rebuild on every message
void onNewMessage() {
  conversations = [];
  fetchAllConversations(); // O(n) network call, UI flashes
}

// ✅ CORRECT — use getConversation() to patch single item in-place
void onNewMessage(BaseMessage message) async {
  final (partner, type) = extractPartner(message, loggedInUid);
  final updated = await getConversation(partner, type);
  conversations.removeWhere((c) => c.conversationId == updated.conversationId);
  conversations.insert(0, updated); // move to top with updated lastMessage/unreadCount
}
```
See `cometchat-flutter-compositions` skill for the full partner extraction and patching pattern.

## Checklist

- subscriptionType set during init (required for presence events)
- All listeners registered in initState, removed in dispose
- Unique listener IDs per screen/widget instance
- Typing indicators: startTyping on text input change, endTyping on send or pause
- Conversation list typing: use keyed map (not boolean) to track per-conversation typing state, with safety-net timer (5-8s) to auto-clear stale entries
- markAsRead called when chat window opens and on new real-time messages
- ConnectionListener registered at app level to handle reconnection UI
- onError callbacks handled (not empty) on markAsRead/markAsDelivered
