---
name: apple-platform-automation
description: "Use when automating Apple/macOS services: Notes, Reminders, Messages, Find My, or GUI-only macOS apps. Provides class-level workflows with service-specific command subsections."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [apple, macos, notes, reminders, imessage, findmy, computer-use]
    related_skills: []
---

# Apple Platform Automation

## Overview

Use this umbrella when the user asks to operate Apple/macOS-only surfaces from Hermes: Apple Notes, Reminders, Messages/SMS, Find My, or apps that only expose a GUI. Prefer purpose-built CLIs when available; fall back to macOS computer-use automation only when the surface has no reliable command/API path.

## Routing Decision

1. **Notes knowledge capture or lookup** → use the `memo` CLI subsection.
2. **Todos/reminders** → use the `remindctl` subsection.
3. **iMessage/SMS** → use the `imsg` CLI subsection.
4. **AirTags/devices in Find My** → use AppleScript/screenshot/Peekaboo-style UI automation.
5. **Any other macOS GUI app** → use the macOS computer-use canonical workflow.

## Apple Notes via `memo`

### Prerequisites

- Running on a macOS host with Apple Notes access.
- `memo` CLI installed/configured for the user account.

### Common operations

```bash
memo search "project name"
memo create "Title" --body "Body text"
memo edit "Title" --append "New note text"
```

Use Notes for durable user-facing note content. Do not store Hermes operational memory in Apple Notes unless explicitly requested.

## Apple Reminders via `remindctl`

### Common operations

```bash
remindctl list
remindctl add "Call Alice" --due "tomorrow 9am"
remindctl complete <reminder-id-or-title>
```

Use explicit dates/times when the user gives them; if natural language date parsing is ambiguous, state the interpreted date before creating many reminders.

## Messages via `imsg`

### Common operations

```bash
imsg chats
imsg send "+15551234567" "Message text"
imsg read --limit 20
```

Confirm the recipient identity when multiple contacts/numbers match. For sensitive or irreversible messages, show the exact outgoing text and target before sending unless the user already supplied an unambiguous send command.

## Find My / AirTags / Device Location

Find My usually requires UI automation rather than a stable CLI. Recommended order:

1. Try a direct AppleScript/UI query if a known script exists for the host.
2. Open Find My and use screenshot-based inspection for the current visible location.
3. Use Peekaboo or the configured macOS computer-use tool to click/search/select the device.
4. For tracking over time, capture timestamped screenshots and transcribe visible location text; do not infer precision beyond what the UI shows.

## macOS Computer-Use Canonical Workflow

Use for GUI-only operations. The stable loop is:

1. Capture the current screen/window.
2. Identify the exact target control and coordinates/accessibility label.
3. Act once.
4. Re-capture and verify.
5. Continue in small steps; avoid blind multi-click macros.

### Text input pattern

Prefer selecting the target field, clearing with the platform shortcut, typing/pasting once, then verifying the text appears. For drag/drop or complex gestures, take before/after screenshots and keep coordinate assumptions explicit.

## Common Pitfalls

1. **Using GUI automation when a CLI exists.** CLI operations are usually faster, scriptable, and easier to verify.
2. **Assuming a cloud service is reachable from Linux.** Apple Notes/Reminders/Messages/Find My generally require the user’s macOS environment.
3. **Overstating Find My precision.** Report only the visible UI data and timestamp.
4. **Sending messages to ambiguous contacts.** Resolve identity before sending.
5. **Blind UI sequences.** Always recapture after each meaningful UI action.

## Verification Checklist

- [ ] Confirmed the current runtime can access the required Apple/macOS service.
- [ ] Used the narrow CLI path before GUI automation where available.
- [ ] Verified the final state by reading/listing/capturing the target surface.
- [ ] For messages, verified recipient and exact content.
