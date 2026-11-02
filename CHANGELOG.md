## 5.1.0

**New**
- Added offline messaging, an opt-in mode that keeps chats usable without a connection. Enable it with `OfflineConfig(enabled: true, …)` in `AppSettings`, through `init` or `initFromSettings` (which reads `chatSDK.offline`). Offline stays off unless you enable it; apart from the changes under **Breaking Changes**, an app that does not enable it behaves as on 5.0.9. On web, run `dart run cometchat_sdk:setup_web` once from your app's root; the README has the steps.
- Added cache-first reads for messages, conversations, threads, users, groups, group members, reactions and the notification feed. Each fetch returns cached data first, then the server's, through new fetch listeners such as `MessageFetchListener` and `ConversationFetchListener`.
- Added queued sending while offline: messages sent without a connection are stored and sent automatically on reconnect. Manage the queue with `getPendingMessages()`, `retryMessage()`, `retryAllFailedMessages()`, `discardLocalMessage()` and `addOfflineMessageListener()`, read why a send failed from `BaseMessage.failureReason`, and size the queue with the `OfflineConfig` limits and overflow policy.
- Added reconnect sync, which catches up on missed messages, edits, deletes and conversation changes after a reconnect. Observe or drive it with `addSyncListener()`, `forceSyncNow()`, `getSyncState()` and `getLastSyncTimestamp()`.
- Added media caching, controlled by `MediaCachingMode`, with `downloadMedia()`, `getCachedMediaPath()`, `addMediaDownloadListener()` and `getCacheSize()`.
- Added encryption of the offline cache at rest through `OfflineConfig.encryption`, with `getOfflineEncryptionStatus()` to inspect it and `EncryptionFailureBehavior` to choose what happens when encryption is unavailable. Encryption also needs the `sqlite3mc` build-hook stanza in your app's own `pubspec.yaml`, and on web a copy of `sqlite3mc.wasm`; the README shows both.
- Added `clearOfflineCache()` to remove cached data, and `addOfflineCacheResetListener()` to learn when the SDK resets the cache.
- Added `message.status`, a `CometChatMessageStatus`, to read a message's delivery state.
- Added two-phase listeners for single lookups: `ConversationDetailsListener` for `getConversation` and `MessageDetailsListener` for `CometChatHelper.getMessageDetails` deliver the cached result first, then the server's.

**Enhancements**
- Improved cross-device sync: a conversation pinned or unpinned on another device now triggers `onConversationPinned` or `onConversationUnpinned`.
- Improved `MessageThread.subscribed`, which is now reliable on the default "my threads" list. On a `participatedByMe(false)` list it reads `false` unless the server provides the flag, and thread lists now send their page size as `per_page`.
- Raised the `ReactionsRequest` limit from 20 to a maximum of 1000.
- Changed `getMediaCachingMode()` to return `OFF` while offline mode is not active.
- Changed `isOfflineEnabled()` to report whether offline mode is running, not only whether it is configured.

**Fixes**
- Fixed paging on thread, pinned and saved message lists, which now return the next page; earlier versions sent a stale cursor from page 2 onward.
- Fixed message edits failing when the message metadata was an empty list.
- Fixed user and group timestamps that arrived in milliseconds being misread; they are now read as seconds.
- Fixed `lastReadMessageId` and `latestMessageId` being misread when the server sent them as strings.
- Fixed `toJson` throwing for a message with no receiver.
- Fixed a login that a newer `login` call had replaced still completing its session setup in the background.
- Fixed `markMessageAsUnread` on a received one-to-one message marking the logged-in user's own conversation instead of that chat.

**Breaking Changes**
- Raised the minimum requirements to Dart 3.10, Flutter 3.38 and iOS 13 (the Android minimum stays at API 21), and the minimum `flutter_secure_storage` to 10.1.0, along with newer `drift` and `sqlite3` minimums. **Update your toolchain and dependency constraints before upgrading.**
- Changed the package to a Flutter plugin that downloads a prebuilt SQLite binary from GitHub at build time, through the `sqlite3` package's native build hook. **Build machines must be able to reach GitHub**; air-gapped or locked-down CI needs an allowance. The package is also about 0.8 MB larger (`sqlite3mc.wasm`).
- Changed logging in as a different user to end the previous session first: `logoutSuccess` fires, and the connection drops and reconnects under the new user. A failed re-login that ends the session fires `logoutSuccess`, then `loginFailure`.
- Changed `setAttachment` to append to a message's attachments instead of replacing them. Use `setAttachments(list)` to replace them.
- Changed `BlockedUsersRequest` with no direction set to request `blockedByMe` instead of `both`. Set the direction to `BlockedUsersRequest.directionBoth` to keep the old behaviour.
- Added members to public types such as `MessagesRequest` and `MessageListener`. A class that `implements` one of them must add the new members.
- Added new top-level exports such as `OfflineConfig`, `CometChatMessageStatus`, `SyncState` and `MessageSource`. If one clashes with a name in your app, import the SDK with `hide` or a prefix.

## 5.0.9

**New**
- None

**Enhancements**
- Improved WebSocket reconnection with progressive delays and automatic credential refresh.
- Improved manual connection handling, logout, and disconnect to prevent unwanted connections and preserve registered listeners.
- Updated `ping()` and connection error handling for clearer failure reporting.

**Fixes**
- Fixed reconnection failures, stuck connection attempts, duplicate WebSockets, and hanging `connect()` or `disconnect()` calls.
- Fixed login, session, and initialization issues affecting listeners and connection state.
- Fixed brief app interruptions being incorrectly treated as backgrounding.

## 5.0.8

**New**
- None

**Enhancements**
- None

**Fixes**
- Fixed read and delivery receipts being silently lost after an app relaunch that restored a saved session instead of calling `login()` again. `markAsRead` and `markAsDelivered` reported success, but the receipt was never delivered to the other participant; receipts now work on restored sessions.

## 5.0.7

**New**
- Added thread subscriptions, allowing users to manage notifications for replies in specific threads.
- Added support for viewing and managing threads, including filtering by user or group.
- Added message pinning and saving, making it easier to highlight important messages and bookmark messages for later.
- Added conversation pinning to help users keep important conversations easily accessible.
- Added notification preferences for quoted replies and thread activity.
- Added support for registering web push notifications.

**Enhancements**
- None

**Fixes**
- Fixed an issue where ending a call did not notify the other participant, causing their call screen to remain open.
- Fixed an issue where transient messages were reported as sent but were not delivered to recipients.
- Fixed crashes that could occur when notification preferences contained missing or unsupported values.

## 5.0.6

**New**
- Added a multi-file upload API for sending several attachments in one message. `CometChat.createUploadFileRequest()` returns an `UploadFileRequest` you upload through and manage per file — `uploadAttachment` / `uploadAttachments`, `retryAttachment`, `removeAttachment`, `clearAll` — and inspect with `getStatus()`, `getAttachments()`, and `getAttachmentCount()`.
- Added `UploadFileListener` for per-file progress, success, rejection, and failure updates, with rejection (`onFileError`) and retryable failure (`onFileFailure`) as distinct callbacks.

**Enhancements**
- Improved Web and WebAssembly (`dart2wasm`) support.
- Added caption editing for `image`, `video`, `audio`, and `file` messages via `editMessage()`, which now returns `Future<BaseMessage?>` (resolving to `null` on failure).
- Relicensed the package under the MIT License.

**Fixes**
- None

## 5.0.5

**New**
- Added a `CardMessage` class for receiving developer card messages, with accessors for the card payload, fallback text, and `data.text`.
- Added an `onCardMessageReceived` callback to `MessageListener` for real-time card message delivery.
- Added `AIAssistantCardStartedEvent`, `AIAssistantCardReceivedEvent`, and `AIAssistantCardEndedEvent` for AI assistant card streaming through `onAIAssistantEventReceived`.
- Added an `AIAssistantElement` model with `getType()` and `getData()` to represent ordered content blocks in AI assistant messages.
- Added `getElements()` and `setElements()` to `AIAssistantMessage` for working with ordered `AIAssistantElement` blocks from `data.elements`.

**Enhancements**
- None

**Fixes**
- None

## 5.0.4

**New**
- None

**Enhancements**
- None

**Fixes**
- Fixed an issue where custom server configurations could fail when endpoint URLs were provided without `https://`.

## 5.0.3

**New**
- None

**Enhancements**
- Improved the SDK reconnect behavior by stopping further reconnection attempts after a connection-not-allowed error is received from the WebSocket connection.

**Fixes**
- Fixed an issue on web that could prevent some applications from connecting successfully in browser environments.

## 5.0.2

**New**
- Added Notification Feed support with new data models including `NotificationFeedItem`, `NotificationCategory`, and `PushNotification`.
- Added `FeedReadState` enum with values `read`, `unread`, and `all` for filtering notification feed items by read status.
- Added `FeedEngagementType` enum with values `viewed`, `clicked`, and `interacted` for reporting user engagement on feed items.
- Added `NotificationFeedRequestBuilder` for fetching paginated notification feed items with filters for read state, category, channel ID, tags, and date range.
- Added `NotificationCategoriesRequestBuilder` for fetching paginated notification categories.
- Added `markFeedItemAsDelivered()` and `markFeedItemsAsDelivered()` methods to mark notification feed items as delivered.
- Added `markFeedItemAsRead()` method to mark a notification feed item as read.
- Added `markAllFeedItemsAsRead()` method to mark all notification feed items as read in a single call.
- Added `reportFeedEngagement()` method to report user engagement (viewed, clicked, interacted) on a feed item.
- Added `getNotificationFeedUnreadCount()` method to retrieve the unread count for notification feed items.
- Added `getNotificationFeedItem()` method to fetch a single notification feed item by ID for deep linking.
- Added `markPushNotificationDelivered()` method to mark a push notification as delivered.
- Added `markPushNotificationClicked()` method to mark a push notification as clicked.
- Added `NotificationFeedListener` for receiving real-time notification feed items via WebSocket with `onFeedItemReceived()` callback.
- Added `addNotificationFeedListener()` and `removeNotificationFeedListener()` methods for managing real-time notification feed listeners.

**Enhancements**
- None

**Fixes**
- None

## 5.0.1

**New**
- None

**Enhancements**
- Enhanced CometChat Skills to improve support for agentic integrations and workflow interoperability.

**Fixes**
- Fixed an issue where push notification tokens were not registered correctly on iOS devices, preventing reliable notification delivery.

**Deprecations**
- None

**Removals**
- None

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

## 5.0.0-beta.1

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

