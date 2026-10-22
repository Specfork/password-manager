# Security

**Status:** Draft · **Prefix:** SEC

## Purpose

Rules that keep secrets on the device and out of sight: no telemetry, no network by default, memory hygiene, clipboard clearing and automatic locking.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| SEC-1 | No telemetry, analytics, usage pings or crash uploads, ever, in any setting. | capture traffic during a full session with defaults → zero outbound connections |
| SEC-2 | No network by default: a fresh install makes no outbound connection in any workflow. Only the features in SEC-3 may connect, each only while its own setting is on. | every SEC-3 setting off → zero connections while using every feature |
| SEC-3 | Opt-in network features, each off by default: `icon_download` (fetch a site icon for one entry on the user's request: GET the URL host's icon over HTTPS, no cookies, 10 s timeout, no third-party icon service unless a separate setting is on); `breach_check` ([GEN-11](generator.md)); `update_check` (asked once at first start; sends only a version request). | icon download off → icon button hidden |
| SEC-4 | Memory hygiene (goal, verified by review): passwords, protected fields, keys and derived keys live in buffers that are overwritten with zeros before release, locked against swapping where the OS allows, and kept XORed with a per-session random key while not in use. | manual: review secret buffer handling; memory dump after lock → no known test password |
| SEC-5 | Process hardening where the OS allows: core and crash dumps of process memory off; other same-user processes cannot attach a debugger or read process memory. | manual: attach debugger as same user → refused on supporting OS |
| SEC-6 | Logs, error messages and the settings file never contain passwords, field values, one-time codes, key file contents or key material. Settings hold only preferences and, when allowed, recent vault and key file paths. | grep settings and logs for a test password → none |
| SEC-7 | Copying a secret starts a clear timer (setting `clear_clipboard`, default on, 10 s, range 1..999 s). When it ends the clipboard is cleared only if it still holds the copied value. Locking and quitting also clear it. Copies are flagged to the OS as sensitive so clipboard history and cloud sync skip them where supported. After copying, the window minimises (default), hides, or stays (setting). | copy password, wait 10 s → clipboard empty; copy, then copy other text → that text kept; default, copy username → window minimised |
| SEC-8 | Auto-lock: after `idle_lock_seconds` with no input to the app (setting on, default 900 s, range 10..86400), when the OS screen locks (default on), when the OS user session switches (default on), on minimise (default off), after auto-type (default off). | idle 900 s → vault locked |
| SEC-9 | Lock discards all decrypted data and keys from memory, closes open dialogs of that vault and shows only the file name and unlock form. Unsaved changes: saved first when auto-save is on; otherwise the user is asked and an idle lock waits rather than discard. | auto-save off, edit, idle → prompt, data not lost |
| SEC-10 | Opening an attachment writes it to a temporary file readable only by the current user; it is deleted on close of the viewer, lock or exit, and any left over are deleted at next start. No other decrypted temporary files are written. | open attachment, lock → temp file gone |
| SEC-11 | Passwords are masked with a fixed count of dots that hides their length, in lists, preview and edit form (setting, default masked); each field has a reveal toggle. Notes can be masked too (default off). | 4-char and 40-char passwords → same mask |
| SEC-12 | Window content is excluded from screen capture and screen sharing where the OS supports it (setting `allow_screen_capture`, default off). Only one app instance runs per user; a second launch passes its file argument to the first over a per-user local channel and exits. | screenshot tool → window blank; launch twice → one window |

## Out of scope

- Protection against malware already running as the user or as an administrator.
- Secure input against keyloggers.

## Decisions

- Network is a short opt-in list instead of a general toggle: each connection has one stated purpose and payload.
- Memory hygiene is a stated goal checked by review: managed runtimes cannot always prove zeroing, but must try.
