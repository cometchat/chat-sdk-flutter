# CometChat Flutter SDK v5

This repository contains the CometChat Chat SDK for Flutter. When working with this codebase, use the agent skills in the `skills/` directory for SDK-specific guidance.

## Skills

Load the relevant skill based on the task:

- `skills/cometchat-flutter-init/SKILL.md` — SDK initialization, AppSettings, region config
- `skills/cometchat-flutter-auth/SKILL.md` — Login, logout, auth tokens, session management
- `skills/cometchat-flutter-messaging/SKILL.md` — Sending/receiving messages, listeners, pagination
- `skills/cometchat-flutter-realtime/SKILL.md` — Typing indicators, presence, read receipts
- `skills/cometchat-flutter-groups/SKILL.md` — Group CRUD, member management, kick/ban
- `skills/cometchat-flutter-error-handling/SKILL.md` — CometChatException, error codes, retry patterns
- `skills/cometchat-flutter-sdk/SKILL.md` — General SDK overview and architecture
- `skills/cometchat-flutter-core/SKILL.md` — Core SDK patterns and conventions
- `skills/cometchat-flutter-compositions/SKILL.md` — Chaining SDK methods: registration flows, conversation patching, AI gating, bootstrap sequences

## Key Rules

- CometChat.init() uses callbacks (onSuccess/onError), not direct returns. Use a Completer for async/await.
- Always call CometChat.getLoggedInUser() before login() to check for existing sessions.
- Use auth tokens in production, not auth keys.
- Documentation: https://www.cometchat.com/docs/sdk/flutter/overview