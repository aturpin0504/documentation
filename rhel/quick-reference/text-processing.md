---
layout: default
title: Text Processing
parent: Quick Reference
grand_parent: RHEL
---

# Text Processing

Daily text viewing, filtering, editing, and stream-processing tools for RHEL.

## Text Viewing Tools

### Viewing File Contents

| Task | Command |
|---|---|
| View file with a pager | `less /etc/passwd` |
| Display file contents | `cat /etc/passwd` |
| Display file contents with line numbers | `cat -n /etc/hosts` |
| Display multiple files | `cat file1 file2` |
| Show first 10 lines | `head /etc/passwd` |
| Show first 5 lines | `head -5 /etc/passwd` |
| Show first 20 lines | `head -n 20 file.txt` |
| Show line 11 | `head -11 /etc/passwd \| tail -1` |
| Show last 10 lines | `tail /etc/passwd` |
| Show last 5 lines | `tail -5 /etc/passwd` |
| Show last 20 lines | `tail -n 20 file.txt` |
| Follow file updates | `tail -f /var/log/messages` |

```bash
cat /etc/hosts              # Print an entire file to standard output
cat -n /etc/hosts           # Print an entire file with line numbers

head /var/log/messages      # Show the first 10 lines by default
head -n 20 file.txt         # Show the first 20 lines
head -20 file.txt           # Shortcut for head -n 20

tail /var/log/messages      # Show the last 10 lines by default
tail -n 20 file.txt         # Show the last 20 lines
tail -20 file.txt           # Shortcut for tail -n 20
tail -f /var/log/messages   # Follow a log file in real time; use Ctrl+C to stop
```

## `less` Navigation

| Key | Action |
|---|---|
| `Space` / `Page Down` | Page down |
| `b` / `Page Up` | Page up |
| `g` | Go to the beginning of the file |
| `G` | Go to the end of the file |
| `/text` | Search forward |
| `?text` | Search backward |
| `n` | Repeat search / next match |
| `q` | Quit |

## Count Text with `wc`

| Task | Command |
|---|---|
| Count lines, words, and characters | `wc /etc/passwd` |
| Count lines only | `wc -l /etc/passwd` |
| Count words only | `wc -w /etc/passwd` |
| Count characters only | `wc -c /etc/passwd` |

## Extract Fields with `cut`

`cut` extracts fixed character positions or delimiter-separated fields from each line.

| Task | Command |
|---|---|
| Extract field 1 using `:` as delimiter | `cut -d : -f 1 /etc/passwd` |
| Extract fields 1 and 3 | `cut -d : -f 1,3 /etc/passwd` |
| Extract fields 1 through 3 | `cut -d : -f 1-3 /etc/passwd` |
| Extract characters 1 through 5 | `cut -c 1-5 /etc/passwd` |

```bash
cut -d: -f1,7 /etc/passwd        # Use ':' as delimiter and print fields 1 and 7
cut -d, -f2-4 data.csv           # Print fields 2 through 4 from a CSV
cut -c1-10 file.txt              # Print characters 1 through 10 from each line
```

> [!NOTE]
> `cut` treats every individual space as a separate delimiter. If columns are separated by variable spaces or tabs—such as `ls -l` or `df -h` output—use `awk` instead.

## Sorting and Deduplicating with `sort`

| Task | Command |
|---|---|
| Sort alphabetically | `sort /etc/passwd` |
| Sort numerically | `sort -n file` |
| Reverse sort | `sort -r file` |
| Sort by field 3 using `:` as separator | `sort -k3 -t : /etc/passwd` |
| Sort field 3 numerically in reverse order | `ps aux \| sort -k3 -rn` |
| Sort the first field of `/etc/passwd` | `cut -f 1 -d : /etc/passwd \| sort` |
| Sort by size, largest first | `du -h \| sort -rn` |
| Sort and remove duplicate lines | `sort -u names.txt` |
| Sort human-readable sizes | `du -sh * \| sort -h` |

```bash
sort names.txt                   # Sort alphabetically
sort -r names.txt                # Reverse sort
sort -n numbers.txt              # Numeric sort: 1, 2, 10 instead of 1, 10, 2
sort -u names.txt                # Sort and remove duplicate lines
sort -t: -k3 -n /etc/passwd      # Sort numerically by field 3 using ':' as the delimiter
```

Count and rank the most frequent lines:

```bash
sort access.log | uniq -c | sort -nr | head -10
```

## `grep`: Search and Filter Text

Use `grep` to filter command output or files to lines matching a pattern.

| Task | Command |
|---|---|
| Find lines containing text | `grep root /etc/passwd` |
| Find lines starting with text | `grep '^root' /etc/passwd` |
| Find lines ending with text | `grep 'bash$' /etc/passwd` |
| Case-insensitive search | `grep -i text file` |
| Exclude matching lines | `grep -v '^#' file` |
| Recursive search | `grep -r text /etc` |
| Search multiple patterns | `grep -e root -e linda /etc/passwd` |
| Show 5 lines after a match | `grep -A 5 pattern file` |
| Show 5 lines before a match | `grep -B 5 pattern file` |
| Show matching line numbers | `grep -n "Port" /etc/ssh/sshd_config` |
| Count matching lines | `grep -c pattern file` |
| Show only matching filenames | `grep -l pattern files` |
| Match whole words only | `grep -w "root" /etc/passwd` |
| Use extended regular expressions | `grep -E "error\|fail\|crit" /var/log/messages` |
| Show context before and after a match | `grep -C 3 "panic" /var/log/messages` |

### Common `grep` Options

| Option | Description |
|---|---|
| `-i` | Case-insensitive matching |
| `-v` | Invert match; exclude matching lines |
| `-r` | Search recursively |
| `-e` | Specify multiple patterns |
| `-A n` | Show `n` lines after a match |
| `-B n` | Show `n` lines before a match |
| `-C n` | Show `n` lines before and after a match |
| `-n` | Display line numbers |
| `-c` | Count matching lines |
| `-l` | Display matching filenames only |
| `-w` | Match whole words only |
| `-E` | Use extended regular expressions |

```bash
grep "error" /var/log/messages           # Find case-sensitive matches
grep -i "error" /var/log/messages        # Find case-insensitive matches
grep -n "Port" /etc/ssh/sshd_config      # Include line numbers
grep -v "^#" /etc/ssh/sshd_config        # Show lines not beginning with '#'
grep -r "10.0.0.5" /etc/                 # Recursively search a directory
grep -w "root" /etc/passwd               # Match whole words only
grep -E "error|fail|crit" /var/log/messages  # Match one of several patterns
grep -C 3 "panic" /var/log/messages      # Show three context lines before and after
```

## Regular Expressions

| Expression | Meaning |
|---|---|
| `^text` | Line starts with `text` |
| `text$` | Line ends with `text` |
| `.` | Any single character |
| `[abc]` | Any one of `a`, `b`, or `c` |
| `[^abc]` | Any character except `a`, `b`, or `c` |
| `*` | Zero or more occurrences of the preceding expression |
| `\{2\}` | Exactly two occurrences of the preceding expression |
| `\{1,3\}` | One to three occurrences of the preceding expression |
| `?` | Zero or one occurrence of the preceding expression |

### Character Classes

| Class | Meaning |
|---|---|
| `[0-9]` | Any digit |
| `[a-z]` | Any lowercase letter |
| `[A-Z]` | Any uppercase letter |
| `[a-zA-Z]` | Any letter |
| `[a-zA-Z0-9]` | Any alphanumeric character |

## `sed`: Stream Editing and Find/Replace

Use `sed` to search and replace text, delete lines, or extract selected lines without opening an editor.

| Task | Command |
|---|---|
| Print line 5 | `sed -n 5p /etc/passwd` |
| Print lines 1 through 5 | `sed -n '1,5p' /etc/passwd` |
| Replace the first occurrence per line | `sed 's/old/new/' file` |
| Replace all occurrences per line | `sed 's/old/new/g' file` |
| Edit a file in place | `sed -i 's/old/new/g' file` |
| Delete line 2 | `sed -i -e '2d' file` |
| Delete multiple lines | `sed -i -e '2d;20,25d' file` |

> [!TIP]
> ### Preview Before Using `sed -i`
>
> By default, `sed` writes its output to standard output without modifying the source file. Test the command without `-i` first, then add `-i` to edit in place. Use `-i.bak` if you want a backup copy before editing.

```bash
# Replace the first match on each line; output only
sed 's/old/new/' file.txt

# Replace all matches on each line
sed 's/old/new/g' file.txt

# Edit in place and retain a .bak backup
sed -i.bak 's/old/new/g' file.txt

# Use an alternate delimiter for file paths containing '/'
sed 's|/var/www/html|/srv/www|g' httpd.conf

# Delete comments and blank lines
sed -e '/^#/d' -e '/^$/d' /etc/ssh/sshd_config

# Print lines 10 through 20 only
sed -n '10,20p' /var/log/messages
```

## `awk`: Working with Columns and Fields

Use `awk` for column-based data, especially when fields are separated by variable spaces or tabs, or when filtering based on a particular field.

### `awk` Field References

| Reference | Meaning |
|---|---|
| `$0` | The entire line |
| `$1`, `$2`, `$3`, ... | First, second, third, and subsequent fields |
| `$NF` | The last field on the line |

| Task | Command |
|---|---|
| Print column 4 | `awk '{ print $4 }' file` |
| Print field 4 using `:` as delimiter | `awk -F : '{ print $4 }' /etc/passwd` |
| Print field 4 when a line matches a pattern | `awk -F : '/user/ { print $4 }' /etc/passwd` |

```bash
# Print columns 1 and 5 from space- or tab-separated output
df -h | awk '{print $1, $5}'

# Specify a custom delimiter
awk -F: '{print $1, $7}' /etc/passwd

# Match a pattern and print selected fields
awk '/Failed password/ {print $1, $2, $3, $11}' /var/log/secure

# Filter records where UID is 1000 or greater
awk -F: '$3 >= 1000 {print $1, $3}' /etc/passwd

# Print the last field on each line
awk '{print $NF}' file.txt

# Sum values in the first column
awk '{sum += $1} END {print sum}' numbers.txt
```

## `grep` vs. `sed` vs. `awk`

| Tool | Primary Job | Think of It As |
|---|---|---|
| `grep` | Find lines matching a pattern | “Show me lines containing `error`.” |
| `sed` | Modify text in a stream or file | “Replace `foo` with `bar`, or delete matching lines.” |
| `awk` | Process fields, columns, and conditions | “Split lines into columns, filter by a value, and print selected fields.” |

## Editing Files with `vi`

`vi` is installed on every RHEL system. It has two primary modes:

- **Normal (Command) mode:** The default mode when `vi` opens. Use it to navigate, delete, copy, search, and execute commands.
- **Insert mode:** Use it to type text into the file.

Open a file:

```bash
vi filename
```

### Entering and Exiting Insert Mode

| Key | Action |
|---|---|
| `i` | Enter Insert mode before the cursor |
| `a` | Enter Insert mode after the cursor |
| `o` | Open a new line below the cursor and enter Insert mode |
| `Esc` | Return to Normal mode |

### Common Normal-Mode Commands

| Key | Action |
|---|---|
| `dd` | Delete (cut) the current line |
| `yy` | Yank (copy) the current line |
| `p` | Paste below the current line |
| `u` | Undo the last change |
| `Ctrl-r` | Redo |
| `v` | Enter Visual mode |
| `gg` | Go to the first line |
| `G` | Go to the last line |
| `^` | Go to the first position in the current line |
| `$` | Go to the last position in the current line |
| `/text` | Search forward |
| `?text` | Search backward |
| `n` | Go to the next search result |
| `N` | Go to the previous search result |

### Ex Mode Commands

Run these commands from Normal mode by typing `:` first.

| Task | Command |
|---|---|
| Save changes | `:w` |
| Save with a new filename | `:w filename` |
| Quit | `:q` |
| Quit without saving | `:q!` |
| Save and quit | `:wq` or `:x` |
| Replace all occurrences in the file | `:%s/old/new/g` |
| Show line numbers | `:set number` or `:set nu` |
| Hide line numbers | `:set nonumber` or `:set nonu` |
| Execute a shell command | `:!command` |

## Practical Examples

```bash
# Show active SSH configuration lines by excluding comments and blank lines
grep -v -e '^#' -e '^$' /etc/sysconfig/sshd

# Display the highest CPU-consuming processes
ps aux | sort -k3 -rn | head

# Display the highest memory-consuming processes
ps aux | sort -k4 -rn | head

# List local users alphabetically
cut -d : -f 1 /etc/passwd | sort

# Follow a system log in real time
tail -f /var/log/messages
```

## Important Tips

- Test `sed` commands without `-i` before editing files in place.
- Use `tail -f` to monitor log-file updates in real time.
- Use `awk`, rather than `cut`, when input uses variable whitespace as its field separator.
