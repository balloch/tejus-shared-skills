---
name: macos-messages-contacts-access
description: |
  Access macOS Messages and Contacts programmatically without Full Disk Access. Use when:
  (1) "Operation not permitted" error accessing ~/Library/Messages/chat.db or ~/Library/Application Support/AddressBook/,
  (2) Need to read iMessage history, contacts, or chat data from terminal/scripts,
  (3) Need to SAVE or update a contact (name, phone, email, Instagram/social handle, note),
  (4) CNContactStore/CNSaveRequest write fails with a CoreData 4097 XPC error,
  (5) Full Disk Access isn't granted or isn't desirable. Covers AppleScript automation
  permissions workaround, the read-vs-write permission split, and Beeper Desktop CLI as
  alternatives to direct database access.
author: Claude Code
user-invocable: false
---

# macOS Messages & Contacts Access

## Problem

Accessing macOS Messages (`chat.db`) or Contacts (`AddressBook`) databases directly from
terminal fails with "Operation not permitted" even when running legitimate scripts, because
these locations are protected by macOS privacy controls requiring Full Disk Access.

## Context / Trigger Conditions

- Running `ls ~/Library/Messages/` returns `Operation not permitted`
- Running `sqlite3 ~/Library/Messages/chat.db` fails
- Running `ls ~/Library/Application\ Support/AddressBook/` returns `Operation not permitted`
- You want to access message history or contacts without granting Full Disk Access to terminal
- Terminal app doesn't have Full Disk Access in System Settings

## Solution

### Option 1: AppleScript (Recommended for Quick Access)

AppleScript uses **Automation permissions** (different from Full Disk Access), which are
easier to grant and more granular.

**Access Messages:**
```bash
# List chat names
osascript -e 'tell application "Messages" to get name of every chat'

# Get recent messages from chats (limited API)
osascript -e '
tell application "Messages"
    set chatList to every chat
    repeat with c in chatList
        set chatName to name of c
        set msgs to messages of c
        -- Process messages
    end repeat
end tell
'
```

**Access Contacts:**
```bash
# List all contact names
osascript -e 'tell application "Contacts" to get name of every person'

# Get phone numbers
osascript -e 'tell application "Contacts" to get value of phones of every person'

# Get emails
osascript -e 'tell application "Contacts" to get value of emails of every person'
```

**First run:** macOS will prompt for Automation permission. Click "OK" to allow.

### Option 2: Beeper CLI (Recommended for Rich Access)

If Beeper Desktop is running locally:

- Provides structured access to messages across all connected platforms (iMessage, WhatsApp, Instagram, etc.)
- Offers search, filtering, and pagination
- No need for Full Disk Access or Automation permissions
- CLI: `beeper-desktop-cli` (see beeper skill for full command reference)
- Requires `BEEPER_ACCESS_TOKEN` env var

### Option 3: Full Disk Access (Direct Database)

If you need direct SQLite access:

1. Open **System Settings** → **Privacy & Security** → **Full Disk Access**
2. Click **+** and add your terminal app (Terminal.app, iTerm, VS Code, etc.)
3. Restart the terminal completely
4. Now you can directly query:
   ```bash
   sqlite3 ~/Library/Messages/chat.db "SELECT * FROM message LIMIT 10;"
   ```

## Writing to Contacts (not just reading)

**Reads and writes have different permission requirements.** A plain `swiftc`-compiled
binary using `CNContactStore` can *read* contacts with only the TCC Contacts grant, but
`CNSaveRequest` *writes* fail:

```
CoreData: error: XPC: synchronousRemoteObjectProxyWithErrorHandler: store
'.../AddressBook-v22.abcddb' encountered error: NSCocoaErrorDomain Code=4097
...
{"error":"save failed: A Core Data error occurred.","ok":false}
```

The XPC store refuses a read-write connection to an unsigned, unentitled binary. It retries
~8 times over 60s before failing, so it looks like a hang first.

**Fix: use AppleScript for writes.** Contacts.app scripting goes through Automation
permission, which the terminal already has:

```bash
osascript <<'EOF'
tell application "Contacts"
    set thePerson to make new person with properties {first name:"Jane", last name:"Doe", note:"context"}
    make new phone at end of phones of thePerson with properties {label:"mobile", value:"+1 (555) 555-1234"}
    make new social profile at end of social profiles of thePerson with properties {service name:"Instagram", user name:"janedoe"}
    save
end tell
EOF
```

`save` is required — without it the change stays in memory and is lost.

**Use `<repo-root>/scripts/save-contact`** rather than hand-rolling this. It is idempotent
(updates instead of duplicating) and supports `--first --last --nickname --company --note
--birthday --phone --email --instagram --twitter --linkedin --dry-run`.

```bash
./scripts/save-contact --first "Jane" --last "Doe" --phone "+15555551234" \
    --instagram janedoe --note "met at conference"
```

### Performance trap: never iterate people in AppleScript

`repeat with p in people` over a large address book takes **minutes** (it was still running
after 90s and had to be killed). Two safe patterns:

1. **Reads/matching** → use the fast Swift binary `<repo-root>/scripts/lookup-contacts`
   (~1s, uses `CNContactStore` which is fine for reading).
2. **AppleScript lookups** → only use native `whose` clauses, which Contacts.app evaluates
   internally: `people whose name is "Jane Doe"` returns in ~2s.

`save-contact` uses exactly this hybrid: `lookup-contacts` resolves a phone number to a
name, then a `whose` query resolves that name to a person id.

## Verification

**For AppleScript:**
```bash
osascript -e 'tell application "Messages" to get name of every chat' 2>&1 | head -5
```
Should return chat names (or "missing value" for unnamed chats).

**For Beeper CLI:**
`beeper-desktop-cli chats search --query "test" --limit 1 --format json`

**For Full Disk Access:**
```bash
ls ~/Library/Messages/chat.db
```
Should show the file without "Operation not permitted".

## Example

**Scenario:** Extract action items from recent messages

**Using AppleScript:**
```bash
# Get recent messages - note: AppleScript Messages API is limited
osascript -e '
tell application "Messages"
    set output to ""
    repeat with c in (every chat)
        try
            set msgs to messages of c
            repeat with m in msgs
                set output to output & (text of m) & "\n"
            end repeat
        end try
    end repeat
    return output
end tell
'
```

**Using Beeper CLI (preferred for rich data):**
```bash
beeper-desktop-cli messages search --query "lunch" --sender others --limit 20 --format json
```

## Notes

- **AppleScript limitations:** The Messages AppleScript API doesn't expose all message metadata
  (timestamps, read receipts, etc.). For rich data, use beeper-desktop-cli or direct database access.

- **Permission differences:**
  - Full Disk Access: Grants access to all protected files (broad)
  - Automation permissions: Per-app control over specific apps (granular)

- **Contacts app must be running** for AppleScript to work (it will launch automatically).

- **Message history:** AppleScript only accesses messages cached in the Messages app. Very old
  messages may not be accessible without direct database access.

- **Privacy consideration:** Beeper Desktop API only accesses messages synced to Beeper Desktop, which
  may be a subset of all your messages depending on your configuration.

## References

- [Apple Developer: Messages Scripting](https://developer.apple.com/library/archive/documentation/AppleScript/Conceptual/AppleScriptLangGuide/)
- [macOS Privacy Preferences Policy Control](https://support.apple.com/guide/security/privacy-preferences-policy-control-sec830c83560/web)
- [Beeper Developer Docs](https://developers.beeper.com/desktop-api)
