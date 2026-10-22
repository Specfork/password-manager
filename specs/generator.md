# Generator

**Status:** Draft · **Prefix:** GEN

## Purpose

Generate strong passwords and passphrases, show how strong any password is, and report weak, reused, expiring or breached ones.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| GEN-1 | Two modes, `password` and `passphrase`, available standalone and from any password field; the last-used options are remembered (not the output). The generator keeps no history of outputs. | generate, close, reopen → same options, new value |
| GEN-2 | Password options: length 1..999 (default 32); classes `lower` a–z, `upper` A–Z, `digits` 0–9, `brackets` `()[]{}<>`, `punctuation` `.,:;`, `quotes` `"'` and backtick, `dashes` `-_/\|`, `math` `+*=!?%^~`, `symbols` `#$&@`, `extended` printable U+00A1..U+00FF; default lower + upper + digits. Plus `also_include` (extra characters) and `exclude` (characters removed). | defaults → 32 chars from a–z A–Z 0–9 only |
| GEN-3 | `avoid_lookalikes` (default on) removes `0 O o 1 l I \| 5 S 8 B 6 G` from the pool. | on, 1000 outputs → none contains `0` or `O` |
| GEN-4 | `every_class` (default on): each output holds at least one character from every selected class (and from `also_include` when set). Length below the class count, or an empty pool, disables generating and says why. | 3 classes, length 2 → disabled with reason |
| GEN-5 | Randomness comes from the OS cryptographic generator; each character is picked uniformly from its pool without modulo bias, and the result is shuffled uniformly. | chi-square over 100,000 single-char outputs → uniform at p > 0.01 |
| GEN-6 | Passphrase options: word count 1..40 (default 7); word list = bundled list of 7776 words (default) or a user file with one word per line (blanks and duplicates dropped; under 1000 words → warning); separator any string (default one space); case `lower` (default), `upper`, `title`, `mixed` (each word randomly lower or title). | 7 words, space → 6 spaces |
| GEN-7 | The generator shows entropy in bits: password = length × log2(pool size); passphrase = words × log2(list size). | 32 chars from 56-char pool → ≈ 185.8 bits |
| GEN-8 | Every password field (entry, vault key) shows an estimated entropy for the typed value from a pattern-aware estimator (dictionary words, keyboard runs, repeats, dates, common substitutions), not the naive formula. | `Password1!` → < 40 bits |
| GEN-9 | Quality bands by bits: ≤ 0 `bad`, < 40 `poor`, < 75 `weak`, < 100 `good`, ≥ 100 `excellent`; shown as a coloured meter with the band name. | 80 bits → `good` |
| GEN-10 | Health report per entry (not in the recycle bin, not excluded by its own flag): start from GEN-8 bits; if the same password is in N > 1 entries subtract 15 × (N − 1) and cap at 74; expired → 0; expiring in d ≤ 30 days → cap at 70 then subtract 2 × (30 − d). Entries scoring < 75 are listed lowest first with reasons. | 2 entries share a 120-bit password → both 74, listed as reused |
| GEN-11 | Breach check (opt-in, off): for each password compute uppercase hex `SHA-1`, send only its first 5 characters to the user-configured range endpoint, compare the returned suffixes locally, and show the count. A local hash-list file can be checked instead with no network. Usernames, URLs and full hashes are never sent. | capture traffic → only 5-hex-char prefixes leave the device |

## Out of scope

- Password rotation on websites; storing generator history.

## Decisions

- Two entropy figures: the generator's is exact for its own output; the estimator's is honest about human-chosen text.
- Health penalties follow the source's numbers so reports compare across implementations.
