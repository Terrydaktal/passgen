# passgen

Deterministic password/key generator for terminal use.

`passgen` has two modes:

- `passgen` (default): PBKDF2-HMAC-SHA256 via `openssl kdf`
- `passgen --argon`: Argon2id via the compiled `argon2` CLI binary

It is designed so you can safely pipe the generated output (stdout) into other commands while keeping prompts on your terminal (`/dev/tty`).

## How It Works

In both modes, `passgen`:

1. Builds `salt = (site + username)` (UTF-8).
2. Builds `key = (password + PASSGEN_PEPPER)` (UTF-8).
3. Derives `64` bytes:
   - default mode: `PBKDF2-HMAC-SHA256` via `openssl kdf`
   - `--argon` mode: `Argon2id` via `argon2`
4. Encodes the derived bytes as either:
   - Base64 (unpadded; `=` removed), or
   - Z85 (ZeroMQ Base85 alphabet)
5. Optionally truncates to the requested length.

Important: `passgen` writes the result with **no trailing newline**.

## Requirements

- POSIX `sh`
- `openssl` 3.x (`openssl kdf`)
- `awk`
- `od`
- `base64`
- `argon2` binary (only for `--argon` mode)

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

Default mode requires `PASSGEN_PEPPER`.

`--argon` mode accepts `PASSGEN_PEPPER` as optional (empty if unset).

The pepper should be:

- long (at least 16+ bytes; longer is fine)
- high entropy (random)
- stable (if you change it, all derived passwords/keys change)
- treated like a secret

Example (generate a pepper once, store it somewhere secure, then export it):

```bash
export PASSGEN_PEPPER="$(openssl rand -base64 48 | tr -d '\n')"
```

Avoid hard-coding the pepper into shell history or world-readable dotfiles.

## Usage

Interactive usage:

```bash
passgen
```

Argon mode:

```bash
passgen --argon
```

You will be prompted for:

- In both modes:
  - `Site`
  - `Username`
  - `Encoding` (`64` for Base64, `85` for Z85)
  - `Length` (blank for full output)
- In default mode:
  - `Master Password` (hidden)
- In `--argon` mode:
  - `Password` (hidden)
  - `Time`
  - `Memory` KiB
  - `Parallelism`

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

## Encoding Notes

- Base64 output uses the standard Base64 alphabet (`A-Z a-z 0-9 + /`).
- Z85 output uses the ZeroMQ Z85 alphabet, which includes punctuation; it can be more acceptable in some systems because it avoids `+` and `/` but includes other symbols.

If a target system has strict password character rules, choose the encoding/length that fits, or truncate accordingly.

## Security Model and Caveats

- Deterministic: the same inputs produce the same output.
- If any inputs/parameters for a mode change, the output changes.
- `PASSGEN_PEPPER` acts as an extra secret factor when set. Treat it like a key.
- Truncation reduces strength. Prefer using full-length output where you can.
- This is not a password manager: it does not store, sync, or rotate secrets. It only derives them.

## Troubleshooting

- `Error: PASSGEN_PEPPER environment variable is not set.`:
  - Export `PASSGEN_PEPPER` before running default mode (`passgen`).
- `Error: argon2 binary not found.`:
  - Install package `argon2` (Debian/Ubuntu: `sudo apt install argon2`).
- `combined Site+Username must be at least 8 characters`:
  - Argon2 salt input must be at least 8 chars.
- `No /dev/tty available. Run from an interactive terminal.`:
  - Run from a real interactive TTY (not from a context that has no controlling terminal).
- Pasting/clipboard tools appear to hang:
  - Remember the output has no newline; add `; echo` if a newline is required for your workflow.
