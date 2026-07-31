---
name: swift-macos-native-apis
description: |
  Use compiled Swift binaries instead of AppleScript for macOS system integrations.
  Use when: (1) an AppleScript is slow (>2s) for Contacts, Calendar, Reminders, or other
  system data, (2) need to read/write macOS Contacts, Calendar, Reminders, Focus status,
  or Notifications programmatically, (3) osascript is timing out or iterating slowly,
  (4) building a new macOS system integration for an AI agent or CLI tool. Swift + native
  frameworks (CNContactStore, EventKit, UNUserNotificationCenter) are ~100x faster than
  AppleScript equivalents. Compile once with swiftc, call from Python/Node/Bash.
author: Claude Code
user-invocable: false
---

# Swift for macOS Native APIs

## Problem

AppleScript is the default way to access macOS system data (Contacts, Calendar, Messages)
from scripts and AI agents. But it's painfully slow — iterating through contacts takes
10-30+ seconds via AppleScript, while the same operation takes <1s via Swift's CNContactStore.

## Context / Trigger Conditions

- An `osascript` command takes >2 seconds
- AppleScript iterates through collections with `repeat with` loops
- Need to access Contacts, Calendar, Reminders, or Notifications
- Building a CLI tool or wrapper that needs macOS system data
- An existing AppleScript is timing out

## Solution

### Pattern: Compiled Swift Binary with JSON I/O

1. Write a Swift script using the native framework
2. Compile with `swiftc -O script.swift -o binary`
3. Binary accepts args, outputs JSON to stdout
4. Call from Python/Node/Bash, parse JSON

### Example: Contact Lookup (the pattern that proved this)

**Before (AppleScript — 10s per lookup):**
```applescript
tell application "Contacts"
  repeat with p in people        -- iterates ALL contacts
    repeat with ph in phones of p  -- nested loop
      -- string comparison
    end repeat
  end repeat
end tell
```

**After (Swift — 0.8s for batch lookup):**
```swift
import Contacts
import Foundation

let store = CNContactStore()
let keys = [CNContactGivenNameKey, CNContactFamilyNameKey,
            CNContactPhoneNumbersKey] as [CNKeyDescriptor]
let request = CNContactFetchRequest(keysToFetch: keys)

var result: [String: String] = [:]
try store.enumerateContacts(with: request) { contact, _ in
    let name = "\(contact.givenName) \(contact.familyName)"
        .trimmingCharacters(in: .whitespaces)
    guard !name.isEmpty else { return }
    for phone in contact.phoneNumbers {
        result[phone.value.stringValue] = name
    }
}

let data = try JSONSerialization.data(withJSONObject: result)
print(String(data: data, encoding: .utf8)!)
```

**Compile and use:**
```bash
swiftc -O lookup-contacts.swift -o lookup-contacts
./lookup-contacts "+1 555-123-4567" "+1 555-234-5678"
# → {"+1 555-123-4567":"Jane Doe","+1 555-234-5678":"John Smith"}
```

### Available Native Frameworks

| Framework | What it accesses | Key classes | AppleScript equivalent speed |
|-----------|-----------------|-------------|------------------------------|
| **Contacts** | Contacts.app | `CNContactStore`, `CNContactFetchRequest` | ~100x faster |
| **EventKit** | Calendar + Reminders | `EKEventStore`, `EKEvent`, `EKReminder` | ~50x faster |
| **UserNotifications** | Notification Center | `UNUserNotificationCenter` | No AppleScript equivalent |
| **CoreLocation** | Location services | `CLLocationManager` | No AppleScript equivalent |
| **ScreenTime** | Screen Time data | `STWebpageController` | No AppleScript equivalent |

### When AppleScript is Still Better

- **Writing to Contacts** — `CNContactStore` *reads* fine from a plain `swiftc` binary, but
  `CNSaveRequest` *writes* fail with `NSCocoaErrorDomain Code=4097` (the XPC store refuses a
  read-write connection to an unsigned, unentitled binary). It retries ~8x over 60s, so it
  looks like a hang. Use AppleScript against Contacts.app for the write, and keep Swift for
  the lookup. See the `macos-messages-contacts-access` skill and `scripts/save-contact` in
  the personal_assistant_claude repo.
- **Browser tab control** — No Swift framework for cross-browser tab enumeration
- **UI automation** — System Events / Accessibility API (though Swift can do this too via AX)
- **One-off quick scripts** — If it runs once and speed doesn't matter
- **App-specific scripting** — Some apps only expose AppleScript dictionaries

## Verification

```bash
# Compare speeds
time osascript -e 'tell application "Contacts" to get name of every person'
time ./lookup-contacts "+1 555-123-4567"
```

The Swift binary should be 10-100x faster.

## Notes

- First run may prompt for Contacts/Calendar permission — grant it once
- Compiled binaries are architecture-specific (arm64 on Apple Silicon)
- Add compiled binaries to `.gitignore`, commit only `.swift` source
- The binary needs to be recompiled if the Swift source changes
- JSON I/O makes it easy to integrate with any language (Python, Node, Bash)
- Phone number normalization is important — strip `+1`, spaces, dashes for matching

## References

- [CNContactStore docs](https://developer.apple.com/documentation/contacts/cncontactstore)
- [EventKit docs](https://developer.apple.com/documentation/eventkit)
- [UNUserNotificationCenter docs](https://developer.apple.com/documentation/usernotifications)
- Proven in production: `scripts/lookup-contacts` in personal_assistant_claude repo
