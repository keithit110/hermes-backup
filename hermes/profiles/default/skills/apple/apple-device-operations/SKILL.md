---
name: apple-device-operations
description: "Operate Apple ecosystem apps and device surfaces from Hermes: Notes, Reminders, iMessage/SMS, Find My, and macOS UI automation."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [apple, macos, notes, reminders, imessage, findmy, computer-use]
    related_skills: []
---

# Apple Device Operations

## Overview

Use this umbrella when a task touches local Apple surfaces rather than a web API: Apple Notes, Reminders, Messages/iMessage/SMS, Find My, or macOS UI automation. Start with purpose-built CLIs when available, and fall back to macOS computer-use / AppleScript / screenshots only when the target app has no reliable CLI path.

## When to Use

- Create, search, edit, or delete Apple Notes.
- Add, list, complete, or inspect Apple Reminders.
- Send/read iMessage or SMS history from macOS.
- Locate Apple devices or AirTags through Find My.
- Drive a macOS GUI when a CLI/API is missing.

## Decision Tree

1. **Notes:** prefer `memo` for search/create/edit/delete. Verify the note exists after writes.
2. **Reminders:** prefer `remindctl` for list CRUD, completion, due dates, and alarms.
3. **Messages:** prefer `imsg` for chats, history, send, and watch flows. Confirm recipient/chat identity before sending.
4. **Find My:** use FindMy.app automation plus screenshots/Peekaboo. Treat locations as approximate and time-stamped.
5. **Generic app UI:** use macOS computer-use tooling with explicit capture/action/verification loops.

## Notes Workflow

- Check prerequisites and app permissions first.
- Search before creating duplicates.
- After edits/deletes, read the target again or search by title/body to verify state.

## Reminders Workflow

- List reminder lists before choosing a target list.
- Distinguish due time from early alarm/nudge.
- Verify completion or creation by listing the relevant list after mutation.

## Messages Workflow

- Resolve the chat/contact before sending.
- For history retrieval, include enough context for disambiguation and quote exact messages when reporting.
- Sending messages is an external side effect: use the user's target and wording exactly.

## Find My Workflow

- Prefer UI automation plus screenshot capture when no CLI is available.
- Record device name, displayed location, and timestamp.
- Do not overclaim precision; Find My results may be stale or approximate.

## macOS UI Automation Workflow

1. Capture current screen/window state.
2. Identify the target control and expected result.
3. Act once.
4. Capture again and verify.
5. Repeat in small steps; do not batch blind UI actions.

## Pitfalls

- macOS permissions (Automation, Accessibility, Full Disk Access) often block otherwise-correct commands.
- iCloud sync delay can make recently created Notes/Reminders briefly invisible.
- GUI focus changes invalidate coordinates; recapture before each meaningful action.
- Messages and Reminders mutations are user-visible side effects; avoid speculative actions.

## Verification Checklist

- [ ] Correct Apple surface chosen before fallback UI automation.
- [ ] Permissions/setup checked when a command fails.
- [ ] Mutating action verified by read/list/screenshot.
- [ ] User-visible side effects reported with exact target and result.
