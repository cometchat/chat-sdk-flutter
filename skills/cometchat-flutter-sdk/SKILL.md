---
name: cometchat-flutter-sdk
description: Integrate CometChat into any Flutter app. Auto-detects your project setup and walks you through adding chat — init, auth, messaging, groups, real-time events. Start here — do not invoke feature-specific skills directly. Trigger with "add CometChat to my Flutter app", "integrate CometChat", "set up chat in Flutter", or any mention of CometChat Flutter SDK.
license: "MIT"
compatibility: "Flutter >=3.x; Dart >=3.x; cometchat_sdk ^5.0.0; Kiro, Claude Code, Cursor, or any agentskills.io-compatible agent"
metadata:
  author: "CometChat"
  version: "1.0.0"
  tags: "chat cometchat flutter dart messaging real-time groups"
---

# CometChat Flutter SDK Integration

You are the entry point for all CometChat Flutter integrations. Follow every step in order.

> **Trigger phrases:** "add CometChat", "integrate CometChat", "set up chat in Flutter", "CometChat Flutter", "cometchat_sdk"

---

## AUTONOMOUS MODE

If ALL of the following are true, proceed without asking:
1. `pubspec.yaml` contains `cometchat_sdk` dependency
2. Credentials exist in code or config (APP_ID, REGION)
3. The user's request clearly maps to a specific feature

If ANY require clarification, ask once and proceed.

DO NOT ask for confirmation on:
- SDK version detection (read pubspec.yaml)
- Whether to use Completer pattern (always use it)
- Whether to check getLoggedInUser before login (always check)

---

## Step 1 — Detect Project & Ensure SDK Installed

Read `pubspec.yaml` to confirm Flutter project and check SDK status.

**Detection rules (execute in order):**

1. `pubspec.yaml` does NOT exist → **Not a Flutter project.** Stop and tell the user.

2. `pubspec.yaml` exists → **Flutter project confirmed.** Check dependencies:

3. Check if `cometchat_sdk` is in `dependencies`:
   - **Version ≥5.0.0 found** → SDK ready. Proceed to Step 2.
   - **Version <5.0.0 found** → Outdated. Tell user: "You have CometChat SDK v4.x. These skills require v5+. Want me to upgrade?"
     - If yes → update the dependency and run `flutter pub get`
     - If no → stop
   - **Not found** → SDK not installed. **Install it automatically:**

**Auto-install flow (when SDK not present):**

```yaml
# For current beta — add Cloudsmith hosted dependency
dependencies:
  cometchat_sdk:
    hosted:
      url: https://dart.cloudsmith.io/cometchat/cometchat/
    version: 5.0.0-beta.1
```

Execute these steps without asking:
1. Read `pubspec.yaml`
2. Add `cometchat_sdk` to dependencies (use Cloudsmith URL for beta, or `cometchat_sdk: ^5.0.0` when stable)
3. Run `flutter pub get`
4. Verify no errors in output
5. If `flutter pub get` fails → show the error and stop

**iOS additional setup (check if `ios/` folder exists):**
- Add to `ios/Podfile` inside `post_install`:
  ```ruby
  target.build_configurations.each do |build_configuration|
    build_configuration.build_settings['EXCLUDED_ARCHS[sdk=iphonesimulator*]'] = 'arm64 i386'
  end
  ```
- Set iOS deployment target to 12 or higher
- Run `pod install` in the `ios/` directory

**Android additional setup (check if `android/` folder exists):**
- Verify `minSdkVersion` is 21 or higher in `android/app/build.gradle`

After SDK is installed and verified, proceed to Step 2.

**Log one line:** "CometChat SDK v5 installed successfully — proceeding with integration."

---

## Step 2 — Determine What the User Needs

Map the user's request to the right feature skill:

| User Says | Feature | Skill to Read |
|-----------|---------|---------------|
| "set up CometChat", "initialize", "init" | SDK Setup | `cometchat-flutter-init` |
| "login", "auth", "sign in to chat" | Authentication | `cometchat-flutter-auth` |
| "send messages", "chat screen", "messaging" | Messaging | `cometchat-flutter-messaging` |
| "typing indicator", "online status", "read receipts" | Real-time | `cometchat-flutter-realtime` |
| "group chat", "create group", "add members" | Groups | `cometchat-flutter-groups` |
| "error handling", "retry", "error codes" | Errors | `cometchat-flutter-error-handling` |
| "add chat to my app" (general) | Full integration | Follow Steps 3-7 below |

If the request maps to a specific feature, read that skill and follow it. If it's a general "add chat" request, follow the full integration flow below.

---

## Step 3 — Collect Credentials

**Inference order — check all before asking:**

1. Check `.env`, `lib/constants.dart`, or any config file for APP_ID, REGION, AUTH_KEY
2. Grep source for `CometChat.init` — extract credentials from existing call
3. Check for existing auth token endpoint in the project

Only ask for values still missing. Default REGION to "us" if not specified.

> ⚠ Auth Key is for development only. In production, use server-generated Auth Tokens via `loginWithAuthToken`.

---

## Step 4 — Initialize SDK

Read and follow `cometchat-flutter-init` skill. Key points:

First, ensure the import is in the file:
```dart
import 'package:cometchat_sdk/cometchat_sdk.dart';
```

```dart
// AppSettingsBuilder — appId is NOT a property, it goes to CometChat.init()
AppSettings appSettings = (AppSettingsBuilder()
  ..subscriptionType = CometChatSubscriptionType.allUsers
  ..region = "us"
  ..autoEstablishSocketConnection = true
).build();

// Use Completer to bridge callback → async/await
final completer = Completer<void>();
CometChat.init("APP_ID", appSettings,
  onSuccess: (_) => completer.complete(),
  onError: (e) => completer.completeError(e),
);
await completer.future;
```

Place in `main()` before `runApp()`. Call `WidgetsFlutterBinding.ensureInitialized()` first.

---

## Step 5 — Set Up Authentication

Read and follow `cometchat-flutter-auth` skill. Key points:

```dart
// Always check existing session before login
final existingUser = await CometChat.getLoggedInUser();
if (existingUser == null) {
  await CometChat.loginWithAuthToken(authToken, onSuccess: ..., onError: ...);
}
```

- Use `loginWithAuthToken` for production (auth key never in client code)
- Unregister push tokens before logout: `CometChatNotifications.unregisterPushToken()`

---

## Step 6 — Add Chat Features

Based on what the user needs, read the appropriate skill:

**Basic chat screen** → `cometchat-flutter-messaging`
- TextMessage with CometChatConversationType.user/.group
- MessageListener in initState, removeMessageListener in dispose
- MessagesRequestBuilder for history (fetchPrevious for older, fetchNext for newer)

**Real-time features** → `cometchat-flutter-realtime`
- Typing: CometChat.startTyping/endTyping + MessageListener.onTypingStarted/onTypingEnded
- Presence: UserListener.onUserOnline/onUserOffline (requires subscriptionType in init)
- Receipts: CometChat.markAsRead + MessageListener.onMessagesRead

**Group chat** → `cometchat-flutter-groups`
- createGroupWithMembers for atomic create+add
- GroupListener for membership events
- Owner must transferGroupOwnership before leaveGroup

---

## Step 7 — Error Handling

Read `cometchat-flutter-error-handling` for:
- Error code → user-friendly message mapping
- Retry with exponential backoff (only for network errors)
- Never leave onError callbacks empty

---

## HARD RULES

These are non-negotiable. Every feature skill inherits them.

```
INIT_BEFORE_EVERYTHING
  CometChat.init() MUST complete before any other SDK call. No exceptions.

SESSION_CHECK_BEFORE_LOGIN
  Always call getLoggedInUser() after init. Only login if null.

LISTENER_LIFECYCLE
  Register listeners in initState(), remove in dispose(). Always.
  Use unique listener IDs per screen/widget. Always.

COMPLETER_PATTERN
  CometChat methods use callbacks. Wrap with Completer for async/await.

NEVER_EMPTY_ONERROR
  Every onError callback MUST log and surface the error. Never swallow.

APPID_NOT_ON_BUILDER
  appId goes to CometChat.init(), NOT to AppSettingsBuilder.

REGION_LOWERCASE
  Region must be lowercase: 'us', 'eu', 'in'. Validated against exact list.

PUSH_TOKEN_ON_LOGOUT
  Call CometChatNotifications.unregisterPushToken() before CometChat.logout().
```

---

## Complete Integration Example

For a full working example, see the recipe files:
- Basic chat: `doc/recipes/01-basic-chat.dart`
- Group chat: `doc/recipes/02-group-chat.dart`
- Typing indicators: `doc/recipes/03-typing-indicators.dart`

---

## ERROR DEBUGGING TABLE

| Symptom | Cause | Fix |
|---------|-------|-----|
| ERR_NOT_INITIALIZED | SDK method called before init completes | Gate all calls behind init onSuccess |
| ERR_INVALID_REGION | Wrong region string | Use lowercase: 'us', 'eu', 'in' |
| No presence events | Missing subscriptionType in init | Set CometChatSubscriptionType.allUsers |
| Listener leak | Missing removeListener in dispose | Always pair add/remove in initState/dispose |
| Messages on wrong screen | Duplicate listener IDs | Use unique IDs per screen instance |
| ERR_FILTERS_MISSING | fetchNext without cursor | Use fetchPrevious for history, fetchNext needs messageId |
| Owner can't leave group | Must transfer ownership first | Call transferGroupOwnership before leaveGroup |

---

## Companion: CometChat Docs MCP

For the best experience, also install the CometChat Docs MCP server for live access to the full documentation:

Kiro (`.kiro/settings/mcp.json`):
```json
{
  "mcpServers": {
    "cometchat-docs": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-remote", "https://www.cometchat.com/docs/mcp"]
    }
  }
}
```

No authentication required.
