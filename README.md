---
specfork: 1
name: "@specfork/password-manager"
summary: Offline desktop password manager. One encrypted vault file in the open KDBX 4 format; entries, generator, TOTP, auto-type, auto-lock, clipboard clearing, history, import, export, merge.
license: CC-BY-4.0
derived_from:
  repo: https://github.com/keepassxreboot/keepassxc
  commit: d5e43b779d5e23c23963d53b806dcda5dd38f50e
  url: https://github.com/keepassxreboot/keepassxc
  license: GPL-2.0-only OR GPL-3.0-only
---

# Password manager

One person keeps logins, notes and secrets in a single encrypted vault file on their own computer.
The file uses the open KDBX 4 format, so other compatible apps can open it. The vault is unlocked
with a password, a key file, or both; it locks itself when idle, clears copied secrets from the
clipboard, types logins into other windows, generates passwords and one-time codes, keeps a
history of every change, and merges two copies of the same vault.
Behaviour derived from KeePassXC's open-source code (GPL-2.0 or GPL-3.0), written fresh and
stack-free.
Not affiliated with or endorsed by the KeePassXC Team. The name is used only to credit the source.

| Area | Prefix | Covers |
|---|---|---|
| [vault.md](specs/vault.md) | VLT | KDBX 4 file layout, KDF, ciphers, integrity, saving, reload, vault settings |
| [keys.md](specs/keys.md) | KEY | Composite key: password, key file; unlock, change key, quick unlock |
| [entries.md](specs/entries.md) | ENT | Entries, groups, tags, custom fields, attachments, references, search |
| [history.md](specs/history.md) | HIS | Entry history, recycle bin, deleted objects, merge |
| [generator.md](specs/generator.md) | GEN | Password and passphrase generator, entropy, quality, health report, breach check |
| [totp.md](specs/totp.md) | OTP | Time-based one-time codes |
| [security.md](specs/security.md) | SEC | No telemetry, network policy, memory hygiene, clipboard, auto-lock |
| [autotype.md](specs/autotype.md) | ATY | Typing into the focused window: matching, sequences, timing |
| [transfer.md](specs/transfer.md) | XFR | Import (CSV, other managers' formats), export with warnings |
| [browser.md](specs/browser.md) | BRW | Local IPC contract for browser extensions |

Single user, no account, no server: everything runs on one computer (Windows, macOS, Linux) with no
network unless the user turns on a feature in [SEC-3](specs/security.md).
Shortcuts are defaults and can be changed; `Mod` = Ctrl on Windows and Linux, Cmd on macOS.
Instants are UTC; byte layouts are little-endian unless stated.
