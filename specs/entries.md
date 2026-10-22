# Entries

**Status:** Draft · **Prefix:** ENT

## Purpose

What a vault holds: entries in a tree of groups, with tags, custom fields, attachments, references and expiry, and how to find them.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| ENT-1 | Entry: `uuid` (16 random bytes), `title`, `username`, `password`, `url`, `notes` (multi-line), `tags` (set), `icon` (built-in index or custom icon uuid), `fg_color`, `bg_color`, `expires` (bool) + `expiry_time`, `created`, `modified`, `accessed`, `location_changed`, custom fields, attachments, auto-type settings ([ATY-3](autotype.md)), history ([HIS-1](history.md)). Text is UTF-8 with no length limit. | new entry → all times equal now, `expires` false |
| ENT-2 | `password` is always protected (masked in the UI, protected in the file per [VLT-11](vault.md)). Per-vault flags may also protect `title`, `username`, `url`, `notes` (default off). New entries get the vault's `default_username`. | read file raw → password not in clear text |
| ENT-3 | Custom field: non-empty name, unique within the entry (case-sensitive), not one of the standard names `Title`, `UserName`, `Password`, `URL`, `Notes`; text value; `protected` flag (default off) masks it and protects it in the file. | add field `Password` → refused |
| ENT-4 | Attachment: name unique within the entry (adding a clash asks replace or rename to `name (1)`), bytes. Identical contents are stored once per vault. Adding a file over 10 MiB asks for confirmation. | same 1 MB file on 2 entries → vault grows by ≈ 1 MB once |
| ENT-5 | Groups form a tree under one root. Group: `uuid`, `name`, `notes`, `icon`, `expires`/`expiry_time`, `default_autotype_sequence` (inherit), `autotype_enabled` and `search_enabled` (`inherit`, `on`, `off`; root inherits = on). Every entry is in exactly one group. Entries and groups move by drag or cut/paste; a group cannot move into itself or a descendant. Moving sets `location_changed`. | group search `off` → its entries absent from search; drag group A into its child → refused |
| ENT-6 | Tags: trimmed, non-empty, no `,` or `;`; duplicates within an entry dropped ignoring case. A tag list shows every tag with its entry count; choosing a tag filters the list. Renaming or deleting a tag there changes every entry that has it. | rename tag `work` → `job` on 5 entries → 5 entries changed, each with history |
| ENT-7 | Expiry: an entry with `expires` and `expiry_time` ≤ now is expired: shown struck through, excluded from auto-type and browser results unless a setting allows. On unlock, entries expired or expiring within N days (default 3, setting, on by default) are listed. | entry expiring tomorrow → listed on unlock |
| ENT-8 | Field reference `{REF:<w>@<s>:<text>}`: `w` ∈ `T U P A N I` (title, username, password, url, notes, uuid) to return; `s` ∈ `T U P A N I O` (O = any custom field) to search; resolves to the first matching entry outside the recycle bin. Nesting up to 10 levels; deeper or circular references stay as literal text. | `{REF:P@I:<uuid of B>}` in A's password → copying A's password gives B's |
| ENT-9 | Placeholders `{TITLE}`, `{USERNAME}`, `{PASSWORD}`, `{URL}`, `{NOTES}`, `{S:<field name>}` and references are resolved when copying, opening a URL, auto-typing and answering the browser IPC; the stored value stays unresolved. | url `https://x.test/?u={USERNAME}` opens with the username filled in |
| ENT-10 | Clone: new uuid, title suffix ` - Clone` (optional), options: replace username and password with references to the original, include history (default off). | clone with references → change original password → clone's resolves to new one |
| ENT-11 | Open URL uses the system browser. A `cmd://` URL runs a local command only after a confirmation that shows the command, each time unless the user ticks "remember for this entry". | open `cmd://notepad` → confirmation first |
| ENT-12 | Default shortcuts: new entry `Mod+N`, edit `Enter`, delete `Del`, copy username `Mod+B`, copy password `Mod+C`, copy URL `Mod+U`, copy one-time code `Mod+T`, open URL `Mod+Shift+U`, auto-type `Mod+Shift+V`, search `Mod+F`, lock `Mod+L`, save `Mod+S`. | press `Mod+C` on an entry → password on clipboard |
| ENT-13 | Edit form: changes apply on OK or Apply; Cancel with changes asks to discard. Applying with no field changed does not update `modified` nor add history. | open, OK without change → `modified` unchanged |
| ENT-14 | Search runs as the user types (or on Enter, setting) over all groups (default) or the current group; excludes the recycle bin and groups with search off; `Esc` clears it. Search syntax: space-separated terms, all must match; a bare term matches a substring, ignoring case, of title, username, url, notes, tags or custom field values (not passwords). Prefixes: `title:`, `user:`, `pass:`, `url:`, `notes:`, `field:<name>=`, `attach:`, `group:`, `tag:`, `uuid:`, `is:expired`, `is:expired-<days>`, `is:weak`, `has:otp`. Modifiers: `-` excludes, `+` matches the whole value exactly, `*` treats the word as a regular expression; `"..."` keeps spaces. A case-sensitive toggle applies to all terms. | type `bank` → matching entries within 300 ms; `tag:work -url:intranet` → work-tagged entries whose url lacks "intranet" |

## Out of scope

- SSH agent integration, passkeys, secret-service style OS APIs, group sharing between vaults.
- Printing.

## Decisions

- Search syntax is the app's own (inspired by the source): prefixes are short English words; implementations may add aliases.
- Tag rename edits every entry and so creates history on each: tags live on entries, not as separate records.
