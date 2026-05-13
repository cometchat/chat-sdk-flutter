---
name: cometchat-flutter-messaging
description: Use when sending or receiving messages with CometChat Flutter SDK v5. Triggers on mentions of TextMessage, MediaMessage, CustomMessage, sendMessage, sendMediaMessage, sendCustomMessage, MessageListener, MessagesRequest, MessagesRequestBuilder, fetchPrevious, fetchNext, editMessage, deleteMessage, or message pagination. Also use when the user asks about chat message delivery, editing or deleting messages, thread replies, or loading message history. Make sure to use this skill whenever the user mentions sending or receiving chat messages, even if they just say 'add chat functionality', 'display conversation', or 'load message history'.
---

# CometChat Flutter SDK v5 Messaging

Four message types: TextMessage, MediaMessage, CustomMessage, InteractiveMessage. Plus AI message types: AIAssistantMessage, AIToolResultMessage, AIToolArgumentMessage. All sent via dedicated methods on the CometChat class. Real-time receiving via MessageListener. History via MessagesRequestBuilder.

## Sending Messages

### Text Message

```dart
TextMessage textMessage = TextMessage(
  text: "Hello!",
  receiverUid: "cometchat-uid-1",
  receiverType: CometChatConversationType.user,  // or .group
  type: CometChatMessageType.text,
);

CometChat.sendMessage(textMessage,
  onSuccess: (TextMessage message) {
    debugPrint("Sent: ${message.id}");
  },
  onError: (CometChatException e) {
    debugPrint("Failed: ${e.code} - ${e.message}");
  },
);
```

For groups, use `CometChatConversationType.group` and pass the GUID as `receiverUid`.

Optional fields: `metadata` (Map), `tags` (List<String>), `quotedMessageId` (int), `quotedMessage` (BaseMessage).

### Media Message (file upload)

```dart
MediaMessage mediaMessage = MediaMessage(
  receiverType: CometChatConversationType.user,
  type: CometChatMessageType.image,  // .video, .audio, .file
  receiverUid: "cometchat-uid-1",
  file: "/path/to/image.jpg",
);
mediaMessage.caption = "Check this out";

CometChat.sendMediaMessage(mediaMessage,
  onSuccess: (MediaMessage message) {
    debugPrint("Sent media: ${message.attachment?.fileUrl}");
  },
  onError: (CometChatException e) {
    debugPrint("Failed: ${e.message}");
  },
);
```

Two ways to send media: file path (SDK uploads) or URL via Attachment object.
Multiple files: use `files: [path1, path2]` instead of `file:`.
SDK validates file size and count (configurable in Dashboard). Errors: ERR_FILE_SIZE_EXCEEDED, ERR_FILE_COUNT_EXCEEDED.

### Custom Message

```dart
CustomMessage customMessage = CustomMessage(
  receiverUid: "cometchat-uid-1",
  type: CometChatMessageType.custom,
  customData: {"latitude": "19.0760", "longitude": "72.8777"},
  receiverType: CometChatConversationType.user,
  subType: "LOCATION",
);
customMessage.updateConversation = false;  // optional: don't update last message
customMessage.conversationText = "Shared a location";  // custom notification body

CometChat.sendCustomMessage(customMessage,
  onSuccess: (CustomMessage message) { debugPrint("Sent custom: $message"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);
```

## Receiving Messages (Real-time)

Register a MessageListener to receive messages while the app is running. Remove it in dispose().

```dart
class ChatScreen extends StatefulWidget { ... }

class _ChatScreenState extends State<ChatScreen> with MessageListener {
  static const _listenerId = "chat_screen_listener";

  @override
  void initState() {
    super.initState();
    CometChat.addMessageListener(_listenerId, this);
  }

  @override
  void dispose() {
    CometChat.removeMessageListener(_listenerId);
    super.dispose();
  }

  @override
  void onTextMessageReceived(TextMessage textMessage) {
    setState(() { _messages.add(textMessage); });
  }

  @override
  void onMediaMessageReceived(MediaMessage mediaMessage) {
    setState(() { _messages.add(mediaMessage); });
  }

  @override
  void onCustomMessageReceived(CustomMessage customMessage) {
    setState(() { _messages.add(customMessage); });
  }

  @override
  void onMessageEdited(BaseMessage message) {
    // Update the message in your list
  }

  @override
  void onMessageDeleted(BaseMessage message) {
    // Remove or mark as deleted in your list
  }
}
```

Key: As a sender, you will NOT receive your own message via the listener. Multi-device users will receive their own messages on other devices.

Full MessageListener callbacks: onTextMessageReceived, onMediaMessageReceived, onCustomMessageReceived, onInteractiveMessageReceived, onTypingStarted, onTypingEnded, onMessagesDelivered, onMessagesRead, onMessagesDeliveredToAll, onMessagesReadByAll, onMessageEdited, onMessageDeleted, onTransientMessageReceived, onInteractionGoalCompleted, onMessageReactionAdded, onMessageReactionRemoved, onMessageModerated, onAIAssistantMessageReceived, onAIToolResultReceived, onAIToolArgumentsReceived.

## Fetching Message History

Use MessagesRequestBuilder for paginated message fetching. `fetchPrevious()` loads older messages, `fetchNext()` loads newer messages.

### Load conversation history (older messages)

```dart
MessagesRequest messageRequest = (MessagesRequestBuilder()
  ..uid = "cometchat-uid-1"  // or ..guid = "group-guid"
  ..limit = 30
).build();

messageRequest.fetchPrevious(
  onSuccess: (List<BaseMessage> messages) {
    for (var msg in messages) {
      if (msg is TextMessage) debugPrint("Text: ${msg.text}");
      else if (msg is MediaMessage) debugPrint("Media: ${msg.attachment?.fileUrl}");
    }
  },
  onError: (CometChatException e) {
    debugPrint("Fetch failed: ${e.message}");
  },
);
```

Call `fetchPrevious()` repeatedly on the same object for pagination (loads progressively older messages).

### Pagination Metadata

`MessagesResult` includes optional page-based pagination fields alongside cursor-based pagination:

```dart
// MessagesResult fields:
// - messages: List<BaseMessage>
// - hasMore: bool
// - currentPage: int?   (from API response meta.pagination.current_page)
// - totalPages: int?    (from API response meta.pagination.total_pages)
```

The SDK automatically extracts `currentPage` and `totalPages` from the API response's `meta.pagination` object (matching Android SDK behavior). These are used internally by `MessagesRequest` to determine when pagination is exhausted (`currentPage == totalPages`). For most consumer use cases, just call `fetchPrevious()`/`fetchNext()` repeatedly — the SDK handles pagination state internally.

### Load missed messages (newer messages since last seen)

```dart
MessagesRequest messageRequest = (MessagesRequestBuilder()
  ..uid = "cometchat-uid-1"
  ..limit = 30
  ..messageId = lastReceivedMessageId  // cursor: fetch after this ID
).build();

messageRequest.fetchNext(
  onSuccess: (List<BaseMessage> messages) { ... },
  onError: (CometChatException e) { ... },
);
```

fetchNext requires at least one cursor: messageId, timestamp, or updatedAfter. Without it: ERR_FILTERS_MISSING.

### Builder filters

| Filter | Type | Purpose |
|--------|------|---------|
| uid | String | Filter by user conversation |
| guid | String | Filter by group conversation |
| limit | int | Max messages per page (default 30, max 100) |
| messageId | int | Cursor for pagination |
| searchKeyword | String | Search in message text |
| categories | List<String> | Filter by category (message, action, call, custom) |
| types | List<String> | Filter by type (text, image, video, audio, file, custom) |
| parentMessageId | int | Fetch thread replies |
| hideReplies | bool | Exclude thread replies from main list |
| hideDeleted | bool | Exclude deleted messages |
| unread | bool | Only unread messages |
| withTags | bool | Include tag info |
| tags | List<String> | Filter by tags |

## Handling Action Messages in Message Lists

`fetchPrevious()`/`fetchNext()` return ALL message categories, including `Action` messages (category: `action`). These represent system events — member joined, member kicked, message deleted, message edited, scope changed, etc.

### Distinguishing Action types

The `Action` class has an `actionOn` field that tells you what the action targets:

- `actionOn is BaseMessage` → message-level action (delete or edit). The original message in your list already reflects the state via `deletedAt`/`editedAt`, so the Action is redundant for display.
- `actionOn is User` or `actionOn is GroupMember` → group-level action (member joined/left/kicked/banned, scope changed). These are typically rendered as system banners.

```dart
for (var msg in messages) {
  if (msg is Action) {
    // Message-level actions (delete/edit) — skip, the bubble handles it
    if (msg.actionOn is BaseMessage) continue;

    // Group-level actions — render as system banner
    displaySystemBanner(msg.message ?? msg.action ?? 'Action');
    continue;
  }

  // Regular messages — render as bubbles
  if (msg is TextMessage) { ... }
  else if (msg is MediaMessage) { ... }
}
```

### Why skip message-level Action messages?

When a message is deleted, two things happen:
1. The original message's `deletedAt` is set — the bubble shows "This message was deleted"
2. An `Action` message is added to history — showing a redundant "message deleted" banner

Displaying both creates duplicate UI for the same event. The bubble's deleted state is the canonical display.

### Alternative: filter at the request level

Use the `categories` filter on `MessagesRequestBuilder` to exclude action messages entirely:

```dart
MessagesRequest request = (MessagesRequestBuilder()
  ..uid = "cometchat-uid-1"
  ..limit = 30
  ..categories = [CometChatMessageCategory.message]  // excludes action, call
).build();
```

This is simpler but also hides group-level actions (member joined/left). Use it only if you don't need system banners at all.

## Edit & Delete Messages

### Edit

```dart
TextMessage updatedMessage = TextMessage(
  text: "Updated text",
  receiverUid: receiverID,
  receiverType: receiverType,
  type: CometChatMessageType.text,
);
updatedMessage.id = originalMessage.id;  // MUST set the original message ID

CometChat.editMessage(updatedMessage,
  onSuccess: (BaseMessage msg) { debugPrint("Edited: ${msg.editedAt}"); },
  onError: (CometChatException e) { debugPrint("Edit failed: ${e.message}"); },
);
```

Only TextMessage and CustomMessage can be edited. The editedAt and editedBy fields are set on the returned message.

### Delete

```dart
CometChat.deleteMessage(messageId,
  onSuccess: (BaseMessage msg) { debugPrint("Deleted at: ${msg.deletedAt}"); },
  onError: (CometChatException e) { debugPrint("Delete failed: ${e.message}"); },
);
```

## Anti-Patterns

**Forgetting to remove listener in dispose():**
```dart
// ❌ WRONG — listener leak, receives events after widget is disposed
@override
void initState() {
  CometChat.addMessageListener("id", this);
}
// Missing: CometChat.removeMessageListener("id") in dispose()
```

**Duplicate listener IDs:**
```dart
// ❌ WRONG — second listener replaces the first (same ID)
CometChat.addMessageListener("chat", screenA);
CometChat.addMessageListener("chat", screenB);  // screenA stops receiving
// ✅ Use unique IDs per screen/widget
```

**Not handling BaseMessage subtypes:**
```dart
// ❌ WRONG — treats all messages as text
for (var msg in messages) {
  print(msg.text);  // BaseMessage has no .text property
}
// ✅ Use type checking
for (var msg in messages) {
  if (msg is TextMessage) print(msg.text);
  else if (msg is MediaMessage) print(msg.attachment?.fileUrl);
}
```

**Calling fetchNext without a cursor:**
```dart
// ❌ WRONG — ERR_FILTERS_MISSING
MessagesRequest req = (MessagesRequestBuilder()..uid = "uid"..limit = 30).build();
req.fetchNext(...);  // No messageId, timestamp, or updatedAfter set
// ✅ fetchPrevious works without cursor (loads from latest). fetchNext needs one.
```

## Checklist

- MessageListener registered in initState, removed in dispose
- Unique listener IDs per screen/widget
- BaseMessage subtypes checked with `is` operator when processing message lists
- Action messages filtered: `actionOn is BaseMessage` skipped (bubble handles deleted/edited state), group-level actions rendered as banners
- fetchPrevious for history, fetchNext for missed messages (with cursor)
- onError callbacks handle and surface errors
- Media messages use correct CometChatMessageType (.image, .video, .audio, .file)
- Edit sets original message ID before calling editMessage
