[linux-password-management.md](https://github.com/user-attachments/files/30293425/linux-password-management.md)
# Linux Password Management

A guide to password security, the shadow file, aging policies, and account locking - core topics for RHCSA.

---

## Table of Contents

1. [The /etc/shadow File](#the-etcshadow-file)
2. [Password Aging with chage](#password-aging-with-chage)
3. [The passwd Command](#the-passwd-command)
4. [Locking and Disabling Accounts](#locking-and-disabling-accounts)
5. [Practice Questions](#practice-questions)

---

## The /etc/shadow File

`/etc/shadow` stores hashed passwords and aging policies. Only root can read it. **Never edit it directly** - use `passwd`, `chage`, or `usermod` which have built-in validation.

A typical line looks like:

```
username:$6$salt$hashedpassword:19460:0:90:7:-1:-1:
```

Each field is separated by `:`:

| Field | Example | Meaning |
|---|---|---|
| Username | `username` | The login name |
| Password hash | `$6$...` | Hashed password (`$6$` = SHA-512) |
| Last changed | `19460` | Days since Jan 1, 1970 that password was last changed |
| Min days | `0` | Minimum days before the password can be changed again |
| Max days | `90` | Maximum days before the password must be changed |
| Warning days | `7` | Days of warning given before expiry |
| Inactive days | `-1` | Grace period after expiry before account is locked (-1 = disabled) |
| Expiry date | `-1` | Account expiration date (-1 = never expires) |

### What the password field looks like

| Value | Meaning |
|---|---|
| `$6$abc...` | Normal hashed password - account works fine |
| `!$6$abc...` | Locked with `usermod -L` (one `!` prepended) |
| `!!` | Locked with `passwd -l`, or newly created with no password set |
| (empty) | No password - dangerous, allows local login without a password |

---

## Password Aging with chage

`chage` (Change Age) is the safest way to view and modify password aging policies for existing users. It edits `/etc/shadow` safely behind the scenes.

### Viewing Current Policy

```bash
chage -l username    # List all aging settings for a user
```

### Setting Aging Policies

```bash
chage -m 0 username              # Min days (0 = user can change password anytime)
chage -M 90 username             # Max days before password must be changed
chage -W 7 username              # Warning days before expiry
chage -E "2026-12-31" username   # Set exact account expiration date
chage -d 0 username              # Force password change on next login
```

### /etc/login.defs - Default Aging Settings

This file controls the defaults applied when **new users are created**:

```bash
vi /etc/login.defs
```

Key settings to know:

```
PASS_MAX_DAYS   90
PASS_MIN_DAYS   0
PASS_WARN_AGE   7
```

> **Important:** Changes to `/etc/login.defs` only affect users created **after** the change. For existing users, you must run `chage` manually.

### Example: Force all new users to change password every 10 days

```bash
vi /etc/login.defs
# Set: PASS_MAX_DAYS 10
```

For existing users you still need:

```bash
chage -M 10 username
```

---

## The passwd Command

### Basic Usage

```bash
passwd              # Change your own password
passwd username     # Change another user's password (root only)
```

Root can change any user's password without knowing their current one. Regular users must know their current password to change it.

### passwd Options

| Flag | Action | Example |
|---|---|---|
| `-S` | Show password status summary | `passwd -S ava` |
| `-d` | Delete the password (dangerous) | `passwd -d ava` |
| `-e` | Expire - force change on next login | `passwd -e ava` |
| `-l` | Lock the account | `passwd -l ava` |
| `-u` | Unlock the account | `passwd -u ava` |
| `--stdin` | Read password from pipe (for scripts) | `echo "pass" \| passwd --stdin ava` |

### Reading `passwd -S` Output

```bash
passwd -S ava
# ava PS 2026-04-18 0 90 7 -1 (Password set, SHA512 crypt.)
```

The second column is the most important:

| Code | Meaning |
|---|---|
| `PS` | Password Set - account is working normally |
| `LK` | Locked - cannot log in via password |
| `NP` | No Password - password was deleted (dangerous) |

### `passwd -e` vs `chage -d 0`

These do the exact same thing - force a password change on next login. `passwd -e` is just faster to type.

```bash
passwd -e username     # Force change on next login
chage -d 0 username    # Same result
```

### `passwd -d` vs `passwd -l` - Know the Difference

| Command | Effect | Risk Level |
|---|---|---|
| `passwd -l` | Locks account (adds `!` to hash - password preserved) | Safe |
| `passwd -d` | Deletes the password entirely | Dangerous |

> `passwd -d` sets the account to `NP` (No Password) status. Anyone who knows the username can log in locally without being prompted for a password. SSH blocks this by default (`PermitEmptyPasswords no` in `sshd_config`) but **local console logins do not**. Avoid `passwd -d` unless you have a very specific reason.

### Using openssl to Generate Password Hashes

When writing automation scripts, you can generate a secure SHA-512 hash on the fly:

```bash
openssl passwd -6 mypassword         # Generate a SHA-512 hash

# Use it directly in useradd
useradd -m -p $(openssl passwd -6 mypassword) luke
```

---

## Locking and Disabling Accounts

There are three levels of restricting access, from lightest to strongest.

### Level 1 - Lock with usermod

```bash
usermod -L username    # Lock (adds one ! to shadow file)
usermod -U username    # Unlock
```

### Level 2 - Lock with passwd

```bash
passwd -l username     # Lock (adds !! to shadow file)
passwd -u username     # Unlock
```

### Difference between usermod -L and passwd -l

Both prevent password-based login but they differ in one important way:

- `usermod -L` adds a single `!` - if the user has SSH keys set up, they may still be able to log in via SSH key authentication
- `passwd -l` adds `!!` and is generally considered a stronger lock

> Always unlock with the same tool you used to lock. If you locked with `usermod -L`, unlock with `usermod -U`. If you locked with `passwd -l`, unlock with `passwd -u`. Mixing them can cause inconsistencies in the shadow file.

### Level 3 - Remove Shell Access Completely (Strongest)

```bash
usermod -s /sbin/nologin username
```

This blocks the user from getting a shell even if they have SSH keys. When they connect, they see a message and are immediately disconnected.

You can also do this by editing the user's entry in `/etc/passwd` using `vipw` and changing the shell field from `/bin/bash` to `/sbin/nologin`.

> On RHEL/AlmaLinux, always use `/sbin/nologin`, not `/bin/nologin`. Using the wrong path on the RHCSA exam may cause it to be rejected.

### Locking vs Deleting a Password - Summary

| Action | Command | What happens in /etc/shadow | Can user log in? |
|---|---|---|---|
| Lock | `usermod -L` | `!` prepended to hash | No (password blocked) |
| Lock | `passwd -l` | `!!` replaces hash | No (stronger block) |
| Delete password | `passwd -d` | Password field emptied | Yes (no password needed locally) |
| Remove shell | `usermod -s /sbin/nologin` | Hash unchanged | No shell access at all |

---

*Notes built while studying for RHCSA - part of the [Cloud Girl Logs](https://cloudgirllogs.hashnode.dev/) series.*
