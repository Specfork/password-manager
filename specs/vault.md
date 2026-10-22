# Vault

**Status:** Draft · **Prefix:** VLT

## Purpose

The vault file: its on-disk layout in the open KDBX 4 format, key derivation, encryption, integrity checks, and how it is saved, backed up and reloaded.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| VLT-1 | A vault is one file (default extension `.kdbx`) holding every group, entry, attachment, custom icon and vault setting. No vault content is written anywhere else except attachments the user opens ([SEC-10](security.md)) and exports ([XFR-7](transfer.md)). | create vault, add 3 entries, copy only the file to another computer → all 3 entries open there |
| VLT-2 | File start: `uint32 0x9AA2D903`, `uint32 0xB54BFB67`, `uint32 version` (major in the high 16 bits, minor in the low). Writes 4.0 (`0x00040000`), or 4.1 (`0x00040001`) when the vault uses 4.1-only data: tags, custom icon names or times, previous parent group, entry health-check exclusion. Reads 3.1, 4.0, 4.1. Wrong signatures → "not a vault file"; major version above 4 → "made by a newer app", nothing opened; KDBX 1 files only through import ([XFR-1](transfer.md)). | vault with a tag → saved version 4.1; without → 4.0; random file → "not a vault file"; version 5.0 → refused |
| VLT-3 | Outer header: repeated `{id uint8, length uint32, value}`. Fields: `2` cipher id (16-byte UUID), `3` compression (`uint32`: 0 none, 1 gzip), `4` master seed (32 bytes), `7` encryption IV (16 bytes for AES and Twofish, 12 for ChaCha20), `11` KDF parameters (variant map), `12` public custom data (variant map, optional, never secret), `0` end. A missing required field, unknown id or wrong length → refuse as corrupted. | header without field 4 → "corrupted" |
| VLT-4 | Variant map: `uint16` version `0x0100` (high byte must be `0x01`), then items `{type uint8, name_len uint32, name UTF-8, value_len uint32, value}` until type `0`. Types: `0x04` uint32, `0x05` uint64, `0x08` bool, `0x0C` int32, `0x0D` int64, `0x18` UTF-8 string, `0x42` bytes. | map version `0x0200` → refused |
| VLT-5 | The header bytes are followed by `SHA-256(header)` and `HMAC-SHA-256(header)` keyed with the block key for index `2^64−1` (VLT-8). Hash mismatch → "corrupted"; HMAC mismatch → "invalid credentials" ([KEY-5](keys.md)). | flip one header byte → "corrupted"; wrong password → "invalid credentials" |
| VLT-6 | KDF parameters: `$UUID` (16 bytes) selects the KDF. Argon2 (id `9e298b19-56db-4773-b23d-fc3ec6f0a1e6` = Argon2id, `ef636ddf-8c29-444b-91f7-a9a403e30a0c` = Argon2d) reads `S` salt (32 bytes), `P` parallelism uint32, `M` memory in bytes uint64, `I` iterations uint64, `V` version uint32 = `0x13`, optional `K` secret and `A` associated data. AES-KDF (`c9d9f39a-628a-4460-bf74-0d08c18a4fea`) reads `R` rounds uint64 and `S` seed (32 bytes). All three open; any other → refuse. | vault with Argon2d → opens; unknown KDF id → refused |
| VLT-7 | New vaults use Argon2id: memory 64 MiB, parallelism min(logical CPUs, 4), iterations calibrated so one derivation takes ≈ the target time (default 1000 ms, settable 100..5000 ms), at least 2. Manual limits: memory 8 KiB..4 GiB, parallelism 1..256, iterations 1..2^32−1. Settings warn, without blocking, when Argon2 memory < 16 MiB or estimated unlock < 100 ms ("weak"), or memory > 2 GiB or estimated unlock > 10 s ("may not open on smaller devices"). Changing KDF settings shows the new estimated unlock time and applies on the next save. | new vault on a 2-core machine → `P` = 2, `M` = 67108864; set memory 4 MiB → weak warning shown, save allowed |
| VLT-8 | Keys: `transformed = KDF(composite key)`; `enc_key = SHA-256(master_seed ‖ transformed)`; `hmac_base = SHA-512(master_seed ‖ transformed ‖ 0x01)`; block key `i` = `SHA-512(uint64 i ‖ hmac_base)`. | known test vector vault → derived keys match |
| VLT-9 | Ciphers: AES-256-CBC (`31c1f2e6-bf71-4350-be58-05216afc5aff`), ChaCha20 (`d6038a2b-8b6f-4cb5-a524-339a31dbb59a`), Twofish-256-CBC (`ad68f29f-576f-4bb9-a36a-d47af965346c`); CBC uses PKCS#7 padding. Default AES-256; selectable per vault. | switch to ChaCha20, save, reopen → header cipher id is ChaCha20, IV 12 bytes |
| VLT-10 | Payload after the header: blocks `{hmac 32 bytes, size int32, data}` with `hmac = HMAC-SHA-256(block key i, uint64 i ‖ int32 size ‖ data)`, `i` from 0, data ≤ 1 MiB, ending with an empty block. Any block mismatch → refuse; never show partial data. | flip one byte in block 2 → open refused, nothing shown |
| VLT-11 | The joined blocks are decrypted, then gunzipped when compression = 1 (default on), giving the inner header `{id uint8, length uint32, value}`: `1` inner stream id (`3` = ChaCha20; `2` = Salsa20, read only), `2` inner stream key (64 random bytes), `3` attachment (1 flag byte, bit `0x01` = protected, then content; order gives index 0..n), `0` end. The XML document follows. Protected values (passwords, protected custom fields) in the XML are XORed with the inner stream (ChaCha20, key = first 32 bytes and nonce = next 12 bytes of `SHA-512(inner key)`) in document order, then base64-encoded. | vault with 2 attachments → inner header holds 2 binaries; read password element raw → not the plain value |
| VLT-12 | XML: root `KeePassFile` with `Meta` (VLT-16 settings, custom icons, custom data) and `Root` (one root group, `DeletedObjects` list of `{UUID, DeletionTime}`). UUIDs are base64 of 16 bytes; times are base64 of `int64` seconds since `0001-01-01T00:00:00Z`. Custom data on vault, group and entry is kept unchanged on save; other unknown elements are ignored. | custom data key set by another app → still present after save |
| VLT-13 | Every save makes a new master seed, IV, KDF salt and inner stream key. Saving is atomic: write a temporary file in the same folder, flush to disk, then replace the vault. On failure the old file stays intact, an error is shown and the vault stays marked modified. A setting `direct_write` (default off) writes in place for folders that cannot rename. | save twice with no change → files differ, both open; disk full during save → old file opens, `*` still shown |
| VLT-14 | Backup before save (default off): copy the current file to a path from a pattern (default `{DB_FILENAME}.old.kdbx`; `{TIME:fmt}` inserts the local time) before writing; a failed backup aborts the save. | backup on, save → `.old.kdbx` holds the previous version |
| VLT-15 | Auto-save after every change (default on) and on lock and exit (default on). With auto-save off the tab shows `*`; closing, locking or quitting asks save / discard / cancel. | auto-save off, edit, quit → prompt |
| VLT-16 | Vault settings: `name`, `description`, `default_username`, `color` (tab marker), `recycle_bin_enabled` (default true), `history_max_items` (default 10, −1 = no limit), `history_max_size` (default 6 MiB, −1 = no limit), `compression` (default gzip), cipher and KDF. | new vault → history limits 10 and 6 MiB |
| VLT-17 | External change: the open file is watched. Changed on disk with no unsaved changes → reloaded silently (setting `auto_reload`, default on); with unsaved changes → ask "merge mine into the new file" ([HIS-10](history.md)) or "keep mine and overwrite on next save". Changed while locked → the new file is read on unlock. | edit file from a second app → first app shows the change within 5 s |
| VLT-18 | Several vaults can be open at once, each in its own tab with its own lock state. Recently opened vault paths are remembered (setting, default on) and reopened at start (default on). | open 2 vaults, lock one → other stays unlocked |

## Out of scope

- Writing KDBX 3.1 or older; hardware challenge-response key components.
- Sharing or syncing folders of vaults between people; cloud storage clients.
- The detailed XML element list beyond VLT-12; implementers follow the published KDBX 4 format.

## Decisions

- Argon2id is the default KDF for new vaults (the source defaults to Argon2d): it resists side channels and is the current general recommendation; Argon2d and AES-KDF still open.
- Always write KDBX 4: one write path, and 4.0 stays readable by older compatible apps unless 4.1 data is present.
- KDF warnings never block: the user may need weak settings for a slow phone app that shares the file.
