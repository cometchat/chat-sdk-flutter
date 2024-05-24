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