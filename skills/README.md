# CometChat Flutter SDK Skills

Agent skills for building with the CometChat Flutter SDK v5. Install individually or browse all available skills.

## How It Works

Skills are bundled in this repository under `skills/`. When you clone the repo and open it in a supported AI coding assistant, skills auto-trigger based on what you're doing — mention "CometChat init" and the init skill loads, ask about "typing indicators" and the realtime skill loads. No manual activation needed.

To use these skills in your own project, copy the `skills/` folder into your project root:

```bash
cp -r skills/ /path/to/your/project/skills/
```

## Available Skills

| Skill | Triggers On |
|-------|-------------|
| `cometchat-flutter-init` | SDK setup, CometChat.init(), AppSettingsBuilder, ERR_NOT_INITIALIZED |
| `cometchat-flutter-auth` | Login, logout, session management, auth tokens |
| `cometchat-flutter-messaging` | Sending/receiving messages, MessageListener, pagination |
| `cometchat-flutter-realtime` | Typing indicators, presence, read receipts, ConnectionListener |
| `cometchat-flutter-groups` | Group CRUD, member management, kick/ban, GroupListener |
| `cometchat-flutter-error-handling` | CometChatException, error codes, retry patterns |

## How Auto-Detection Works

Each skill has a `description` field in its YAML frontmatter that lists trigger keywords. When you mention something related (like "CometChat login" or "add chat to my app"), the agent reads the description, decides the skill is relevant, and loads its full content. You never need to manually select a skill.

## Compatibility

- CometChat Flutter SDK v5 (5.0.0-beta.1+)
- Flutter 3.x
- Dart 3.x
- Works with: Kiro, Claude Code, Cursor, Copilot, and other AI coding assistants that support the skills ecosystem
