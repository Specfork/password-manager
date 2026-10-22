# One-time codes

**Status:** Draft · **Prefix:** OTP

## Purpose

Entries can hold a time-based one-time code secret and show the current code, compatible with standard authenticator setups.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| OTP-1 | Stored in the entry's protected custom field `otp` as an `otpauth://totp/<label>?secret=<base32>&period=<s>&digits=<n>&algorithm=<SHA1\|SHA256\|SHA512>&issuer=<text>` URI; defaults period 30, digits 6, SHA1 (omitted from the URI when default). Optional `encoder=steam`. | set up with defaults → URI has only `secret` and `issuer` |
| OTP-2 | Setup accepts a pasted URI or a Base32 secret plus period (1..86400 s), digits (6..10) and algorithm. Base32 ignores case, spaces and `=`; invalid characters or an empty secret → refused with a message. | secret `jbsw y3dp ehpk 3pxp` → accepted |
| OTP-3 | Older storage read on open and kept as found: a `TOTP Seed` field with a `TOTP Settings` field `<period>;<digits>` (or `<period>;S` for the Steam encoder), and `TimeOtp-Secret-Base32` with `TimeOtp-Length`, `TimeOtp-Period`, `TimeOtp-Algorithm` fields. | entry with `TOTP Settings` `60;8` → 8-digit code every 60 s |
| OTP-4 | Code (RFC 6238): `T = floor(unix_seconds / period)` as 8-byte big-endian, HMAC with the algorithm and secret, dynamic truncation, modulo 10^digits, left-padded with zeros. Uses the system clock; never network time. | RFC 6238 test vectors → all match |
| OTP-5 | Steam encoder: 5 characters, each = alphabet `23456789BCDFGHJKMNPQRTVWXY`[v mod 26] with `v` divided by 26 after each step, from the truncated 31-bit value. | known Steam vector → expected 5 chars |
| OTP-6 | Display: the current code and seconds left with a countdown bar, refreshed at each period boundary; entries with a code show a marker in the list; `Mod+T` copies the code ([SEC-7](security.md) clearing applies); `{TOTP}` in auto-type and IPC gives the current code. | copy at 29 s left → code valid for that window |
| OTP-7 | The setup URI can be shown as a QR code rendered locally to move the secret to another authenticator; the secret is masked until revealed. | show QR → scanning with an authenticator gives the same codes |

## Out of scope

- Counter-based codes (HOTP), push approvals, hardware tokens.

## Decisions

- One field holds the whole URI: the standard format, readable by other apps sharing the file; older field pairs are read but never written.
