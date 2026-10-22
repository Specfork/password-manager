# History

**Status:** Draft · **Prefix:** HIS

## Purpose

Nothing is lost by accident: every change keeps the previous version of an entry, deletions go to a recycle bin, and two copies of a vault can be merged.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| HIS-1 | Applying a change to an entry first stores a snapshot of its previous state (all fields, tags, custom fields, attachment references, icon, colours, times; not its own history) at the end of the entry's history. No snapshot when no stored field changed, or only `accessed` changed. | edit password → history has 1 snapshot with the old password; open entry, copy password → history unchanged |
| HIS-2 | After each snapshot, the oldest are dropped until count ≤ `history_max_items` and total size ≤ `history_max_size` ([VLT-16](vault.md)); size counts snapshot text plus attachment bytes not also used by the current version. −1 means no limit for that rule. | limit 10, 11th edit → 10 snapshots, oldest gone |
| HIS-3 | History view lists snapshots newest first with their `modified` time; a snapshot can be viewed read-only, restored, or deleted; "delete all history" asks to confirm. Restore makes the snapshot current and stores the replaced state as a new snapshot. | restore 3rd snapshot → current = it, history count +1 |
| HIS-4 | Recycle bin (when `recycle_bin_enabled`): deleting an entry or group outside it moves it into a "Recycle Bin" group created on first use, identified by uuid in vault settings (not by name), and sets `location_changed`; no confirmation by default (setting). | delete entry → it appears in the bin |
| HIS-5 | Deleting inside the bin, with the bin disabled, or with `Shift+Del` deletes permanently after confirmation that names the item and count; each removed uuid is added to the vault's deleted-objects list with the time. "Empty recycle bin" asks to confirm with the number of items, then deletes them permanently (HIS-5). | delete inside bin → gone, deleted-objects has its uuid; bin with 4 items → confirm "4" → bin empty |
| HIS-6 | Restore from the bin moves the item back to its previous parent group if it still exists, else to the root. | delete entry from group G, restore → back in G |
| HIS-7 | The bin's contents are excluded from search, auto-type, browser IPC, health report and expiry lists. | entry in bin matches search text → not shown |
| HIS-8 | Merge: choose another vault file and its key; its content is merged into the open vault, which is then saved per [VLT-15](vault.md). A preview lists every change (added, updated, moved, deleted, metadata) before the user confirms. Merging a vault with itself or an identical copy reports "no changes" and does not save. | merge → preview shows 2 added, 1 updated; cancel → no change; merge same file → "no changes" |
| HIS-9 | Matching is by uuid. An item only in the other vault is added under its parent group (by uuid; missing parents are recreated along its path), unless this vault's deleted-objects list has it with a time later than its `modified`. | entry new in B → added to A in the same group |
| HIS-10 | An entry in both with different content: the one with the later `modified` becomes current; the other's current state joins history; both histories are combined by `modified` time with equal snapshots removed, then trimmed (HIS-2). Equal `modified` and content → nothing. | A edited Monday, B edited Tuesday → result = B's version, A's in history |
| HIS-11 | Location: the copy with the later `location_changed` decides the parent group. Groups merge like entries using their own `modified` and `location_changed`. | B moved entry later → moved in A |
| HIS-12 | Deletions: an item deleted in either vault is removed from the result when its deletion time is later than its `modified` in the other; otherwise it is kept and the deletion record dropped. Deleted-objects lists are combined. | B deleted X after A last edited X → X removed |
| HIS-13 | Vault settings merge: custom icons combined by uuid; name, description and other settings taken from the vault whose settings changed later; the recycle bin uuid of the open vault is kept. | B renamed vault later → name from B |

## Out of scope

- Three-way merge with a common ancestor; per-field conflict resolution by the user.
- Restoring permanently deleted items.

## Decisions

- Last-modified-wins with the loser kept in history: simple, deterministic, and never loses data that history limits allow.
- Merge always previews first; the source can merge without preview.
