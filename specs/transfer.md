# Transfer

**Status:** Draft · **Prefix:** XFR

## Purpose

Bring passwords in from CSV files and other managers' export formats, and take them out, with clear warnings when the output is not encrypted.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| XFR-1 | Import sources: CSV; KDBX 1 (`.kdb`, needs its key); unencrypted or password-encrypted JSON item exports (folders, items with login, card, identity and note types); `1pux` zip archives; `opvault` folders. The target is a new vault (asks for location and composite key) or a new group in the open vault. | import JSON into open vault → new group with all items |
| XFR-2 | CSV import: preview of the first 20 rows; choose encoding (UTF-8 default; legacy code pages), delimiter (auto-detected among `,` `;` tab `\|`), quote `"`, comment prefix `#`, and whether row 1 is a header. Each column maps to `group`, `title`, `username`, `password`, `url`, `notes`, `otp`, `icon`, `modified`, `created`, a custom field named after its header, or ignore. | `;`-delimited file → detected, preview correct |
| XFR-3 | CSV values: `group` is a `/`-separated path (created as needed; a leading root name is dropped); times accept ISO 8601 or Unix seconds (invalid → import time). Bad rows are listed with row numbers and skipped; a summary gives counts imported and skipped. | row with 3 cols in a 5-col file → reported, others imported |
| XFR-4 | JSON and archive imports: folders → groups; login items → entries (first URI → `url`, further URIs → custom fields `url_2`, `url_3`…), hidden fields → protected custom fields, one-time code → `otp` ([OTP-1](totp.md)), other item types → entries with their fields as custom fields; attachments imported when present; tags kept. | item with 3 URIs → `url`, `url_2`, `url_3` |
| XFR-5 | Export formats: KDBX (save a copy with the same or a new key), CSV, HTML (printable tables per group), XML (the decrypted vault document). | export each → 4 files open in their viewers |
| XFR-6 | Before any unencrypted export the user confirms a warning that the file holds every password in plain text and anyone who gets it can read them; there is no "don't ask again". | export CSV → warning every time |
| XFR-7 | Export files are created readable only by the current user where the OS supports it, at a path the user chose; nothing is exported implicitly. | export CSV → file mode owner-only |
| XFR-8 | CSV export: UTF-8 without BOM, RFC 4180 quoting, `\n` line ends, header `Group, Title, Username, Password, URL, Notes, TOTP, Icon, Last Modified, Created` (ISO 8601 UTC); group = path below the root; recycle bin and history excluded; a note says custom fields and attachments are not included. | entry in `Root/Work/Mail` → Group `Work/Mail` |
| XFR-9 | XML export holds everything, history and recycle bin included, with protected values in plain text and attachments inline as base64. Re-importing it into a new vault gives the same content. | export XML, import → entry count and fields equal |

## Out of scope

- Importing from browsers' built-in password stores directly; scheduled or automatic exports.
- Product-specific formats beyond the generic ones in XFR-1.

## Decisions

- Import never writes into existing entries: new vault or new group, so a bad import is one delete away.
- `@specfork/privacy` is not required: its export must exclude secrets, and here the secrets are the data; local export lives in this spec.
