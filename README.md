# ssh.qui.icu

Import SSH public keys from [sshid.io](https://sshid.io) into `~/.ssh/authorized_keys` with one command. New keys are merged with the ones already in the file, duplicates are removed, and a backup is kept.

## Quick start

**Import anyone's keys (interactive):**

```sh
curl -fsSL https://ssh.qui.icu/import | sh
```

The script asks for an sshid.io username, lists that user's keys with numbers, and lets you pick which ones to import. It shows exactly which file it will change and asks before writing anything.

**All of Qui's keys, every type (non-interactive):**

```sh
curl -fsSL https://ssh.qui.icu/keys | sh
```

**For root instead of your own account,** pipe to `sudo sh` instead of `sh`. Keys always go to the authorized_keys file of the user the script runs as.

## Options for `import`

Pass options after `sh -s --`:

```sh
curl -fsSL https://ssh.qui.icu/import | sh -s -- [-a] [-t TYPES] [-n NUMS] [-y] [USERNAME]
```

| Option | What it does |
|---|---|
| `USERNAME` | sshid.io username. If left out, you'll be prompted. |
| *(none)* | With no `-a`, `-t` or `-n`, the default is **`ed25519,ecdsa,sk`**. That's ED25519, ECDSA, and hardware security keys/passkeys (`sk-ecdsa`, `sk-ed25519`). Only RSA is left out. |
| `-a` | Import all keys, including RSA, and skip the selection prompt. |
| `-t TYPES` | Import only these key types, comma-separated: `ed25519`, `ecdsa`, `rsa`, `sk` (all hardware keys), `sk-ed25519`, `sk-ecdsa`. |
| `-n NUMS` | Import only these keys by their number in the list, e.g. `1,3,5`. |
| `-y` | Don't ask for confirmation. |
| `-h` | Show usage. |

You can combine `-t` and `-n`. A key is imported if it matches either one.

At the interactive selection prompt, you can press **Enter** for the default (`ed25519,ecdsa,sk`), type `all`, type numbers (`1 3`), or type key types (`ed25519,sk`).

## Examples

```sh
# Prompt for everything
curl -fsSL https://ssh.qui.icu/import | sh

# Only hardware security keys (YubiKey etc.) for user qui-le
curl -fsSL https://ssh.qui.icu/import | sh -s -- -t sk qui-le

# Keys 1 and 3 from the list, no confirmation
curl -fsSL https://ssh.qui.icu/import | sh -s -- -n 1,3 -y qui-le

# Everything, into root's authorized_keys
curl -fsSL https://ssh.qui.icu/import | sudo sh -s -- -a qui-le
```

## Running it on a schedule

With no terminal attached (cron, Ansible, cloud-init), the script never prompts. Pass the username, and add `-a` or `-t`/`-n` if needed. Without those, it imports the default `ed25519,ecdsa,sk` keys (everything except RSA).

```sh
# crontab -e   (daily at 3:17am)
17 3 * * * curl -fsSL https://ssh.qui.icu/import | sh -s -- -a qui-le >/dev/null
```

## What it does

1. Fetches every sshid.io key list for the user: `https://sshid.io/USERNAME` plus `/ECDSA-SK`, `/ED25519`, `/ECDSA` and `/RSA`. It keeps only valid key lines (`ssh-`, `ecdsa-`, `sk-`) and drops duplicates. A type the user doesn't have is skipped.
2. Shows the keys with fingerprints so you can choose.
3. Copies your current file to `~/.ssh/authorized_keys.bak`.
4. Merges the chosen keys into `~/.ssh/authorized_keys`, removing duplicates by key value. Your existing keys and comments are kept.
5. Sets `~/.ssh` to `700` and `authorized_keys` to `600`.

**What it won't do:**

- **It never removes keys.** A key deleted from sshid.io stays on the server until you remove it by hand.
- **It doesn't touch the file** if the fetch fails, the username is invalid, or nothing matches your selection.

**To undo the last run:**

```sh
cp -p ~/.ssh/authorized_keys.bak ~/.ssh/authorized_keys
```

## Security

Importing someone's keys gives **that sshid.io account** SSH access to the account you run the script as. Only import usernames you trust, and type them carefully.

Piping a script into `sh` runs whatever the server sends. If you want to read it first:

```sh
curl -fsSL https://ssh.qui.icu/import | less
```

**Requirements:** a POSIX `sh`, `curl`, `awk`, and `getent` (any normal Linux system). `ssh-keygen` is optional and only used to show fingerprints.
