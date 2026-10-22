# Keys

**Status:** Draft · **Prefix:** KEY

## Purpose

The composite key that unlocks a vault: a password, a key file, or both; unlocking, changing the key, and optional quick unlock.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| KEY-1 | A composite key has a password, a key file, or both; at least one is required. A blank password is allowed only after an explicit "use no password" confirmation and is then omitted, which differs from an empty-string password. | create vault with key file only → opens with key file alone |
| KEY-2 | Password component = `SHA-256(UTF-8 bytes of the password)`, with no trimming or normalisation. Key file component = 32 bytes (KEY-3). Composite key = `SHA-256(password component ‖ key file component)`, omitting absent components, password first. | password `pässword` typed decomposed vs precomposed → different keys |
| KEY-3 | Key file reading, first match wins: XML version 2.0 (hex data plus a 4-byte hash checked against the first 4 bytes of `SHA-256(data)`; mismatch → refuse); XML version 1.0 (base64 of 32 bytes); exactly 32 bytes → used as is; exactly 64 hex characters → decoded; any other non-empty file → `SHA-256(file)`. Empty file → refuse. | 64-hex-char file → its 32 decoded bytes; 1 KB photo → its SHA-256 |
| KEY-4 | "Generate key file" writes 32 random bytes as XML version 2.0 with its hash. Using a legacy key file kind (XML 1.0, raw, hex, hashed) works but shows a suggestion to replace it. The vault file itself can never be its own key file. The app never modifies a key file. Creating a vault with a key file warns that losing the file means losing the vault. | choose the vault as key file → refused; open vault with key file → key file mtime unchanged |
| KEY-5 | Wrong key → "invalid credentials" only, never which component was wrong; the password field is cleared and refocused. There is no attempt limit; the KDF cost is the brake. | wrong password, right key file → same message as both wrong |
| KEY-6 | The unlock screen remembers the last key file path per vault (setting `remember_key_files`, default on) and never the password. Off → stored paths are deleted. | turn off → reopen app → key file field empty |
| KEY-7 | Change key: while unlocked, set a new password (typed twice, shown matching or not) and/or key file; takes effect on the next save. Earlier backups and copies keep the old key; a notice says so. | change password, save, reopen → only new password works |
| KEY-8 | New and changed vault passwords show estimated entropy and quality ([GEN-9](generator.md)); a setting `min_vault_password_quality` (`none` default, `poor`, `weak`, `good`, `excellent`) blocks weaker ones. | minimum `good`, enter `summer2024` → refused |
| KEY-9 | The password may be supplied on standard input when opening from a command line, never as a command-line argument. | pass password as argument → refused with usage message |
| KEY-10 | Quick unlock (optional, offered only where the OS provides biometric or device-PIN protected key storage, default on there): after a full unlock, a locked vault reopens with that OS prompt. The stored secret is removed on app exit, vault close, key change, a failed OS prompt 3 times, or turning it off. | quick unlock, exit app, restart → full key required |

## Out of scope

- Hardware challenge-response tokens and smart cards as key components.
- Key escrow or recovery; a lost key cannot be recovered.

## Decisions

- No unlock attempt limit: the file can be copied and attacked offline anyway, so only the KDF cost protects it.
- Quick unlock never survives an app restart: the OS store holds a secret only for the running session.
