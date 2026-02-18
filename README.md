# passgen

Deterministic password/key generator for terminal use.

`passgen` derives a stable secret from:

- `site` (prompted)
- `username` (prompted)
- `master password` (hidden prompt)
- `PASSGEN_PEPPER` (required env var)

It is designed so you can safely pipe the generated output (stdout) into other commands while keeping prompts on your terminal (`/dev/tty`).

## How It Works

`passgen` runs an embedded Python program that:

1. Builds `salt = (site + username)` (UTF-8).
2. Builds `key = (master password + PASSGEN_PEPPER)` (UTF-8).
3. Runs `PBKDF2-HMAC-SHA256` with:
   - iterations: `100000`
   - derived key length: `64` bytes
4. Encodes the derived bytes as either:
   - Base64 (unpadded; `=` removed), or
   - Z85 (ZeroMQ Base85 alphabet)
5. Optionally truncates to the requested length.

Important: `passgen` writes the result with **no trailing newline**.

## Requirements

- `bash`
- `python3`

## Installation

This repo is a single executable script named `passgen`.

Option A: run it from the repo:

```bash
./passgen
```

Option B: put it on your `PATH`:

```bash
install -m 0755 ./passgen ~/.local/bin/passgen
```

## Setup: PASSGEN_PEPPER

`PASSGEN_PEPPER` is required. Without it, `passgen` exits with an error.

The pepper should be:

- long (at least 16+ bytes; longer is fine)
- high entropy (random)
- stable (if you change it, all derived passwords/keys change)
- treated like a secret

Example (generate a pepper once, store it somewhere secure, then export it):

```bash
export PASSGEN_PEPPER="$(python3 - <<'PY'
import secrets
print(secrets.token_urlsafe(48))
PY
)"
```

Avoid hard-coding the pepper into shell history or world-readable dotfiles.

## Usage

Interactive usage:

```bash
passgen
```

You will be prompted for:

- `Site`
- `Username`
- `Master Password` (hidden)
- `Encoding` (`64` for Base64, `85` for Z85)
- `Length` (blank for full output; otherwise truncate)

Because prompts use `/dev/tty`, stdout is clean for piping:

```bash
passgen | wl-copy
```

If you want a newline for human-readable output:

```bash
passgen; echo
```

## Output Lengths (Full)

With the default 64-byte derived key:

- Base64 unpadded length: 86 characters
- Z85 length: 80 characters

The script suggests common truncation lengths at the prompt:

- Z85: 44 or 80
- Base64: 20 or 86

## Encoding Notes

- Base64 output uses the standard Base64 alphabet (`A-Z a-z 0-9 + /`).
- Z85 output uses the ZeroMQ Z85 alphabet, which includes punctuation; it can be more acceptable in some systems because it avoids `+` and `/` but includes other symbols.

If a target system has strict password character rules, choose the encoding/length that fits, or truncate accordingly.

## Security Model and Caveats

- Deterministic: the same inputs produce the same output.
- If any of `site`, `username`, `master password`, or `PASSGEN_PEPPER` change, the output changes.
- `PASSGEN_PEPPER` acts as an extra secret factor (separate from the master password). Treat it like a key.
- Truncation reduces strength. Prefer using full-length output where you can.
- This is not a password manager: it does not store, sync, or rotate secrets. It only derives them.

## Troubleshooting

- `Error: PASSGEN_PEPPER environment variable is not set.`:
  - Export `PASSGEN_PEPPER` in your shell before running `passgen`.
- `No /dev/tty available. Run from an interactive terminal.`:
  - Run from a real interactive TTY (not from a context that has no controlling terminal).
- Pasting/clipboard tools appear to hang:
  - Remember the output has no newline; add `; echo` if a newline is required for your workflow.

