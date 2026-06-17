# Linux File Management & Navigation

A practical guide to file operations, navigation, documentation tools, and content viewing in Linux - core skills for RHCSA and daily sysadmin work.

---

## Table of Contents

1. [Key Directories in /](#key-directories-in-)
2. [Path Basics & Navigation](#path-basics--navigation)
3. [File & Directory Operations](#file--directory-operations)
4. [Links: Hard vs Soft](#links-hard-vs-soft)
5. [Aliases and Shortcuts](#aliases-and-shortcuts)
6. [Getting Help: man, whatis, whereis](#getting-help-man-whatis-whereis)
7. [Viewing and Extracting Content](#viewing-and-extracting-content)
8. [Pagers: more vs less](#pagers-more-vs-less)
9. [HEREDOC: Writing Files in Scripts](#heredoc-writing-files-in-scripts)

---

## Key Directories in /

A quick recap of the most commonly used directories, with practical notes.

| Directory | Purpose | Notes |
|---|---|---|
| `/bin` | Essential user commands (`ls`, `cp`, `ping`, `grep`) | On modern RHEL, often a symlink to `/usr/bin` |
| `/sbin` | Admin/system commands (`fdisk`, `iptables`, `ifconfig`) | Normal users typically need `sudo` to run these |
| `/dev` | Device files representing hardware | `/dev/sda` (disk), `/dev/null` (discards input), `/dev/random` (random data) |
| `/etc` | Configuration files for the system and installed programs | `/etc/ssh/sshd_config`, `/etc/passwd` |
| `/home` | Personal storage for regular users | e.g. `/home/username` |
| `/root` | Home directory for the root user | Kept separate from `/home` so root can still log in even if the `/home` partition fails |
| `/var` | Data that grows over time | Logs (`/var/log/messages`), databases, site files |
| `/tmp` | Temporary files | Usually cleared on every reboot - never store important work here |

**Interview Q: Difference between `/` and `/root`?**  
`/` is the top-level directory containing the entire filesystem. `/root` is specifically the home directory of the root user, the same way `/home/username` is for a regular user.

**Interview Q: Where would you go to change the hostname or network config?**  
`/etc`

### Quick Exploration

```bash
ls -F /     # -F appends a "/" to directory names so you can tell files and folders apart
```

---

## Path Basics & Navigation

### Absolute vs Relative Paths

- **Absolute path** - always starts from root `/`. Full address. Example: `/home/username/Documents`
- **Relative path** - starts from your current location, no leading `/`. Example: from `/home`, typing `cd username` is relative.

```bash
pwd    # Print Working Directory - shows your current absolute path
```

### Navigation Shortcuts

| Command | Meaning |
|---|---|
| `cd ~` | Go to home directory |
| `cd ..` | Move up one level |
| `cd -` | Toggle back to the previous directory |
| `cd /etc` | Jump directly to a given path |

**Tab completion:** Type `cd /et` and hit `Tab` to auto-complete to `cd /etc/`. Hit `Tab` twice to see all matches if it doesn't complete.

### The Tilde (`~`)

`~` is shorthand for the current user's home directory.

- Logged in as `root` -> `~` = `/root`
- Logged in as `username` -> `~` = `/home/username`

Typing `cd` with no arguments (or `cd ~`) always takes you home.

---

## File & Directory Operations

### `ls` Options

| Flag | Meaning |
|---|---|
| `-l` | Long listing - permissions, owner, size, date |
| `-a` | All files, including hidden ones (starting with `.`) |
| `-h` | Human-readable sizes (KB, MB, GB) |
| `-R` | Recursive - includes subdirectories |
| `-i` | Shows inode numbers |

Common combo: `ls -lah`

### `touch` - Creating Files

```bash
touch file.txt              # Create an empty file, or update its timestamp if it exists
touch file{1..5}.txt        # Brace expansion: creates file1.txt through file5.txt
touch {day,night}.log       # Creates day.log and night.log
touch parent/demo.txt       # Creates demo.txt inside parent/
```

### `mkdir` - Creating Directories

```bash
mkdir folder1                          # Create one directory
mkdir -p a/b/c/d                       # -p (parents): creates the full nested path at once
mkdir -p parent/child1 parent/child2   # Creates parent/ with two subdirectories
```

### `cp` - Copying

| Flag | Meaning |
|---|---|
| `cp source dest` | Basic copy |
| `cp -r` | Recursive - required for copying directories |
| `cp -p` | Preserve permissions, ownership, timestamps |
| `cp -a` | Archive - combines `-r` and `-p` plus more; best for full backups |

### `mv` - Moving and Renaming

```bash
mv file.txt /tmp/          # Move a file
mv oldname.txt newname.txt # Rename (Linux has no separate "rename" command - mv does both)
```

### `rm` and `rmdir` - Deleting

```bash
rmdir emptyfolder    # Only works on empty directories
rm file.txt           # Delete a file
rm -r folder/         # Recursive - deletes a folder and its contents
rm -rf folder/        # Force - deletes without confirmation prompts
```

> **Caution:** `rm -rf` does not ask for confirmation. Running it on the wrong path (especially as root) can destroy data instantly. Double-check the path before hitting enter.

### Redirection Operators

```bash
echo "Hello" > file.txt    # Overwrites file content
echo "World" >> file.txt   # Appends to file content
```

### Identifying File Types

```bash
file somefile    # Linux doesn't rely on extensions - this inspects the file's actual content
                 # e.g. "ASCII text" or "ELF 64-bit executable"
```

### Case Sensitivity

Linux is strictly case-sensitive. `File.txt`, `file.txt`, and `FILE.txt` are three different files in the same directory. Commands are case-sensitive too - `LS` will return "command not found."

---

## Links: Hard vs Soft

| Type | Command | Behavior |
|---|---|---|
| Soft link (symbolic) | `ln -s /path/original linkname` | Points to a filename. Breaks ("dangling link") if the original is deleted or moved. |
| Hard link | `ln /path/original linkname` | Points to the actual inode/data. Data persists even if the original filename is deleted. |

**Restrictions on hard links:**
- Cannot link directories
- Cannot link across different filesystems/disks

**Interview Q: Why can't you hard-link a directory?**  
To prevent infinite loops in the filesystem structure.

---

## Aliases and Shortcuts

```bash
alias c="clear"     # Create a shortcut - typing 'c' now clears the screen
alias                # List all current aliases
unalias c            # Remove an alias
```

> Aliases disappear after logout unless saved in `~/.bashrc`.

**Chaining commands with `;`:** runs commands sequentially regardless of success/failure.

```bash
mkdir backup ; cd backup ; pwd
```

**Useful info commands:**

```bash
lsblk          # List block devices - disks and partitions, with mount points
ll             # Common alias for 'ls -l' on RHEL/AlmaLinux systems
type cd        # Shows whether a command is a shell builtin or an actual file
```

---

## Getting Help: man, whatis, whereis

In the RHCSA exam there's no internet access, so knowing the built-in documentation tools is essential.

### `man` - The Manual

```bash
man ls          # Open the manual page for 'ls'
man man         # The manual for the manual itself
man -k keyword  # Search for commands by what they do
man 5 sshd_config   # Section 5 = config file formats (different from the command's own man page)
```

**Navigation inside `man`:** `Space` to page down, `q` to quit, `/word` to search, `n` to jump to the next match.

### `mandb` - Updating the Manual Index

```bash
sudo mandb
```

If you install a new program and its man page doesn't show up in searches, run this to rebuild the index.

### `whatis` and `whereis`

```bash
whatis ssh      # One-line description of what a command does
whereis ssh     # Shows the binary, source, and man page locations
```

### `--help`

```bash
ls --help       # Quick summary printed directly in the terminal (faster, less detailed than man)
```


## Viewing and Extracting Content

### `cat` - Concatenate and Display

```bash
cat file.txt                  # Print the whole file
cat file1 file2 > combined.txt   # Merge two files into one
cat > newfile.txt             # Create a file by typing directly; save with Ctrl+D
cat >> existing.txt           # Append typed text to a file; save with Ctrl+D
```

### `tac` - Cat Reversed

Prints a file from the last line to the first. Useful for logs where the newest entries are at the bottom.

### `head` and `tail`

```bash
head -n 5 file.txt    # First 5 lines (default 10)
tail -n 5 file.txt     # Last 5 lines
tail -f file.txt       # Follow mode - updates in real time as new lines are added (used for live log monitoring)
```

### `strings` - Reading Binary Files

```bash
strings /bin/ls    # Extracts human-readable text from a binary file
```

Running `cat` directly on a binary file fills the screen with unreadable characters - `strings` filters out everything except readable text.

### `echo`

```bash
echo "Hello World"                          # Print text
echo $USER                                   # Print a variable (current username)
echo "nameserver 8.8.8.8" > /etc/resolv.conf  # Quick way to write a single line into a config file
```

---

## Pagers: more vs less

When a file is too long for one screen:

| Feature | `more` | `less` |
|---|---|---|
| Scroll up | No | Yes |
| Search | Limited | Yes (`/searchterm`) |
| Loads whole file into memory | Yes | No (faster on huge files) |

```bash
more /etc/passwd    # Enter = next line, Space = next page
less /etc/passwd    # Scroll freely, search with /
```

> "Less is more" - `less` is the modern standard and is generally preferred over `more`.

---

## HEREDOC: Writing Files in Scripts

A way to write multi-line content into a file without using an editor - commonly used in automation scripts.

```bash
cat << END > file1
hello
linux is fun
this is practice
END
```

Everything between the two `END` markers gets written into `file1`. This pattern is widely used in DevOps scripts to generate configuration files on the fly.

---

*Notes built while studying for RHCSA - part of the [Cloud Girl Logs](https://cloudgirllogs.hashnode.dev/) series.*
