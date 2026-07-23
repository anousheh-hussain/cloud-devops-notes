[linux-user-management.md](https://github.com/user-attachments/files/30293368/linux-user-management.md)
# Linux User Management

A guide to user creation, modification, group management, and privilege escalation - core topics for RHCSA.

---

## Table of Contents

1. [Identity Commands](#identity-commands)
2. [Switching Users](#switching-users)
3. [Creating Users](#creating-users)
4. [Modifying and Deleting Users](#modifying-and-deleting-users)
5. [Groups and visudo](#groups-and-visudo)
6. [Important Config Files](#important-config-files)
7. [Practice Questions](#practice-questions)

---

## Identity Commands

These three commands look similar but return different information:

| Command | What it shows |
|---|---|
| `whoami` | Your **effective** username (changes after `su`) |
| `who am i` | Your **original login** identity (stays the same even after `su`) |
| `who` | All users currently logged into the system |

**Example:** If you logged in as `alice` and then ran `su - root`:
- `whoami` returns `root`
- `who am i` still returns `alice`

---

## Switching Users

```bash
su username        # Switch user but keep your current environment
su - username      # Switch user AND load their full environment (home dir, PATH, etc.)
```

> **Always use `su -`** in practice and in the RHCSA exam. Using `su` without the dash keeps your old `$PATH` which can cause unexpected errors when running commands as the new user.

```bash
exit      # Exit the current shell / drop back to previous user after su
logout    # Close a login shell (like an SSH session)
```

**RHCSA relevance:** You will frequently `su - username` to test whether the permissions or sudo rules you just configured actually work for that user.

---

## Creating Users

### The Full `useradd` Command

```bash
useradd -m -d /home/ava -c "Ava Adams" -s /bin/bash ava
```

| Flag | Meaning |
|---|---|
| `-m` | Create the home directory |
| `-d /home/ava` | Specify the home directory path |
| `-c "Ava Adams"` | Add a comment / full name |
| `-s /bin/bash` | Set the default shell |
| `-M` | Do NOT create a home directory |
| `-p $(openssl passwd -6 password)` | Set a hashed password at creation time |

### Creating a System/Service User (No login)

```bash
useradd -M -s /sbin/nologin servicename
```

Use this for service accounts that should never have interactive shell access.

### Setting a Password After Creation

Users cannot log in until a password is set:

```bash
passwd ava
```

### Automated Password in Scripts

`passwd` prompts interactively, which breaks scripts. Three ways around this:

```bash
# Method 1: openssl hash at creation time
useradd -m -p $(openssl passwd -6 mypassword) luke

# Method 2: pipe into passwd (for existing users)
echo "Welcome123!" | passwd --stdin username

# Method 3: chpasswd (best for bulk user creation)
echo "username:Welcome123!" | chpasswd
```

### What Happens Behind the Scenes

When you create a user:

1. An entry is added to `/etc/passwd` (user info)
2. An entry is added to `/etc/shadow` (password + aging)
3. An entry is added to `/etc/group`
4. The contents of `/etc/skel` are copied into the new home directory

### /etc/skel - The Skeleton Directory

`/etc/skel` contains default files that get copied into every new user's home directory on creation - things like `.bashrc` and `.bash_profile`. If you want all new users to start with a certain config or file, place it here before creating the user.

### Viewing Available Shells

```bash
cat /etc/shells    # Lists all valid login shells on the system
```

> If `chsh -l` is not working, `cat /etc/shells` is the reliable alternative.

### Default useradd Settings

```bash
cat /etc/default/useradd    # Shows defaults used when no flags are specified
```

---

## Modifying and Deleting Users

### `usermod` - Modify an Existing User

```bash
usermod -c "New Full Name" ava      # Change the comment/full name
usermod -s /bin/bash ava            # Change the shell
usermod -d /new/home ava            # Change home directory
usermod -L ava                      # Lock the account
usermod -U ava                      # Unlock the account
usermod -aG groupname ava           # Add to a secondary group (append)
```

> **Critical:** Always use `-aG` (append + group) when adding a user to a group. Using `-G` alone **removes the user from all other secondary groups** they currently belong to.

### `userdel` - Delete a User

```bash
userdel ava          # Delete user only (home directory stays)
userdel -r ava       # Delete user AND their home directory and mail spool
```

### `vipw` vs `vi /etc/passwd`

Use `vipw` instead of editing `/etc/passwd` directly with vi. `vipw` locks the file while you edit it and warns you if someone else is already editing it - preventing corruption from simultaneous edits.

---

## Groups and visudo

### Creating Groups and Adding Users

```bash
groupadd devteam                  # Create a new group
usermod -aG devteam ava           # Add ava to the group (append, never drop -a)
groups ava                        # See what groups ava belongs to
```

### visudo - Safe Sudoers Editing

**Always use `visudo`** to edit `/etc/sudoers`. It validates syntax before saving. Editing the file directly with `vi` and making a typo can completely break `sudo` and lock you out of root.

```bash
visudo
```

### Sudoers Syntax

```
# Give a single user full sudo access
ava  ALL=(ALL)  ALL

# Give a group full sudo access (groups are prefixed with %)
%devteam  ALL=(ALL)  ALL

# Allow a specific command without password prompt
ava  ALL=(ALL)  NOPASSWD: /usr/bin/systemctl restart sshd

# Grant full access but deny specific commands (! = NOT)
ava  ALL=(ALL)  ALL, !/usr/bin/passwd, !/usr/bin/su
```

> Use `whereis commandname` to find the absolute path required in visudo entries.  
> Example: `whereis passwd` -> `/usr/bin/passwd`

### The `wheel` Group

On RHEL-based systems, users in the `wheel` group get full sudo access by default (this rule is already in `/etc/sudoers`). The quickest way to give a user admin rights:

```bash
usermod -aG wheel ava
```

---

## Important Config Files

| File | Purpose |
|---|---|
| `/etc/passwd` | User account info (username, UID, GID, home, shell) |
| `/etc/shadow` | Hashed passwords and aging policies |
| `/etc/group` | Group definitions |
| `/etc/sudoers` | Sudo privilege rules (always edit with `visudo`) |
| `/etc/default/useradd` | Default values used by `useradd` |
| `/etc/skel` | Template files copied to new user home directories |
| `/etc/shells` | List of valid login shells |

---

*Notes built while studying for RHCSA - part of the [Cloud Girl Logs](https://cloudgirllogs.hashnode.dev/) series.*
