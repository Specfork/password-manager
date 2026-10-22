# Browser IPC

**Status:** Draft · **Prefix:** BRW

## Purpose

A local, encrypted message contract that lets a browser extension (or any local client) ask the app for matching logins, with the user approving every client and site.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| BRW-1 | Off by default. When on, the app listens on a per-user local endpoint only (a Unix domain socket in the user's runtime folder with owner-only permissions, or a named pipe restricted to the current user); never a network port. A relay helper bridges the browser's native messaging to it. | setting on → no TCP listener; another OS user cannot connect |
| BRW-2 | Messages are UTF-8 JSON objects of at most 1 MiB: `{action, client_id, nonce, message}` where `message` is the base64 ciphertext of an inner JSON object; replies echo `action` and carry `nonce` = request nonce + 1 (little-endian increment). Key exchange `change_public_keys {public_key, nonce, client_id}` (unencrypted): the app answers with a fresh X25519 public key for that connection. All later messages use X25519 + XSalsa20-Poly1305 boxes with 24-byte nonces; a reused nonce or failed decryption → error `cannot_decrypt`. | reply nonce ≠ request + 1 → client rejects; replay a captured request → `cannot_decrypt` |
| BRW-3 | Pairing `associate {key, id_key}` asks the user in the app to accept and name the client; accepted → `{id, hash}` stored in the vault's custom data with created and last-used times. `test_associate {id, key}` must succeed per vault before other requests; unknown → `not_associated`. The user can list and remove paired clients per vault. | remove pairing → next request `not_associated` |
| BRW-4 | Actions after pairing: `get_logins`, `set_login`, `generate_password`, `get_totp {uuid}`, `get_vault_groups`, `create_group {path}`, `lock_vault`, `request_autotype {search}`, `get_vault_hash` (SHA-256 of the root group uuid, so clients can tell vaults apart). Unknown action → `unknown_action`. | `lock_vault` → vault locks |
| BRW-5 | `get_logins {url, submit_url?, keys:[{id,key}]}` returns entries whose `url` or `url_*` fields match: same host or a subdomain of it, same port, and same scheme (setting, default on); recycle bin, expired (setting) and entries marked hidden from clients excluded. Sorted best first: exact URL, then path prefix, then host, then subdomain; setting "best match only" (default off). | site `https://a.b.test/login`, entry `https://b.test` → returned |
| BRW-6 | The first request from a site for an entry not yet decided shows the user the entries and asks allow or deny, with "remember"; remembered decisions are stored per entry as allowed and denied hosts. Setting "always allow" (default off). | deny + remember → later requests omit that entry silently |
| BRW-7 | A returned login: `{uuid, title, username, password, otp?, fields}` where `fields` holds only custom fields the user marked for clients. Placeholders and references are resolved ([ENT-9](entries.md)). | entry with unmarked custom field → not in reply |
| BRW-8 | `set_login {url, username, password, group?, uuid?}`: without uuid creates an entry in the given or default client group; with uuid updates it after a confirmation (setting "always allow updates", default off), keeping history. `request_autotype` runs [ATY-2](autotype.md) with the site host as search after user confirmation. | update existing → confirmation shown, history +1 |
| BRW-9 | Locked vault: when "unlock on request" (default on) the unlock form appears and the request waits up to 60 s; otherwise or on timeout → `vault_locked`. The app pushes `vault_locked` and `vault_unlocked` events to connected clients. | lock vault → connected client receives `vault_locked` |
| BRW-10 | Error reply: `{action, error, error_code}`, codes: 1 `vault_locked`, 2 `not_associated`, 3 `cannot_decrypt`, 4 `no_logins_found`, 5 `unknown_action`, 6 `denied`, 7 `bad_request`, 8 `too_large`. Messages and their contents are never logged. | 2 MiB message → `too_large` |

## Out of scope

- The browser extension itself, form detection and filling.
- Installing native messaging manifests per browser; passkeys.

## Decisions

- Action and error names are this spec's own; a client written against them works with any implementation of it.
- Pairing lives in the vault, so moving the file keeps paired clients; removing a vault's pairing never affects another.
