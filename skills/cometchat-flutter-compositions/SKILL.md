---
name: cometchat-flutter-compositions
description: Use when combining multiple CometChat SDK methods for app flows like registration, conversation list updates, AI feature gating, or bootstrap sequences. Triggers on mentions of createUser then login, getConversation for patching, isAIFeatureEnabled, setSource, setPlatformParams, conversation list refresh on new message, or when building multi-step SDK integration flows.
---

# CometChat Flutter SDK v5 — Method Compositions

How to chain and combine SDK methods for common app flows. Each composition shows the method sequence, error handling between steps, and the data flow.

## Registration Flow: createUser → login

Registration requires two SDK calls in sequence. `createUser` creates the CometChat user, then `login` authenticates them.

```dart
Future<void> register(String uid, String name, String authKey) async {
  // Step 1: Create user in CometChat
  await CometChat.createUser(User(uid: uid, name: name), authKey,
    onSuccess: (user) => debugPrint("Created: ${user.uid}"),
    onError: (e) => throw e,
  );

  // Step 2: Login the newly created user
  await CometChat.login(uid, authKey,
    onSuccess: (user) => debugPrint("Logged in: ${user.uid}"),
    onError: (e) => throw e,
  );
}
```

Error handling between steps:
- If `createUser` succeeds but `login` fails → the user already exists in CometChat. Retry `login` only — calling `createUser` again will fail with user-already-exists.
- If `createUser` fails → check the error code. `ERR_UID_ALREADY_EXISTS` means the user was created previously (e.g., on a prior attempt). In that case, skip straight to `login`.

```dart
// Robust registration with intermediate failure handling
Future<User> registerRobust(String uid, String name, String authKey) async {
  try {
    await CometChat.createUser(User(uid: uid, name: name), authKey,
      onSuccess: (_) {},
      onError: (e) => throw e,
    );
  } catch (e) {
    // If user already exists, that's fine — proceed to login
    if (e is CometChatException && e.code == 'ERR_UID_ALREADY_EXISTS') {
      debugPrint("User already exists, proceeding to login");
    } else {
      rethrow;
    }
  }

  final c = Completer<User>();
  CometChat.login(uid, authKey,
    onSuccess: (user) => c.complete(user),
    onError: (e) => c.completeError(e),
  );
  return c.future;
}
```

## Bootstrap Sequence: init → configure → session check

The full SDK startup chain includes configuration calls after init that are often missed.

```dart
Future<User?> bootstrapSDK(String appId, AppSettings settings) async {
  // 1. Initialize
  await CometChat.init(appId, settings, onSuccess: (_) {}, onError: (e) => throw e);

  // 2. Configure (often missed — needed for analytics and platform tracking)
  CometChat.setSource("your_app", "flutter", "dart");
  CometChat.setPlatformParams(platform: "flutter", sdkVersion: "1.0.0");
  CometChat.setDemoMetaInfo(jsonObject: {"app": "your_app"});

  // 3. Check existing session
  final completer = Completer<User?>();
  CometChat.getLoggedInUser(
    onSuccess: (user) => completer.complete(user),
    onError: (_) => completer.complete(null),
  );
  return completer.future;
}
```

## Conversation Patching: MessageListener → getConversation → patch list

When a new message arrives, fetch only the affected conversation instead of re-fetching the entire list.

### Step 1: Extract conversation partner from BaseMessage

```dart
/// Determine which conversation a message belongs to.
/// For groups: use receiverUid (the GUID).
/// For users: compare sender against logged-in user to determine direction.
(String conversationWith, String conversationType) extractPartner(
  BaseMessage message,
  String loggedInUid,
) {
  if (message.receiverType == 'group') {
    return (message.receiverUid, 'group');
  }
  // For 1:1 — if we sent it, conversation is with receiver.
  // If we received it, conversation is with sender.
  final partner = message.sender?.uid == loggedInUid
      ? message.receiverUid
      : message.sender?.uid ?? message.receiverUid;
  return (partner, 'user');
}
```

### Step 2: Fetch and patch

```dart
Future<void> patchConversation(
  BaseMessage message,
  String loggedInUid,
  List<Conversation> conversations,
) async {
  final (conversationWith, conversationType) = extractPartner(message, loggedInUid);

  final completer = Completer<Conversation>();
  CometChat.getConversation(conversationWith, conversationType,
    onSuccess: (conv) => completer.complete(conv),
    onError: (e) => completer.completeError(e),
  );
  final updated = await completer.future;

  // Remove old entry, insert updated at top
  conversations.removeWhere((c) => c.conversationId == updated.conversationId);
  conversations.insert(0, updated);
}
```

### Step 3: Debounce rapid messages

When multiple messages arrive in quick succession (e.g., a burst of messages in a group), debounce the patch to avoid redundant `getConversation` calls:

```dart
Timer? _patchDebounce;

void onNewMessage(BaseMessage message) {
  _patchDebounce?.cancel();
  _patchDebounce = Timer(const Duration(milliseconds: 500), () {
    patchConversation(message, loggedInUid, conversations);
  });
}

// Cancel in dispose:
@override
void dispose() {
  _patchDebounce?.cancel();
  super.dispose();
}
```

This coalesces rapid message bursts into a single `getConversation` call for the latest message.

### Anti-pattern: full list refresh on every message

```dart
// ❌ WRONG — nukes list, causes full rebuild, O(n) network call
void onNewMessage() {
  conversations = [];
  fetchAllConversations();
}

// ✅ CORRECT — single conversation fetch, O(1) patch
void onNewMessage(BaseMessage msg) => patchConversation(msg, myUid, conversations);
```

## Typing State on Conversation List: MessageListener → map TypingIndicator to conversation

Similar to conversation patching (above), typing indicators on a conversation list require mapping a real-time event to a specific conversation row. But unlike message patching, typing is transient state — no SDK fetch needed, just map the `TypingIndicator` fields to the right conversation.

### Mapping TypingIndicator to a conversation

The key insight: `TypingIndicator.sender.uid` identifies who is typing, and `TypingIndicator.receiverId` + `receiverType` identifies where. For 1:1 chats, the conversation key is the sender's UID. For groups, it's the receiverId (the GUID).

```dart
/// Build a conversation lookup key from a TypingIndicator.
/// This key matches against User.uid or Group.guid in the conversation list.
String typingConversationKey(TypingIndicator indicator) {
  return indicator.receiverType == 'group'
      ? indicator.receiverId   // GUID — the group conversation
      : indicator.sender.uid;  // sender's UID — the 1:1 conversation
}
```

### Composition with MessageListener

Add `onTypingStarted`/`onTypingEnded` to the same MessageListener already used for conversation patching. Both patterns share the same listener — no extra registration needed.

```dart
// Same MessageListener handles both message patching AND typing state
@override
void onTypingStarted(TypingIndicator indicator) {
  final key = typingConversationKey(indicator);
  setState(() => _typingMap[key] = indicator.sender.name);
}

@override
void onTypingEnded(TypingIndicator indicator) {
  final key = typingConversationKey(indicator);
  setState(() => _typingMap.remove(key));
}

// When rendering each conversation row:
// final typingName = _typingMap[conversationPartnerUidOrGuid];
// if (typingName != null) → show typing indicator instead of last message preview
```

### Anti-pattern: separate listener for typing on conversation list

```dart
// ❌ WRONG — registering a second MessageListener just for typing
// This either uses a duplicate ID (silently replaces the message listener)
// or adds unnecessary listener overhead
CometChat.addMessageListener("chats_typing", typingOnlyListener);
CometChat.addMessageListener("chats_messages", messageOnlyListener);

// ✅ CORRECT — single listener handles both messages and typing
CometChat.addMessageListener("chats_screen", combinedListener);
```

SUGGESTION: Consider adding a safety-net timer (5-8 seconds) that auto-clears stale entries from the typing map. The SDK does not guarantee `onTypingEnded` fires if the sender disconnects or goes idle without explicitly calling `endTyping`.

## AI Feature Gating: isAIFeatureEnabled → conditional AI calls

Always check if an AI feature is enabled before calling AI methods. Calling without checking produces silent failures.

```dart
Future<List<String>> getSmartRepliesIfEnabled(
  String receiverId,
  String receiverType,
) async {
  // Gate: check feature availability first
  final completer = Completer<bool>();
  CometChat.isAIFeatureEnabled('smart-replies',
    onSuccess: (enabled) => completer.complete(enabled),
    onError: (_) => completer.complete(false),
  );
  final enabled = await completer.future;
  if (!enabled) return [];

  // Feature is available — fetch replies
  final repliesCompleter = Completer<List<String>>();
  CometChat.getConversationStarter(receiverId, receiverType,
    onSuccess: (starters) => repliesCompleter.complete(starters),
    onError: (_) => repliesCompleter.complete([]),
  );
  return repliesCompleter.future;
}
```

### Context-aware AI: starters vs replies

Use different AI methods depending on conversation state:
- Empty conversation (no messages yet) → `getConversationStarter` — suggests ice-breaker prompts
- Active conversation (messages exist) → `getSmartReplies` — suggests contextual replies based on recent messages

```dart
Future<void> loadAISuggestions(String receiverId, String receiverType, bool hasMessages) async {
  final enabled = await isAIFeatureEnabled('smart-replies');
  if (!enabled) return;

  if (!hasMessages) {
    // New conversation — show conversation starters
    final starters = await getConversationStarter(receiverId, receiverType);
    setState(() => _smartReplies = starters);
  } else {
    // Active conversation — show contextual smart replies
    final replies = await getSmartReplies(receiverId, receiverType);
    setState(() => _smartReplies = replies.values.toList());
  }
}
```

## Conversation Open: markAsRead → navigate → refresh on return

When opening a conversation, mark it as read before entering the chat. On return, refresh the conversation list to reflect updated state.

```dart
Future<void> openConversation(String uid, String type) async {
  // 1. Mark as read immediately
  CometChat.markConversationAsRead(uid, type,
    onSuccess: (_) {},
    onError: (_) {},
  );

  // 2. Navigate to chat screen
  await Navigator.push(context, MaterialPageRoute(builder: (_) => ChatScreen(...)));

  // 3. On return — refresh conversation list to pick up any changes
  refreshConversations();
}
```

## Parallel SDK Fetching with Future.wait

When you need multiple independent SDK results (e.g., loading a screen that shows both joined groups and all groups), fetch them in parallel:

```dart
// ❌ WRONG — sequential, doubles the wait time
final myGroups = await fetchGroups(joinedOnlyRequest);
final allGroups = await fetchGroups(discoverRequest);

// ✅ CORRECT — parallel, total time = max(request1, request2)
final results = await Future.wait([
  fetchGroups(joinedOnlyRequest),
  fetchGroups(discoverRequest),
]);
final myGroups = results[0];
final allGroups = results[1];
```

This works for any independent SDK calls: group + members, user list + online count, unread counts for users + groups, etc. Only use `Future.wait` when the calls are truly independent — if call B depends on the result of call A, keep them sequential.

## UI Suggestions (Non-Binding)

These patterns worked well in practice. Use them if they fit your app's design.

- SUGGESTION: Consider separating SDK init from auth UI into a dedicated bootstrap step — keeps auth screens testable without SDK dependencies
- SUGGESTION: Consider showing ConnectionListener status as a subtle indicator rather than blocking UI

## Checklist

- Registration chains createUser → login (not login alone)
- Bootstrap includes setSource/setPlatformParams/setDemoMetaInfo after init
- Conversation updates use getConversation for single-item patch, not full re-fetch
- Conversation list typing uses keyed map from TypingIndicator, shares same MessageListener as message patching
- AI methods gated behind isAIFeatureEnabled check
- markConversationAsRead called when opening a conversation
- Partner extraction handles both sender and receiver direction for 1:1 chats
- Conversation patching debounced (300-500ms) to coalesce rapid message bursts
- Parallel SDK fetching used for independent data loads (Future.wait)
- AI suggestions context-aware: starters for empty conversations, smart replies for active ones
