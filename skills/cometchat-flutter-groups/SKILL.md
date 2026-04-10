---
name: cometchat-flutter-groups
description: Use when implementing group chat features with CometChat Flutter SDK v5. Triggers on mentions of Group, createGroup, joinGroup, leaveGroup, deleteGroup, GroupListener, GroupMember, member scope, kick, ban, unban, addMembersToGroup, GroupsRequestBuilder, GroupMembersRequestBuilder, or group CRUD operations. Also use when the user asks about public vs private groups, password-protected groups, managing group membership, or transferring group ownership. Make sure to use this skill whenever the user mentions group chat, channels, or multi-user conversations, even if they just say 'create a group chat' or 'add people to a room'.
---

# CometChat Flutter SDK v5 Groups

Three group types (public, password, private), three member scopes (admin, moderator, participant). Groups use GUID (alphanumeric, underscore, hyphen only).

## Group Types

| Type | Visibility | Join Method |
|------|-----------|-------------|
| `CometChatGroupType.public` | All users | Anyone can join |
| `CometChatGroupType.password` | All users | Requires password |
| `CometChatGroupType.private` | Members only | Invite only (auto-joined) |

## Member Scopes

| Scope | Privileges |
|-------|-----------|
| `CometChatMemberScope.admin` | Full control: manage all members, update/delete group |
| `CometChatMemberScope.moderator` | Kick/ban participants, update group |
| `CometChatMemberScope.participant` | Send/receive messages and calls (default) |

## Create Group

```dart
Group group = Group(guid: "my-group-1", name: "My Group", type: CometChatGroupType.public);

CometChat.createGroup(group: group,
  onSuccess: (Group group) { debugPrint("Created: ${group.guid}"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);
```

Create with members in one step:
```dart
List<GroupMember> members = [
  GroupMember(uid: "uid-1", scope: CometChatMemberScope.participant),
  GroupMember(uid: "uid-2", scope: CometChatMemberScope.admin),
];

CometChat.createGroupWithMembers(group: group, members: members,
  onSuccess: (Group group, Map<String, String> failedMembers) {
    if (failedMembers.isNotEmpty) debugPrint("Failed to add: $failedMembers");
  },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);
```

## Join / Leave / Delete

```dart
// Join (public or password group)
CometChat.joinGroup("group-guid", CometChatGroupType.public,
  password: "",  // required for password groups
  onSuccess: (Group group) { debugPrint("Joined: ${group.name}"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);

// Leave (owner must transfer ownership first)
CometChat.leaveGroup("group-guid",
  onSuccess: (String msg) { debugPrint("Left group"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);

// Delete (admin only)
CometChat.deleteGroup("group-guid",
  onSuccess: (String msg) { debugPrint("Deleted"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);
```

## Member Management

```dart
// Add members
List<GroupMember> members = [
  GroupMember.fromUid(scope: CometChatMemberScope.participant, uid: "uid-3", name: "User 3"),
];
CometChat.addMembersToGroup(guid: "group-guid", groupMembers: members,
  onSuccess: (Map<String?, String?> result) { debugPrint("Added: $result"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);

// Kick (admin/moderator only, kicked user can rejoin)
CometChat.kickGroupMember(guid: "group-guid", uid: "uid-3",
  onSuccess: (String msg) { debugPrint("Kicked"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);

// Ban (admin/moderator only, banned user cannot rejoin until unbanned)
CometChat.banGroupMember(guid: "group-guid", uid: "uid-3",
  onSuccess: (String msg) { debugPrint("Banned"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);

// Unban
CometChat.unbanGroupMember(guid: "group-guid", uid: "uid-3",
  onSuccess: (String msg) { debugPrint("Unbanned"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);

// Change scope (admin only)
CometChat.updateGroupMemberScope(guid: "group-guid", uid: "uid-3", scope: CometChatMemberScope.moderator,
  onSuccess: (String msg) { debugPrint("Scope changed"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);

// Transfer ownership (owner only, required before owner can leave)
CometChat.transferGroupOwnership(guid: "group-guid", uid: "uid-3",
  onSuccess: (String msg) { debugPrint("Ownership transferred"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);
```

## Fetching Groups & Members

```dart
// List groups (paginated)
GroupsRequest groupsRequest = (GroupsRequestBuilder()
  ..limit = 20
  ..joinedOnly = true  // only groups user has joined
).build();

groupsRequest.fetchNext(
  onSuccess: (List<Group> groups) { debugPrint("Groups: ${groups.length}"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);

// List group members (paginated)
GroupMembersRequest membersRequest = (GroupMembersRequestBuilder("group-guid")
  ..limit = 20
).build();

membersRequest.fetchNext(
  onSuccess: (List<GroupMember> members) { debugPrint("Members: ${members.length}"); },
  onError: (CometChatException e) { debugPrint("Failed: ${e.message}"); },
);
```

Builder filters: `searchKeyword`, `tags`, `withTags`, `joinedOnly` (groups), `scopes`, `status` (members).

## GroupListener (Real-time Events)

```dart
class _GroupScreenState extends State<GroupScreen> with GroupListener {
  static const _listenerId = "group_screen_listener";

  @override void initState() { super.initState(); CometChat.addGroupListener(_listenerId, this); }
  @override void dispose() { CometChat.removeGroupListener(_listenerId); super.dispose(); }

  @override void onGroupMemberJoined(Action action, User joinedUser, Group joinedGroup) { }
  @override void onGroupMemberLeft(Action action, User leftUser, Group leftGroup) { }
  @override void onGroupMemberKicked(Action action, User kickedUser, User kickedBy, Group kickedFrom) { }
  @override void onGroupMemberBanned(Action action, User bannedUser, User bannedBy, Group bannedFrom) { }
  @override void onGroupMemberUnbanned(Action action, User unbannedUser, User unbannedBy, Group unbannedFrom) { }
  @override void onGroupMemberScopeChanged(Action action, User updatedBy, User updatedUser, String scopeChangedTo, String scopeChangedFrom, Group group) { }
  @override void onMemberAddedToGroup(Action action, User addedBy, User userAdded, Group addedTo) { }
}
```

## Anti-Patterns

**Owner trying to leave without transferring ownership:**
```dart
// ❌ WRONG — owner cannot leave, will get error
CometChat.leaveGroup("guid", ...);
// ✅ Transfer ownership first, then leave
CometChat.transferGroupOwnership(guid: "guid", uid: "new-owner-uid", ...);
CometChat.leaveGroup("guid", ...);
```

**Kicked vs Banned confusion:**
- Kicked: user removed but CAN rejoin
- Banned: user removed and CANNOT rejoin until unbanned

**Not checking hasJoined before sending messages:**
```dart
// ❌ WRONG — sending to a group you haven't joined
CometChat.sendMessage(textMessage, ...); // Will fail

// ✅ Check group.hasJoined or join first
```

## Checklist

- GUID is alphanumeric with underscore/hyphen only (no spaces or special chars)
- Group type set correctly (public/password/private)
- Password provided for password-protected groups on join
- Owner transfers ownership before leaving
- GroupListener registered in initState, removed in dispose
- Unique listener IDs per screen
- Kicked vs banned distinction understood
- hasJoined checked before sending messages to a group
