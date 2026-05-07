## 5.0.0

**New**
- Added a fully rewritten Flutter SDK built entirely in pure Dart, with unified support across iOS, Android, Web, and desktop platforms.
- Added modular package imports, allowing developers to include only the required modules such as `messaging.dart`, `calling.dart`, `ai.dart`, `notifications.dart`, and `moderation.dart`.
- Added real-time typing indicators powered by native Dart streams.
- Added background isolate parsing for improved performance when handling large message lists.
- Added automatic HTTP retry support with exponential backoff for transient network failures.
- Added improved WebSocket reconnection handling on web platforms during browser tab throttling.
- Added pagination metadata, including `currentPage` and `totalPages`, to message fetch responses.
- Added device-level event routing to prevent duplicate event handling across multiple sessions.
- Added feature flags to disable typing indicators, read receipts, and presence updates.

**Enhancements**
- None

**Fixes**
- None

**Breaking Changes**
- Changed `onTypingIndicator()` to return `Stream<TypingIndicator>` instead of `Stream<String>`.
- Removed deprecated APIs including `receaverUid` and `fetchPushPreferences`.
- Renamed builder setters to `setWithUserAndGroupTags`, `setWithTags`, `setIncludeBlockedUsers`, and `setWithBlockedInfo`.

## 5.0.0-beta.3

**New**
- Added modular exports (`core.dart`, `messaging.dart`, etc.) so you can import only what you need and reduce app size.
- Introduced automatic HTTP retries with backoff for temporary failures (`408`, `429`, `5xx`), improving reliability.
- Added feature flags in `AppSettings` to disable typing indicators, read receipts, and presence.
- Added listener leak detection to help identify missing `removeListener()` calls.
- Simplified `ConversationsRequestBuilder` with clearer method names and public flags for better control.

**Enhancements**
- Disabled request/response logging in release builds to improve performance.
- Improved JSON parsing performance by offloading smaller payloads to background isolates.
- Reduced default HTTP timeout to 15 seconds for faster failure handling.

**Fixes**
- Fixed dropped call events caused by incorrect event mapping.
- Fixed call event parsing and routing for better consistency.
- Resolved a compile issue in `_dispatchCallEvent`.
- Fixed pagination on web by correctly handling `currentPage` and `totalPages`.
- Improved WebSocket stability on web, reducing unnecessary reconnects.

## 5.0.0-beta.2

**New**
- Added device and routing metadata (`deviceId`, `sender`, `receiver`, `receiverType`) to socket events, enabling accurate message routing and preventing duplicate event handling.
- Introduced a native Dart stream for typing indicators via `RealtimeRepository.typingStream`, improving performance and reliability. The `onTypingIndicator()` method now returns `Stream<TypingIndicator>`.
- Expanded the `ReactionData` model to include additional metadata (`id`, `messageId`, `uid`) for more complete reaction tracking.
- Standardized `ReactionActionEvent` action constants to align with the Android SDK (`message_reaction_added`, `message_reaction_removed`).
- Updated message event processing to use a DTO-to-mapper pipeline, ensuring consistent parsing with API responses.
- Added a platform-compatible isolate abstraction (`runInIsolate()`), improving support for web environments.
- Improved action message text formatting to ensure consistency across platforms (e.g., "Message Edited").

**Enhancements**
- Simplified package configuration by removing platform-specific plugin settings (Android/iOS) from `pubspec.yaml`.

**Fixes**
- Resolved an issue where `deviceId` was not parsed in most socket events, which prevented proper echo filtering for typing indicators, receipts, presence, and reactions.
- Fixed an issue where users could see their own typing indicators in group chats.
- Corrected `rawData` handling in `MessageMapper` to store the complete JSON payload instead of only the action string.

## 5.0.0-beta1

**New**
- All platform channel method calls have been replaced with native Dart implementations, resulting in significant speed and performance improvements.
- The SDK now runs entirely on Dart by default, bringing cross-platform support to iOS, Android, and Web.

**Breaking Changes**
- `onTypingIndicator()` now returns `Stream<TypingIndicator>` instead of `Stream<String>`. 

Typing events include `sender`, `receiverId`, `receiverType`, `metadata`, `lastTimestamp`, and `typingStatus` fields. 

Use `TypingIndicator.typingStatus` ("started" or "ended") instead of checking `methodName`. No `EventChannel` dependency — works on all platforms including web. Existing `MessageListener` callbacks (`onTypingStarted` / `onTypingEnded`) continue to work unchanged.

**Removals**
- Removed deprecated `markAsUnread()` method. Use `markMessageAsUnread()` instead.
- Removed deprecated `receaverUid` parameter from `startTyping()` and `endTyping()`. Use `receiverUid` instead.
- Removed deprecated `fetchPushPreferences()`, `updatePushPreferences()`, and `resetPushPreferences()` methods. Use `fetchPreferences()`, `updatePreferences()`, and `resetPreferences()` instead.
- Removed deprecated `PushPreferences` class. Use `NotificationPreferences` instead.

