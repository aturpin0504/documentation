---
layout: default
title: Text Processing
parent: Quick Reference
grand_parent: RHEL
---

# Text Processing

Daily text viewing, filtering, editing, and stream-processing tools for RHEL.

---

## Viewing File Contents (`cat`, `head`, `tail`)

```bash
cat /etc/hosts           # Print entire file to stdout
cat -n /etc/hosts        # Print file with line numbers

head /var/log/messages   # Show the first 10 lines (default)
head -n 20 file.txt      # Show the first 20 lines
head -20 file.txt        # Shortcut for head -n 20

tail /var/log/messages   # Show the last 10 lines (default)
tail -n 20 file.txt      # Show the last 20 lines
tail -20 file.txt        # Shortcut for tail -n 20
tail -f /var/log/messages  # Follow a log file in real time (Ctrl+C to stop)
```

---

## Extracting Columns with `cut`

`cut` extracts fixed character positions or delimiter-separated fields from each line.

```bash
cut -d: -f1,7 /etc/passwd        # Use ':' as delimiter (-d) and print fields 1 and 7 (-f)
cut -d, -f2-4 data.csv           # Print fields 2 through 4 from a CSV
cut -c1-10 file.txt              # Print characters 1 through 10 of each line
```

> [!NOTE]
> **`cut` vs. `awk` on whitespace**
> `cut` treats every single space as a separate delimiter. If columns are separated by **varying numbers of spaces or tabs** (like `ls -l` or `df -h`), use [`awk`](#awk-working-with-columns-fields) instead.

---

## Sorting & Deduplicating (`sort`)

```bash
sort names.txt                   # Sort alphabetically
sort -r names.txt                # Reverse sort
sort -n numbers.txt              # Numeric sort (1, 2, 10 instead of 1, 10, 2)
sort -h                          # Human-readable numeric sort (e.g., du -sh * | sort -h)
sort -u names.txt                # Sort and remove duplicate lines
sort -t: -k3 -n /etc/passwd      # Sort by field 3 (-k3) numerically (-n) using ':' delimiter (-t:)
```

**Count & rank most frequent lines:**

```bash
sort access.log | uniq -c | sort -nr | head -10
```

---

## Editing Files with `vi`

`vi` is installed on every RHEL system. It has two primary modes you switch between constantly:

- **Normal (Command) mode** — Navigate, delete, copy, and run commands (default when `vi` opens).
- **Insert mode** — Type text into the file.

### Entering & Exiting Insert Mode

| Key | Action |
| :--- | :--- |
| `i` | Enter **Insert mode** before the cursor |
| `o` | Open a new line **below** the cursor and enter **Insert mode** |
| `Esc` | Return to **Normal (Command) mode** |

### Common Normal-Mode Commands

| Key | Action |
| :--- | :--- |
| `dd` | Delete (cut) the current line |
| `yy` | Yank (copy) the current line |
| `p` | Paste below the current line |
| `u` | Undo last change |
| `Ctrl+r` | Redo |
| `gg` / `G` | Jump to the top / bottom of the file |
| `/<pattern>` | Search forward (`n` next match, `N` previous match) |

### Colon (`:`) Commands (Run from Normal Mode)

| Command | Action |
| :--- | :--- |
| `:w` | Save (write) changes |
| `:q` | Quit (fails if there are unsaved changes) |
| `:q!` | Quit **without** saving (discard changes) |
| `:x` | Save and quit (only writes if the file was modified) |
| `:set nu` | Show line numbers |
| `:set nonu` | Hide line numbers |

---

## `grep` vs. `sed` vs. `awk` — Which Tool Do I Need?

| Tool | Primary Job | Think of it as… |
| :--- | :--- | :--- |
| **`grep`** | **Find lines** that match a pattern | "Show me the lines containing `error`." |
| **`sed`** | **Modify text** in a stream or file | "Find `foo` and replace it with `bar`, or delete matching lines." |
| **`awk`** | **Work with columns/fields** and conditions | "Split each line into columns, filter by a column value, and print `$1` and `$3`." |

---

## `grep` — Searching for Lines

Use `grep` whenever you want to **filter output or files down to lines matching a pattern**.

```bash
grep "error" /var/log/messages           # Lines containing "error" (case-sensitive)
grep -i "error" /var/log/messages        # Case-insensitive search (-i)
grep -n "Port" /etc/ssh/sshd_config      # Show matching line numbers (-n)
grep -v "^#" /etc/ssh/sshd_config        # Invert match (-v): show lines NOT starting with '#'
grep -r "10.0.0.5" /etc/                 # Recursive search (-r) across a directory
grep -w "root" /etc/passwd               # Match whole word only (-w)
grep -E "error|fail|crit" /var/log/messages  # Extended regex (-E) for OR patterns
grep -C 3 "panic" /var/log/messages      # Show 3 lines of context before & after (-C; or -A/-B)
```

---

## `sed` — Stream Editing & Find-and-Replace

Use `sed` when you want to **search-and-replace text** or **delete/extract specific lines** without opening `vi`.

> [!TIP]
> **Preview before using `-i`**
> `sed` prints to `stdout` by default without changing the file. Run it without `-i` first to verify your output, then add `-i` (or `-i.bak` to save a backup) to edit in place.

```bash
# Replace the first match on each line (stdout only)
sed 's/old/new/' file.txt

# Replace ALL matches on each line (g = global)
sed 's/old/new/g' file.txt

# Edit the file in place (-i) and save a .bak backup (-i.bak)
sed -i.bak 's/old/new/g' file.txt

# Use '|' as delimiter when replacing file paths containing '/'
sed 's|/var/www/html|/srv/www|g' httpd.conf

# Delete lines matching a pattern (e.g., strip comments and blank lines)
sed -e '/^#/d' -e '/^$/d' /etc/ssh/sshd_config

# Print only lines 10 through 20 (-n suppresses default output, p prints)
sed -n '10,20p' /var/log/messages
```

---

## `awk` — Working with Columns & Fields

Use `awk` when your data is organized in **columns** (especially when separated by variable spaces/tabs), or when you want to **filter based on a specific column**.

`awk` automatically splits each line into fields:
- `$0` = the entire line
- `$1`, `$2`, `$3`… = column 1, column 2, column 3…
- `$NF` = the **last** column on the line

```bash
# Print columns 1 and 5 from space/tab-separated output
df -h | awk '{print $1, $5}'

# Specify a custom delimiter (-F)
awk -F: '{print $1, $7}' /etc/passwd

# Combine pattern matching + column printing (replaces grep | awk)
awk '/Failed password/ {print $1, $2, $3, $11}' /var/log/secure

# Filter by a numeric condition on a column (UID >= 1000)
awk -F: '$3 >= 1000 {print $1, $3}' /etc/passwd

# Print the last column of each line
awk '{print $NF}' file.txt

# Sum numbers in column 1 across all lines
awk '{sum += $1} END {print sum}' numbers.txt
```