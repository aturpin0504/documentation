---
layout: default
title: Text Processing
parent: Quick Reference
grand_parent: RHEL
nav_order: 3
---

# Text Processing

Text viewing, filtering, editing, and stream-processing tools for RHEL.

## Text Viewing Tools

| Task | Command |
|---|---|
| View a file with a pager | `less /etc/passwd` |
| Display file contents | `cat /etc/passwd` |
| Display file contents with line numbers | `cat -n /etc/hosts` |
| Display multiple files | `cat file1 file2` |
| Show the first 10 lines | `head /etc/passwd` |
| Show the first 5 lines | `head -n 5 /etc/passwd` |
| Show the first 20 lines | `head -n 20 file.txt` |
| Show a specific line | `sed -n '11p' /etc/passwd` |
| Show the last 10 lines | `tail /etc/passwd` |
| Show the last 5 lines | `tail -n 5 /etc/passwd` |
| Show the last 20 lines | `tail -n 20 file.txt` |
| Follow updates to a file | `tail -f /var/log/messages` |
| Follow a file by name after log rotation | `tail -F /var/log/messages` |

{: .tip }
> Use `less` rather than `cat` for large files. Use `q` to exit `less` or stop `tail -f` and `tail -F`.

## `less` Navigation

| Key | Action |
|---|---|
| `Space` or `Page Down` | Page down |
| `b` or `Page Up` | Page up |
| `g` | Go to the beginning of the file |
| `G` | Go to the end of the file |
| `/text` | Search forward |
| `?text` | Search backward |
| `n` | Repeat search or go to the next match |
| `N` | Go to the previous match |
| `q` | Quit |

## Count Text with `wc`

| Task | Command |
|---|---|
| Count lines, words, and bytes | `wc /etc/passwd` |
| Count lines only | `wc -l /etc/passwd` |
| Count words only | `wc -w /etc/passwd` |
| Count bytes only | `wc -c /etc/passwd` |
| Count characters only | `wc -m /etc/passwd` |

## Extract Fields with `cut`

`cut` extracts fixed character positions or delimiter-separated fields from each line.

| Task | Command |
|---|---|
| Extract field 1 using `:` as the delimiter | `cut -d : -f 1 /etc/passwd` |
| Extract fields 1 and 3 | `cut -d : -f 1,3 /etc/passwd` |
| Extract fields 1 through 3 | `cut -d : -f 1-3 /etc/passwd` |
| Extract fields 1 and 7 | `cut -d : -f 1,7 /etc/passwd` |
| Extract fields 2 through 4 from a CSV file | `cut -d , -f 2-4 data.csv` |
| Extract characters 1 through 5 | `cut -c 1-5 /etc/passwd` |
| Extract characters 1 through 10 | `cut -c 1-10 file.txt` |

{: .note }
> `cut` treats every individual space as a delimiter. When columns are separated by variable whitespace or tabs, such as `ls -l`, `ps`, or `df -h` output, use `awk` instead.

## Sorting and Deduplicating with `sort`

| Task | Command |
|---|---|
| Sort alphabetically | `sort names.txt` |
| Sort numerically | `sort -n numbers.txt` |
| Reverse sort | `sort -r names.txt` |
| Sort by field 3 using `:` as the separator | `sort -t : -k 3,3 /etc/passwd` |
| Sort field 3 numerically | `sort -t : -k 3,3n /etc/passwd` |
| Sort CPU usage from highest to lowest | <code>ps aux &#124; sort -k 3,3nr</code> |
| Sort the first field of `/etc/passwd` | <code>cut -d : -f 1 /etc/passwd &#124; sort</code> |
| Sort human-readable directory sizes, largest first | <code>du -sh * &#124; sort -hr</code> |
| Sort and remove duplicate lines | `sort -u names.txt` |
| Count and rank the most frequent lines | <code>sort access.log &#124; uniq -c &#124; sort -nr &#124; head -n 10</code> |

## `grep`: Search and Filter Text

Use `grep` to filter command output or files to lines matching a pattern.

| Task | Command |
|---|---|
| Find lines containing text | `grep "root" /etc/passwd` |
| Find lines starting with text | `grep "^root" /etc/passwd` |
| Find lines ending with text | `grep "bash$" /etc/passwd` |
| Perform a case-insensitive search | `grep -i "text" file` |
| Exclude matching lines | `grep -v "^#" file` |
| Search recursively | `grep -r "text" /etc` |
| Search for multiple patterns | `grep -e "root" -e "linda" /etc/passwd` |
| Show 5 lines after a match | `grep -A 5 "pattern" file` |
| Show 5 lines before a match | `grep -B 5 "pattern" file` |
| Show lines before and after a match | `grep -C 3 "panic" /var/log/messages` |
| Show matching line numbers | `grep -n "Port" /etc/ssh/sshd_config` |
| Count matching lines | `grep -c "pattern" file` |
| Show matching filenames only | `grep -l "pattern" file1 file2` |
| Match whole words only | `grep -w "root" /etc/passwd` |
| Use extended regular expressions | `grep -E "error|fail|crit" /var/log/messages` |
| Exclude comments and blank lines | `grep -v -e "^#" -e "^$" file` |

### Common `grep` Options

| Option | Description |
|---|---|
| `-i` | Case-insensitive matching |
| `-v` | Invert matching; exclude matching lines |
| `-r` | Search recursively |
| `-e` | Specify one or more patterns |
| `-A n` | Show `n` lines after a match |
| `-B n` | Show `n` lines before a match |
| `-C n` | Show `n` lines before and after a match |
| `-n` | Display line numbers |
| `-c` | Count matching lines |
| `-l` | Display matching filenames only |
| `-w` | Match whole words only |
| `-E` | Use extended regular expressions |
| `-F` | Treat the pattern as a fixed string rather than a regular expression |

## Regular Expressions

`grep` uses **basic regular expressions** by default. Use `grep -E` when you need extended regular-expression syntax.

### Basic Regular Expressions

| Expression | Meaning |
|---|---|
| `^text` | Line starts with `text` |
| `text$` | Line ends with `text` |
| `.` | Any single character |
| `[abc]` | Any one of `a`, `b`, or `c` |
| `[^abc]` | Any character except `a`, `b`, or `c` |
| `[0-9]` | Any digit |
| `[[:digit:]]` | Any digit; locale-aware character class |
| `[[:alpha:]]` | Any alphabetic character; locale-aware character class |
| `*` | Zero or more occurrences of the preceding expression |
| `\{2\}` | Exactly two occurrences of the preceding expression |
| `\{1,3\}` | One through three occurrences of the preceding expression |

### Extended Regular Expressions

Use these expressions with `grep -E`.

| Expression | Meaning |
|---|---|
| `?` | Zero or one occurrence of the preceding expression |
| `+` | One or more occurrences of the preceding expression |
| `*` | Zero or more occurrences of the preceding expression |
| `a|b` | Match `a` or `b` |
| `(text)` | Group an expression |
| `{2}` | Exactly two occurrences of the preceding expression |
| `{1,3}` | One through three occurrences of the preceding expression |
| `error|fail|crit` | Match `error`, `fail`, or `crit` |

## `sed`: Stream Editing and Find/Replace

Use `sed` to search and replace text, delete lines, or print selected lines without opening an editor.

Without `-i`, `sed` does not modify the source file. It writes the processed result to standard output, usually the terminal.

| Task | Command |
|---|---|
| Print line 5 | `sed -n '5p' /etc/passwd` |
| Print lines 1 through 5 | `sed -n '1,5p' /etc/passwd` |
| Print lines 10 through 20 | `sed -n '10,20p' /var/log/messages` |
| Replace the first occurrence per line | `sed 's/old/new/' file` |
| Replace all occurrences per line | `sed 's/old/new/g' file` |
| Preview an in-place replacement without changing the file | `sed 's/old/new/g' file` |
| Save edited output to a separate file | `sed 's/old/new/g' file > file-new` |
| Edit a file in place | `sed -i 's/old/new/g' file` |
| Edit in place and create a `.bak` backup | `sed -i.bak 's/old/new/g' file` |
| Use an alternate delimiter for paths | `sed 's|/var/www/html|/srv/www|g' httpd.conf` |
| Delete line 2 | `sed -i '2d' file` |
| Delete lines 2 and 20 through 25 | `sed -i '2d;20,25d' file` |
| Remove comments and blank lines from output | `sed -e '/^#/d' -e '/^$/d' file` |

### Safer File-Editing Workflow

For important files, preview the result before editing in place:

```bash
sed 's/old/new/g' file
```

To save the proposed change to a separate file for review:

```bash
sed 's/old/new/g' file > file-new
```

Compare the original and proposed versions:

```bash
diff -u file file-new
```

After reviewing the result, replace the original file:

```bash
mv file-new file
```

> **Warning:** Do not redirect `sed` output to the same file being read. The shell may truncate the file before `sed` can read it.
>
> ```bash
> # Incorrect
> sed 's/old/new/g' file > file
> ```

> **Recommendation:** For routine changes where a rollback copy is useful, use `-i.bak`:
>
> ```bash
> sed -i.bak 's/old/new/g' file
> ```
>
> This modifies `file` and retains the original content as `file.bak`.

## `awk`: Working with Columns and Fields

Use `awk` for column-based data, especially when fields are separated by variable spaces or tabs, or when filtering based on a field value.

### `awk` Field References

| Reference | Meaning |
|---|---|
| `$0` | The entire line |
| `$1`, `$2`, `$3`, and so on | First, second, third, and subsequent fields |
| `$NF` | The last field on the line |
| `NF` | The number of fields on the current line |
| `NR` | The current input line number |

| Task | Command |
|---|---|
| Print column 4 | `awk '{ print $4 }' file` |
| Print columns 1 and 5 | `awk '{ print $1, $5 }' file` |
| Print fields 1 and 7 using `:` as the delimiter | `awk -F : '{ print $1, $7 }' /etc/passwd` |
| Print a field when a line matches a pattern | `awk -F : '/user/ { print $4 }' /etc/passwd` |
| Filter records with UID 1000 or greater | `awk -F : '$3 >= 1000 { print $1, $3 }' /etc/passwd` |
| Print the last field on each line | `awk '{ print $NF }' file.txt` |
| Sum values in the first column | `awk '{ sum += $1 } END { print sum }' numbers.txt` |
| Display file system and usage percentage | <code>df -h &#124; awk '{ print $1, $5 }'</code> |
| Display selected fields from failed SSH logins | `awk '/Failed password/ { print $1, $2, $3, $11 }' /var/log/secure` |

## `grep` vs. `sed` vs. `awk`

| Tool | Primary Job | Think of It As |
|---|---|---|
| `grep` | Find lines matching a pattern | “Show me lines containing `error`.” |
| `sed` | Modify text in a stream or file | “Replace `foo` with `bar`, or delete selected lines.” |
| `awk` | Process fields, columns, and conditions | “Split lines into columns, filter by a value, and print selected fields.” |

## Editing Files with `vi`

The `vi` command opens a vi-compatible text editor. It has two primary modes:

- **Normal mode:** The default mode when `vi` opens. Use it to navigate, delete, copy, search, and execute commands.
- **Insert mode:** Use it to type text into the file.

| Task | Command |
|---|---|
| Open a file in `vi` | `vi filename` |
| Open a file in Vim, when installed | `vim filename` |

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
| `dd` | Delete the current line |
| `yy` | Yank (copy) the current line |
| `p` | Paste below the current line |
| `u` | Undo the last change |
| `Ctrl-r` | Redo |
| `v` | Enter Visual mode |
| `gg` | Go to the first line |
| `G` | Go to the last line |
| `^` | Go to the first nonblank character in the current line |
| `$` | Go to the end of the current line |
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

## Important Tips

- Use `less` rather than `cat` when reading large files.
- Use `tail -F` rather than `tail -f` when monitoring a log that may rotate.
- Use `awk`, rather than `cut`, when input uses variable whitespace as its field separator.
- Use `grep -F` when searching for literal text that might contain regular-expression characters.
- Use `grep -E` for extended regular expressions, including alternation with `|`.
- Test `sed` commands without `-i` before editing files in place.
- Use `sed -i.bak` when you want to retain a backup before an in-place edit.
- Use `vi` for basic editing available on RHEL systems; use `vim` when you specifically need Vim features.