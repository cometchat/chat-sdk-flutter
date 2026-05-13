---
name: cometchat-flutter-error-handling
description: Use when handling CometChat errors in a Flutter app. Triggers on mentions of CometChatException, error codes like ERR_NOT_INITIALIZED, ERR_INVALID_APP_ID, ERROR_USER_NOT_LOGGED_IN, onError callbacks, or any CometChat error. Also use when the user asks about retry logic, error recovery, user-friendly error messages for chat failures, or why a CometChat method is throwing an exception. Make sure to use this skill whenever the user encounters a CometChat error or asks about graceful error handling, even if they just say 'my chat app keeps crashing' or 'how do I handle failures'.
---

# CometChat Flutter SDK v5 Error Handling

All CometChat errors arrive as `CometChatException` in onError callbacks. The exception has three fields: `code` (String), `message` (String?), `details` (String?). Never leave onError empty.

## CometChatException Structure

```dart
class CometChatException implements Exception {
  String code;      // Machine-readable error code
  String? details;  // Human-readable details
  String? message;  // Additional context
}
```

Internal SDK uses typed exceptions (AuthException, NetworkException, StateException, ValidationException, NotFoundException) — all converted to CometChatException before reaching your onError callback.

## Error Code Reference

### Init Errors (SdkErrorCode)

| Code | Meaning | Fix |
|------|---------|-----|
| ERR_INVALID_APP_ID | appId empty or wrong | Check Dashboard credentials |
| ERR_INVALID_REGION | Region not in ['us','eu','in'] | Use lowercase, match Dashboard |
| ERR_ALREADY_INITIALIZED | init() called again (same creds) | Safe to ignore — returns success |
| ERR_INITIALIZATION_FAILED | General init failure | Check network, logs |
| ERR_NOT_INITIALIZED | SDK method before init completes | Gate all calls behind init onSuccess |

### Auth Errors

| Code | Meaning | Fix |
|------|---------|-----|
| ERR_AUTH_TOKEN_EXPIRED | Token TTL exceeded | Generate new token from server |
| ERROR_USER_NOT_LOGGED_IN | Method requires auth | Login first |

### Validation Errors (ErrorCode)

| Code | Meaning | Fix |
|------|---------|-----|
| ERROR_INVALID_RECEIVER_ID | Empty/null receiver | Validate before sending |
| ERROR_INVALID_RECEIVER_TYPE | Not 'user' or 'group' | Use CometChatReceiverType constants |
| ERROR_NON_POSITIVE_LIMIT | Limit ≤ 0 | Use positive number |
| ERROR_LIMIT_EXCEEDED | Limit > max (100) | Reduce limit |
| ERROR_INVALID_GUID | Empty/null GUID | Validate before use |
| ERROR_INVALID_MESSAGE_ID | Empty/null message ID | Validate before use |
| ERROR_REQUEST_IN_PROGRESS | Duplicate concurrent request | Wait for previous to finish |

### Notification Errors

| Code | Meaning | Fix |
|------|---------|-----|
| INVALID_PUSH_PLATFORM | Wrong platform for device | Match platform to device (FCM for Android, APNS for iOS) |
| INVALID_FCM_TOKEN | Bad FCM token | Refresh token |
| INVALID_APNS_DEVICE_TOKEN | Bad APNS device token | Refresh token |
| INVALID_APNS_VOIP_TOKEN | Bad APNS VoIP token | Refresh token |

### Calling Errors

Calling uses two exception types: `CometChatException` (from Chat SDK signaling) and `CometChatCallsException` (from Calls SDK WebRTC session).

| Error | Source | Meaning | Fix |
|-------|--------|---------|-----|
| Participants exceed limit | CometChatException | Group has more members than the call participant cap (default 50) | Check Dashboard plan limits. SUGGESTION: Consider gating the call button on `group.membersCount` |
| Token generation failed | CometChatCallsException | `generateToken()` failed — invalid session ID or auth token | Ensure user is logged in, session ID is valid |
| Session start failed | CometChatCallsException | `startSession()` failed — invalid token or network issue | Regenerate token and retry |
| MissingPluginException | Platform error | Calls SDK used on unsupported platform | Calls SDK supports Android and iOS only (method channels). Guard with platform check |

## Error Handling Pattern

```dart
CometChat.sendMessage(textMessage,
  onSuccess: (TextMessage message) {
    // Handle success
  },
  onError: (CometChatException e) {
    debugPrint("Error: ${e.code} - ${e.message}");

    // Map to user-friendly message
    final userMessage = _mapErrorToUserMessage(e.code);
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(userMessage)),
    );
  },
);
```

## Error → User Message Mapping

```dart
String mapCometChatError(String code) {
  switch (code) {
    case 'ERR_NOT_INITIALIZED':
      return 'Chat service is starting up. Please wait.';
    case 'ERR_INVALID_APP_ID':
    case 'ERR_INVALID_REGION':
      return 'Chat configuration error. Please contact support.';
    case 'ERR_AUTH_TOKEN_EXPIRED':
      return 'Your session has expired. Please sign in again.';
    case 'ERROR_USER_NOT_LOGGED_IN':
      return 'Please sign in to use chat.';
    case 'ERROR_INVALID_RECEIVER_ID':
      return 'Could not find the recipient.';
    case 'ERROR_REQUEST_IN_PROGRESS':
      return 'Please wait, request is being processed.';
    case 'ERR_INITIALIZATION_FAILED':
      return 'Could not connect to chat. Check your internet connection.';
    default:
      return 'Something went wrong. Please try again.';
  }
}
```

## Retry Pattern with Exponential Backoff

The SDK defines retry constants: max 3 attempts, initial delay 1s, max delay 10s, multiplier 2x.

```dart
Future<T> retryWithBackoff<T>({
  required Future<T> Function() operation,
  int maxAttempts = 3,
  Duration initialDelay = const Duration(seconds: 1),
  Set<String> retryableCodes = const {'ERR_INITIALIZATION_FAILED'},
}) async {
  Duration delay = initialDelay;

  for (int attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await operation();
    } on CometChatException catch (e) {
      if (!retryableCodes.contains(e.code) || attempt == maxAttempts) rethrow;
      debugPrint("Attempt $attempt failed (${e.code}). Retrying in ${delay.inSeconds}s...");
      await Future.delayed(delay);
      delay *= 2;
      if (delay > const Duration(seconds: 10)) delay = const Duration(seconds: 10);
    }
  }
  throw StateError('Unreachable');
}
```

## Error Categories for Retry Decisions

| Category | Codes | Retry? | Action |
|----------|-------|--------|--------|
| Network | ERR_INITIALIZATION_FAILED | Yes | Exponential backoff |
| Auth | ERR_AUTH_TOKEN_EXPIRED | No | Refresh token, re-login |
| Validation | ERROR_INVALID_*, ERROR_NON_POSITIVE_LIMIT | No | Fix input, don't retry |
| State | ERR_NOT_INITIALIZED, ERROR_USER_NOT_LOGGED_IN | No | Fix call order |
| Concurrency | ERROR_REQUEST_IN_PROGRESS | Wait | Wait for previous request |
| Config | ERR_INVALID_APP_ID, ERR_INVALID_REGION | No | Fix configuration |

## Anti-Patterns

**Stripping CometChatException in repository/wrapper layers:**
```dart
// ❌ WRONG — converts typed exception to String, loses error code
// Upstream code can no longer do retry decisions or user-friendly mapping
catch (e) { throw e.toString(); }

// ✅ CORRECT — preserve the typed exception
on CometChatException catch (e) { rethrow; }
catch (e) { throw CometChatException('UNKNOWN', details: e.toString()); }
```

**Empty onError — silently swallowing errors:**
```dart
// ❌ WRONG
CometChat.sendMessage(msg, onSuccess: (_) {}, onError: (_) {});

// ✅ CORRECT — always log and handle
CometChat.sendMessage(msg,
  onSuccess: (_) {},
  onError: (e) {
    debugPrint("Send failed: ${e.code}");
    showUserFriendlyError(e);
  },
);
```

**Retrying non-retryable errors:**
```dart
// ❌ WRONG — retrying a validation error loops forever
while (true) {
  try { await sendMessage(msg); break; }
  catch (e) { await Future.delayed(Duration(seconds: 1)); }
}

// ✅ CORRECT — only retry network errors
if (e.code == 'ERR_INITIALIZATION_FAILED') { retry(); }
else { showError(e); }
```

**Catching generic Exception instead of CometChatException:**
```dart
// ❌ WRONG — loses error code information
try { await operation(); }
catch (e) { print(e.toString()); }

// ✅ CORRECT — preserve the typed exception
try { await operation(); }
on CometChatException catch (e) { handleCometChatError(e); }
catch (e) { handleUnexpectedError(e); }
```

## Checklist

- Every onError callback logs the error code and surfaces a user-friendly message
- Error codes mapped to user-friendly strings (not raw codes shown to users)
- Retry logic only for network/transient errors, not validation/config errors
- CometChatException caught specifically (not generic Exception)
- ERR_AUTH_TOKEN_EXPIRED triggers re-authentication flow, not retry
- ERROR_REQUEST_IN_PROGRESS handled by waiting, not retrying immediately
