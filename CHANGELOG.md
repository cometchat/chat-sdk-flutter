## 4.0.13
**New**
- Added a new parameter `includeBlockedUsers` in `ConversationsRequestBuilder` to allow fetching conversations of users blocked by the logged-in user.
- Added a new parameter `withBlockedInfo` in `ConversationsRequestBuilder` to fetch blocked information in the conversation object, such as blockedByMe and hasBlockedMe.
- Added a new parameter for `scopes` in `BannedMembersRequestBuilder`.

**Fixes**
- Fixed an issue causing a critical crash on Android devices when invoking the `CometChat.disconnect()` method.

## 4.0.12
**New**
- Added a method named getConversationUpdateSettings which returns the settings saved in Dashboard.

## 4.0.11
**Enhancements**
- Updated all 3rd-party plugins versions.
- Resolved all static Dart Analyser suggestions
- Added namespaces in build.gradle to avoid conflicts.

## 4.0.10
**Fixes**
- Fixed the issue where the onInteractiveMessageReceived listener is not working in iOS.

## 4.0.9
**New**
- Added PrivacyInfo file that complies with the latest apple guidelines for SDKs.

## 4.0.8
**New**
- Added a new boolean property **isBannedFromGroup** to the Group model class to indicate whether the user is banned from the group or not.
- Introduced three new properties in **CustomMessage** class as follows:
  - sendNotification: True value will trigger a notification for the custom message.
  - conversationText: This string can be used to display the last message text in the conversation list.
- updateConversation: If set to true, the message will appear as the last message in the Conversation and will update and reorder the conversation list, placing the conversation at the top.

**Fixes**
- Fixed URL handling in `CometChat.callExtension`, ensuring consistent and reliable endpoint interactions

## 4.0.7
**New**
- Introduced `Reaction` and `ReactionEvent` classes to enhance the functionality of message reactions introduced in v4.0.3.
- Added `ReactionsRequest` class as a replacement for `ReactionRequest` class introduced in v4.0.3.
- Added `ReactionsRequestBuilder` class as a replacement for `ReactionRequestBuilder` class introduced in v4.0.3.
- Introduced Enhanced Push Notification Feature.
    - Added `PushPreferences`, GroupPreferences`, `MutePreferences`, `MutedConversation`, `UnmutedConversation` & `DaySchedule` classes
    - Added method `registerPushToken` method to register push token.
    - Added method `unregisterPushToken` method to unregister push token.
    - Added method `muteConversations` method to mute push notifications of conversations.
    - Added method `unmuteConversations` method to unmute push notifications of conversations.
    - Added method `fetchPushPreferences`, `updatePushPreferences` & `resetPushPreferences` to fetch, update & reset push preferences.

**Enhancements**
- The real-time listeners `onMessageReactionAdded` and `onMessageReactionRemoved` have been improved to return an object of `ReactionEvent` class, providing a more robust event model.
- `ReactionsRequestBuilder` has been enhanced to return a list of `Reaction` objects, offering a more intuitive and consistent API.

**Removals**
- Removed the `MessageReaction` class introduced in v4.0.3, transitioning its responsibilities to the new `Reaction` and `ReactionEvent` classes.
- Removed `myMentionsOnly` method from `MessagesRequestBuilder` introduced in v4.0.2.

## 4.0.6
* Bug: fixed sending metadata property in Call
* Bug: fixed sending muid property in Call

## 4.0.5
* Bug: fixed sending media messages with captions and tags in iOS
* Bug: fixed sending media messages with Attachment URL in iOS

## 4.0.4
* New: Added support to react on Text, Media and Custom Messages
* New: Added a new field unreadMentionsCount field in Conversation Object
* New: Added a new field lastReadMessageId field in Conversation Object

## 4.0.3
* New: Added support to mention a user in a Text & Media Message
* New: Added support to send interactive messages like forms, card, etc.
* New: Added a method to mark a message as unread

## 4.0.2
* New: Added methods for AI Conversation Summary & AI Assist Bot

## 4.0.1
* Bug: Fixes and performance improvements

## 4.0.0
* New: Optimized websocket connection
* New: Ping() for websocket connection
* Bug: Fixes and performance improvements