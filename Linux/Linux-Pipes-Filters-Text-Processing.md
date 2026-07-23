[linux-pipes-filters-and-text-processing.md](https://github.com/user-attachments/files/30293167/linux-pipes-filters-and-text-processing.md)
# Linux Pipes, Filters & Text Processing

A practical guide to I/O redirection, text filters, compression, regex, and the Vi editor - essential skills for RHCSA and day-to-day sysadmin work.

---

## Table of Contents

1. [I/O Redirection](#io-redirection)
2. [Text Filters](#text-filters)
3. [sed - Stream Editor](#sed---stream-editor)
4. [Compression and Archiving](#compression-and-archiving)
5. [Finding Files](#finding-files)
6. [Regular Expressions (Regex)](#regular-expressions-regex)
7. [Vi / Vim Editor](#vi--vim-editor)

---

## I/O Redirection

Every process in Linux has three default "streams" - think of them as pipes carrying data in and out of a command.

| Descriptor | Name | Symbol | Purpose |
|---|---|---|---|
| 0 | stdin | `<` | Standard Input - where the command reads data from (keyboard by default) |
| 1 | stdout | `>` or `1>` | Standard Output - where normal results go (screen by default) |
| 2 | stderr | `2>` | Standard Error - where error messages go (screen by default) |

### Redirecting Output to a File

```bash
ls -l /etc > output.txt        # Save stdout to a file (overwrites)
ls -l /etc >> output.txt       # Append stdout to a file
```

### Separating Output and Errors

```bash
find /etc -name "*.conf" 1> success.txt 2> errors.txt
# Good results go to success.txt, errors go to errors.txt
```

### Merging Streams with `2>&1`

If you want everything (output + errors) in the same file:

```bash
command > file.txt 2>&1
```

How to read this: "Send stdout to file.txt, then send stderr to wherever stdout is going."

### The Black Hole: `/dev/null`

`/dev/null` is a special file that silently discards everything written to it.

```bash
grep -r "something" /etc 2> /dev/null    # Suppress all error messages
command > /dev/null 2>&1                  # Suppress everything - no output at all
```

### Nullifying (Emptying) a File

When a log file grows too large and you want to wipe it without deleting it:

```bash
> file.txt              # Simplest method
true > file.txt         # Alternative
cat /dev/null > file.txt   # Explicit method
```

### No Clobber - Protecting Files from Accidental Overwrite

By default `>` overwrites files without warning. Enable protection with:

```bash
set -o noclobber
```

Now `>` will refuse to overwrite an existing file. Check it's active by running `echo $-` - a capital `C` in the output confirms it's on.

To force overwrite when no clobber is enabled:

```bash
echo "force overwrite" >| existing_file.txt    # >| bypasses the protection
```

### Why doesn't the order of redirection throw an error?

The shell sets up redirection **before** running the command. When you type `ls > file.txt`, the shell first opens the file and replaces stdout, then runs `ls`. The command never knows it's being redirected - it just writes to what it thinks is the screen.

### Interview Q&A

**Q: What does `2>&1` mean?**  
Redirect stderr (stream 2) to the same destination as stdout (stream 1).

**Q: How do you hide all output from a command?**  
`command > /dev/null 2>&1`

**Q: How do you empty a file without deleting it?**  
`> filename`

**Q: What is the file descriptor number for stdin?**  
0

---

## Text Filters

These tools are the core of Linux text processing. They are often chained together using pipes (`|`) to process data step by step.

### The Pipe `|`

The pipe sends the output of one command as the input to the next.

```bash
cat /etc/passwd | grep "bash"    # Filter only lines containing "bash"
ps -ef | wc -l                   # Count total running processes
```

---

### `grep` - Search Inside Files or Streams

The most used text filter. Finds lines that match a pattern.

| Flag | Meaning |
|---|---|
| `-i` | Ignore case (matches ERROR, error, Error) |
| `-v` | Invert - show lines that do NOT match |
| `-r` | Recursive - search all files in a directory |
| `-n` | Show line numbers alongside matches |
| `-w` | Match whole words only |
| `-E` | Extended regex - allows `+`, `?`, `|` without backslashes |

```bash
grep -i "error" /var/log/messages          # Case-insensitive search
grep -v "^#" /etc/ssh/sshd_config          # Skip comment lines
grep -rn "PermitRootLogin" /etc/ssh/       # Recursive search with line numbers
grep -E "apple|orange" file.txt            # Match either word
```

**Lab challenge:** Show only active (non-comment, non-empty) lines from sshd_config:

```bash
grep -v "^#" /etc/ssh/sshd_config | grep -v "^$"
```

---

### `cut` - Extract Columns from Text

Used to slice out specific columns or character ranges from each line.

| Flag | Meaning |
|---|---|
| `-d` | Delimiter - defines the column separator |
| `-f` | Field - which column to extract |
| `-c` | Characters - extract specific character positions |

```bash
cut -d ":" -f 1 /etc/passwd        # Extract all usernames (column 1, separated by :)
cut -d ":" -f 1,3 /etc/passwd      # Extract columns 1 and 3
cut -c 1-10 file.txt               # Extract the first 10 characters of each line
```

---

### `tr` - Translate or Delete Characters

Transforms or removes characters. Only reads from a pipe or redirection, not directly from files.

```bash
echo "hello world" | tr 'a-z' 'A-Z'    # Convert to uppercase
cat file.txt | tr -d ' '                # Delete all spaces
cat file.txt | tr -s ' '                # Squeeze multiple spaces into one
cat file.csv | tr ',' '\t' > new.tsv    # Replace commas with tabs
```

---

### `wc` - Word Count

Counts lines, words, or bytes.

| Flag | Meaning |
|---|---|
| `-l` | Count lines |
| `-w` | Count words |
| `-c` | Count bytes |

```bash
wc -l /etc/passwd              # How many users are in the system
ps -ef | wc -l                 # Count all running processes
grep "error" app.log | wc -l   # Count error lines in a log
```

---

### `sort` - Sort Lines

```bash
sort file.txt           # Alphabetical sort
sort -n file.txt        # Numeric sort (treats 10 as greater than 2)
sort -r file.txt        # Reverse sort
sort -k 2 file.txt      # Sort by the 2nd column
```

---

### `uniq` - Remove Duplicate Lines

**Important:** `uniq` only removes consecutive duplicates. Always run `sort` first.

```bash
sort names.txt | uniq             # Remove duplicates
sort names.txt | uniq -c          # Count occurrences of each line
sort names.txt | uniq -d          # Show only the duplicates
```

**Lab challenge:** List all users and count how many times each appears:

```bash
cut -d ":" -f 1 /etc/passwd | sort | uniq -c
```

---

### `comm` - Compare Two Files

Compares two **sorted** files and shows what is unique to each and what is common.

```bash
comm file1.txt file2.txt
```

Output has three columns:
- Column 1 - lines only in file1
- Column 2 - lines only in file2
- Column 3 - lines in both files

```bash
comm -12 file1.txt file2.txt    # Show only common lines
comm -23 file1.txt file2.txt    # Show only lines unique to file1
```

---

### `od` - Octal Dump

Used to inspect binary files or see hidden characters inside text files.

```bash
od -c filename       # Show content as characters (reveals \n, \t, etc.)
od -x filename       # Show content in hexadecimal
```

---

### Interview Q&A

**Q: How do you count lines containing the word "fail" in a log?**  
`grep -i "fail" logfile | wc -l`

**Q: Why must you use `sort` before `uniq`?**  
`uniq` only removes adjacent duplicates. Without sorting first, scattered duplicates won't be detected.

**Q: How do you extract the first 10 characters of each line?**  
`cut -c 1-10 filename`

**Q: What is the difference between `>` and `>>`?**  
`>` overwrites the file. `>>` appends to it.

---

## sed - Stream Editor

`sed` edits text without opening the file in an editor. It is widely used in automation scripts and configuration management.

### Core Syntax

```
sed 's/old/new/flags' filename
```

### Essential Flags and Commands

| Flag / Command | Meaning | Example |
|---|---|---|
| `s` | Substitute - replace text | `sed 's/red/blue/' file.txt` |
| `g` | Global - replace all matches on a line | `sed 's/a/A/g' file.txt` |
| `-i` | In-place - edit the file permanently | `sed -i 's/off/on/' config.conf` |
| `-i.bak` | In-place with backup | `sed -i.bak 's/a/b/' file.txt` |
| `-n` | Silent mode - suppress default output | Used with `p` to print only matches |
| `d` | Delete matching lines | `sed '/^$/d' file.txt` |
| `p` | Print matching lines | `sed -n '/error/p' file.txt` |
| `!` | NOT - apply to lines that do NOT match | `sed '/important/!d' file.txt` |
| `.` (dot) | Wildcard - matches any single character | `sed 's/202./2026/g' file.txt` |

### Practical Examples

```bash
# Replace first occurrence per line
sed 's/apple/orange/' file.txt

# Replace all occurrences (global)
sed 's/apple/orange/g' file.txt

# Edit permanently (in-place)
sed -i 's/localhost/127.0.0.1/g' config.conf

# Edit permanently with backup (safe approach)
sed -i.bak 's/localhost/127.0.0.1/g' config.conf

# Delete line 5
sed -i '5d' filename

# Delete all empty lines
sed -i '/^$/d' filename

# Print only matching lines
sed -n '/error/p' logfile

# Replace only on lines containing a specific word
sed '/server/s/old/new/g' filename

# Comment out a line containing a pattern
sed -i '/PermitRootLogin/s/^/#/' /etc/ssh/sshd_config

# Remove leading spaces
sed 's/^[ ]*//' file.txt

# Remove trailing spaces
sed 's/[ ]*$//' file.txt

# Remove all spaces
sed 's/ //g' file.txt
```

### Advanced: Grouping and Back-references

In Basic Regex (BRE), parentheses must be escaped. They capture a match so you can reuse it with `\1`, `\2`, etc.

```bash
# Reformat a date from DD-MM-YYYY to YYYY/MM/DD
echo "17-04-2026" | sed 's/\([0-9]\{2\}\)-\([0-9]\{2\}\)-\([0-9]\{4\}\)/\3\/\2\/\1/'
# Output: 2026/17/04
```

### Matching Specific Counts `{number}`

```bash
sed -n '/[0-9]\{5\}/p' file.txt    # Print lines containing exactly 5 digits in a row
```

### Interview Q&A

**Q: What happens if you forget the `g` flag in a substitution?**  
Only the first match on each line gets replaced. The rest are left unchanged.

**Q: Why is `-i` dangerous and how do you use it safely?**  
It permanently modifies the file with no undo. Use `-i.bak` to create a backup before editing.

**Q: How do you print only lines that match a pattern without showing the entire file?**  
`sed -n '/pattern/p' filename`

**Q: How do you replace text only on lines that contain a specific word?**  
`sed '/word1/s/old/new/g' filename`

---

## Compression and Archiving

### Creating Files from Command Output

```bash
ls -l /etc > etc_list.txt    # Save command output directly into a file
```

### Compression Tools (Single Files)

These compress one file at a time. They do not bundle multiple files together.

| Tool | Compress | Decompress | Extension | Notes |
|---|---|---|---|---|
| gzip | `gzip file.txt` | `gunzip file.txt.gz` | `.gz` | Most common, fast |
| bzip2 | `bzip2 file.txt` | `bunzip2 file.txt.bz2` | `.bz2` | Better compression, slower |
| xz | `xz file.txt` | `unxz file.txt.xz` | `.xz` | Best compression ratio, used for kernel files |

### `tar` - Bundle and Compress Multiple Files

`tar` bundles files/directories into a single archive. Combined with a compression flag, it also compresses.

| Flag | Meaning |
|---|---|
| `-c` | Create a new archive |
| `-x` | Extract an archive |
| `-v` | Verbose - show files being processed |
| `-f` | File - must come just before the filename |
| `-z` | Use gzip compression (.tar.gz) |
| `-j` | Use bzip2 compression (.tar.bz2) |
| `-J` | Use xz compression (.tar.xz) |
| `-t` | List contents without extracting |

```bash
# Create a compressed archive
tar -cvzf backup.tar.gz /home/username/       # gzip
tar -cvjf backup.tar.bz2 /etc/               # bzip2
tar -cvJf backup.tar.xz /var/log/            # xz

# Extract an archive
tar -xvzf backup.tar.gz                      # Extract here
tar -xvzf backup.tar.gz -C /tmp/             # Extract to /tmp

# List contents without extracting
tar -tf backup.tar.gz
```

**RHCSA Exam scenario:** "Create an archive of `/etc` called `/root/backup.tar.bz2` using bzip2."

```bash
tar -cvjf /root/backup.tar.bz2 /etc
```

### Interview Q&A

**Q: Difference between `find` and `locate`?**  
`find` searches the live filesystem (accurate, slower). `locate` searches a pre-built database (fast, can be outdated).

**Q: Which compression tool gives the smallest file size?**  
Generally xz > bzip2 > gzip in compression ratio.

**Q: How do you view the contents of a tar file without extracting?**  
`tar -tf filename.tar`

**Q: How do you find and delete files larger than 50MB in `/tmp`?**  
`find /tmp -size +50M -delete` (use with caution)

---

## Finding Files

### `find` - Search the Live Filesystem

The most powerful search tool. Searches in real time.

```bash
find /etc -name "*.conf"           # Find by name (wildcard)
find /var -type d                  # Find only directories (-type f for files)
find / -size +100M                 # Find files larger than 100MB
find /home -user username          # Find files owned by a user
find /var/log -mtime -7            # Modified in the last 7 days
find /etc -newer /etc/passwd       # Modified more recently than passwd
find .                             # List everything in current directory
```

### `locate` - Search a Database

Faster than `find` but searches an index, not the live filesystem. Won't find recently created files until the database is updated.

```bash
# Install (not included in minimal RHEL installs)
dnf install mlocate

# Update the database
updatedb

# Search
locate sshd_config
```

---

## Regular Expressions (Regex)

Regex lets you search for patterns rather than exact text. Used heavily with `grep`, `sed`, and `awk`.

### The Three Flavors

| Flavor | Command | Description |
|---|---|---|
| BRE | `grep` | Basic - `+`, `?`, `{}` need a backslash to work |
| ERE | `grep -E` | Extended - those same characters work without backslashes |
| PCRE | `grep -P` | Perl Compatible - most powerful, supports `\d`, lookaheads, etc. |

### Anchors - WHERE to Match

```bash
grep "^root" /etc/passwd        # Lines starting with "root"
grep "bash$" /etc/passwd        # Lines ending with "bash"
grep "^$" file.txt              # Empty lines
grep -v "^$" file.txt           # Non-empty lines
```

### Word Boundaries

```bash
grep "\bbin\b" file.txt         # Matches "bin" but not "bind" or "cabin"
grep -w "bin" file.txt          # Same result, simpler syntax
```

### Quantifiers - HOW MANY Times

| Symbol | Meaning | Requires ERE? |
|---|---|---|
| `*` | Zero or more of the previous character | No |
| `+` | One or more | Yes (`-E`) |
| `?` | Zero or one (optional) | Yes (`-E`) |

```bash
grep "lo*" file.txt           # Matches "l", "lo", "loo", "looo"
grep -E "lo+" file.txt        # Matches "lo", "loo" but NOT "l"
grep -E "colors?" file.txt    # Matches "color" and "colors"
```

### OR Matching

```bash
grep -E "apple|orange" file.txt    # Match either word
```

### Common Practical Examples

```bash
# Find all non-comment, non-empty lines in a config file
grep -v "^#" /etc/ssh/sshd_config | grep -v "^$"

# Find lines that are exactly one word
grep "^word$" file.txt

# Match lines with exactly 5 digits (like a zip code)
grep -E "[0-9]{5}" file.txt
```

### Interview Q&A

**Q: Difference between `grep "abc*"` and `grep -E "abc+"`?**  
`abc*` matches "ab" plus any number of c's (including zero). `abc+` requires at least one "c".

**Q: How do you search for either "apple" or "orange"?**  
`grep -E "apple|orange" filename`

**Q: How do you find a line that contains ONLY the word "end"?**  
`grep "^end$" filename`

**Q: Why use `grep -w` instead of regular `grep`?**  
To avoid partial matches. `grep "user"` matches "username" and "user1". `grep -w "user"` matches only the exact word "user".

**Q: What does `grep -v "^#"` do?**  
Shows all lines that are NOT comments (does not start with `#`). Useful for reading config files cleanly.

---

## Vi / Vim Editor

Vi (or its improved version Vim) is the most critical tool for the RHCSA exam. The exam is 100% command-line based, so you must be able to edit config files quickly without a GUI.

> **Quick start:** Run `vimtutor` in your terminal for an interactive hands-on tutorial built into Vim itself.

### The 3 Modes of Vi

| Mode | How to Enter | Purpose |
|---|---|---|
| Command Mode | Default / press `Esc` | Navigation, deletion, copying |
| Insert Mode | `i`, `a`, `o`, etc. | Typing and editing text |
| Last Line Mode | `:` | Saving, quitting, search/replace, settings |

---

### Navigation (Command Mode)

| Key | Action |
|---|---|
| `0` | Jump to start of line |
| `$` | Jump to end of line |
| `w` | Jump forward one word |
| `b` | Jump back one word |
| `gg` | Go to the first line of the file |
| `G` | Go to the last line |
| `J` | Join the line below to the current line |

---

### Entering Insert Mode

| Key | Where it inserts |
|---|---|
| `i` | Before the cursor |
| `a` | After the cursor |
| `A` | At the end of the current line |
| `o` | New line below current line |
| `O` | New line above current line |

Press `Esc` to return to Command Mode.

---

### Editing in Command Mode

| Key | Action |
|---|---|
| `x` | Delete character under cursor |
| `X` | Delete character before cursor |
| `xp` | Swap two characters (fix a typo) |
| `r` | Replace one character |
| `R` | Enter Replace mode (overwrites) |
| `u` | Undo last change |
| `.` | Repeat last change |

---

### Cut, Copy, Paste

| Key | Action |
|---|---|
| `dd` | Cut (delete) current line |
| `2dd` / `3dd` | Cut next 2 or 3 lines |
| `dw` | Delete one word |
| `yy` | Copy (yank) current line |
| `3yy` | Copy next 3 lines |
| `p` | Paste below / after cursor |
| `P` | Paste above / before cursor |
| `"add` | Cut line into named buffer 'a' |
| `"ap` | Paste contents of buffer 'a' |

---

### Last Line Mode (`:`) Commands

| Command | Action |
|---|---|
| `:w` | Save (write) |
| `:q` | Quit |
| `:wq` or `:x` | Save and quit |
| `:q!` | Quit WITHOUT saving (emergency exit) |
| `:w new.txt` | Save as a new filename |
| `:/word` | Search forward for "word" |
| `:?word` | Search backward for "word" |
| `n` | Jump to next search match |
| `:%s/apple/orange/g` | Global find and replace in the whole file |
| `:r file.txt` | Insert contents of another file at cursor |
| `:r !lsblk` | Insert the output of a command into the file |
| `:set nu` | Show line numbers |
| `:set nonu` | Hide line numbers |
| `:set all` | Show all available settings |

### Editing Multiple Files

```bash
vim file1.txt file2.txt    # Open multiple files
```

Inside Vi:

| Command | Action |
|---|---|
| `:args` | List all open files |
| `:n` | Move to next file |
| `:prev` | Move to previous file |
| `:rew` | Rewind to first file |
| `:e other.txt` | Open a different file |

### Making Settings Permanent

To keep settings like line numbers on by default, create a config file:

```bash
vi ~/.vimrc
```

Add your preferences:

```
set number
set tabstop=4
```

Now every time you open a file in Vim, those settings apply automatically.

---

### Interview Q&A

**Q: How do you quit Vi without saving changes?**  
`:q!` - the `!` forces quit and discards all changes. Essential during the RHCSA exam if you accidentally break a file.

**Q: How do you copy 5 lines and paste them elsewhere?**  
Go to the first line, type `5yy`, move cursor to destination, press `p`.

**Q: How do you globally replace "apple" with "orange" in the entire file?**  
`:%s/apple/orange/g`

**Q: What is the difference between `p` and `P`?**  
`p` pastes below/after the cursor. `P` pastes above/before.

**Q: How do you insert the output of a command directly into a file while in Vi?**  
`:r !command` - for example `:r !date` inserts the current date at the cursor.

---

*Notes built while studying for RHCSA - part of the [Cloud Girl Logs](https://cloudgirllogs.hashnode.dev/) series.*
