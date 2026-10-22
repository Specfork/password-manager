# Auto-type

**Status:** Draft · **Prefix:** ATY

## Purpose

Type an entry's login into whatever window has focus, so secrets reach other apps without the clipboard.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| ATY-1 | Auto-type sends synthetic key input to the window that has focus, never through the clipboard; characters without a key on the current layout are sent as Unicode input. Where the OS gives no input permission, the feature explains what to grant (or that it is unavailable) and does nothing. | type `ä€` into a text editor with a US layout → both appear |
| ATY-2 | Triggers: a global hotkey (none by default; a taken hotkey is reported), "auto-type selected entry" `Mod+Shift+V` (the app hides and types into the window focused before it), and the IPC request [BRW-8](browser.md). A locked vault prompts to unlock first. | press hotkey on a locked vault → unlock prompt, then typing |
| ATY-3 | Per entry: `autotype_enabled` (inherit from group), `sequence` (empty = inherit), and window associations `[{window pattern, sequence?}]`. Expired entries and the recycle bin are skipped unless a setting allows expired. | entry auto-type off → never matched |
| ATY-4 | Global hotkey matching over all unlocked vaults: an association pattern matches the focused window title as a whole, ignoring case, with `*` = any characters, or as a regular expression when written `//re//`; optionally (both default on) a title containing the entry title, or the entry URL's host, also matches. | pattern `*Mail - Browser` vs title `Inbox - Mail - Browser` → match |
| ATY-5 | Results: one match → typed at once unless "always ask" (default on) shows the chooser; none or several → chooser with search over all entries. The chooser preselects the entry chosen for the same window within the last 15 s. | 2 matches → chooser shows both |
| ATY-6 | Sequence: the first defined of association sequence, entry sequence, nearest group default, vault default `{USERNAME}{TAB}{PASSWORD}{ENTER}`. | entry with no sequence → default typed |
| ATY-7 | Tokens: fields `{TITLE}` `{USERNAME}` `{PASSWORD}` `{URL}` `{NOTES}` `{TOTP}` `{S:<field>}` and references ([ENT-8](entries.md)); keys `{TAB}` `{ENTER}` `{SPACE}` `{BACKSPACE}` `{DELETE}` `{INSERT}` `{HOME}` `{END}` `{PGUP}` `{PGDN}` `{UP}` `{DOWN}` `{LEFT}` `{RIGHT}` `{ESC}` `{F1}`..`{F16}` and literal `{+}` `{^}` `{%}` `{~}` `{#}` `{{}` `{}}`; `{KEY n}` repeats a key n times (n ≤ 100); prefixes `+` Shift, `^` Ctrl, `%` Alt, `#` Super apply to the next key. | `{TAB 3}` → three tabs |
| ATY-8 | Control tokens: `{DELAY n}` pauses n ms (≤ 10000), `{DELAY=n}` sets the gap between keys, `{CLEARFIELD}` selects all and deletes. An unknown token or bad number stops before typing anything and names the token. | `{TABB}` → error naming `{TABB}`, nothing typed |
| ATY-9 | Timing: start delay 500 ms (setting 0..10000), gap between keys 25 ms (setting 0..1000). Typing stops at once if the focused window changes. | switch window mid-sequence → remaining keys not sent |

## Out of scope

- Two-channel obfuscation tricks; per-application keyboard layouts.

## Decisions

- One generic rule ("type into the focused window") instead of per-OS behaviour: each OS supplies its input method; the matching and sequence rules are the same everywhere.
