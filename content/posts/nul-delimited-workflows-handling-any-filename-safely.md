---
title: "NUL-Delimited Workflows: Handling Any Filename Safely"
date: 2026-03-23
description: "Filenames with spaces are annoying. Filenames with newlines are dangerous. Learn the one delimiter that makes any filename safe to process—and how to use it throughout your pipelines."
tags: ["bash", "filenames", "security", "best-practices"]
categories: ["Shell Scripting", "Deep Dive"]
---

Pop quiz: what characters can appear in a filename on Linux?

If you answered "letters, numbers, dots, dashes, underscores," you're thinking of *sensible* filenames. The actual answer is: **every character except `/` (path separator) and NUL (the zero byte)**.

That includes spaces. Tabs. Newlines. Quote marks. Backslashes. Asterisks. Emoji. Control characters. Bell characters that make your terminal beep. All perfectly legal in filenames.

And that's a problem, because most UNIX tools assume newlines separate records. When a filename *contains* a newline, tools get confused:

```bash
# Create a file with a newline in its name (yes, this is legal)
touch $'invoice\n2024.pdf'

# Try to list it with ls and process line by line
ls *.pdf | while read -r file; do
  printf 'Processing: [%s]\n' "$file"
done
```

Output:
```
Processing: [invoice]
Processing: [2024.pdf]
```

One file became two. Any script that processes filenames line-by-line is vulnerable to this—whether the weird filenames are accidental, or deliberately crafted by an attacker.


## The Solution: NUL-Delimited Everything

There's exactly one byte that cannot appear in a filename: the NUL byte (`\0`, ASCII 0). If you use NUL as your delimiter instead of newlines, filename processing becomes unambiguous. No matter what characters a filename contains, a NUL-terminated record captures it correctly.

The key tools:

**`find -print0`** outputs filenames terminated with NUL instead of newline:
```bash
find . -name '*.pdf' -print0
```

**`xargs -0`** reads NUL-terminated input:
```bash
find . -name '*.log' -print0 | xargs -0 wc -l
```

**`read -d ''`** in Bash reads until NUL instead of newline:
```bash
while IFS= read -r -d '' file; do
  printf 'Processing: [%s]\n' "$file"
done < <(find . -name '*.pdf' -print0)
```

Let's try that problematic file again:

```bash
touch $'invoice\n2024.pdf'

while IFS= read -r -d '' file; do
  printf 'Processing: [%s]\n' "$file"
done < <(find . -name '*.pdf' -print0)
```

Output:
```
Processing: [invoice
2024.pdf]
```

One file, correctly captured—newline and all.


{{< book-hook >}}


## The Canonical Pattern

This is the safe, robust way to process files in Bash:

```bash
while IFS= read -r -d '' file; do
  # "$file" is guaranteed to be exactly one filename
  some_command "$file"
done < <(find /path -type f -name '*.log' -print0)
```

Let's break down each piece:

- **`find -print0`**: Output NUL-terminated filenames
- **`< <(...)`**: Process substitution, feeding find's output to the loop's stdin
- **`IFS=`**: Don't split on spaces/tabs (preserve leading/trailing whitespace)
- **`read -r`**: Don't interpret backslashes
- **`-d ''`**: Read until NUL instead of newline
- **`"$file"`**: Always quote when using the variable

Every piece matters. Skip `IFS=` and leading spaces get trimmed. Skip `-r` and backslashes get interpreted. Skip `-d ''` and newlines break your logic. Skip the quotes and word splitting strikes again.


## Building NUL-Safe Pipelines

Once you're producing NUL-delimited output, you need tools that can consume it. Fortunately, GNU coreutils provides `-z` or `-0` flags for many common tools:

**Sorting filenames:**
```bash
find . -type f -print0 | sort -z
```

**Removing duplicates:**
```bash
find . -name '*.log' -print0 | sort -z | uniq -z
```

**Parallel processing:**
```bash
find . -name '*.jpg' -print0 | xargs -0 -P 4 -I {} convert {} -resize 800x600 resized/{}
```

**Creating archives:**
```bash
find ./src -type f -print0 | tar --null -T - -cvf backup.tar
```


## When Newlines Sneak Back In

The tricky part is maintaining NUL-safety through your entire pipeline. One tool that doesn't understand NUL delimiters can break the chain:

```bash
# BROKEN: grep doesn't preserve NUL delimiters by default
find . -type f -print0 | grep 'log' | xargs -0 wc -l
```

The `grep` in the middle outputs newline-delimited results, breaking the NUL chain. On GNU systems, use `grep -z`:

```bash
# Fixed: grep -z reads and outputs NUL-delimited
find . -type f -print0 | grep -z 'log' | xargs -0 wc -l
```

Always check that *every* tool in your pipeline supports NUL delimiters, or you'll have a weak link.


## Loading Into Arrays

Sometimes you need random access to a list of files, not just sequential processing. Bash 4.4+ provides `mapfile` with `-d ''`:

```bash
mapfile -t -d '' files < <(find . -name '*.txt' -print0)

printf 'Found %d files\n' "${#files[@]}"

for file in "${files[@]}"; do
  printf 'File: %s\n' "$file"
done
```

The `-t` strips the delimiter (NUL) from each element. The `-d ''` sets NUL as the delimiter. The result is an array where each element is exactly one filename, no matter what characters it contains.


{{< book-hook >}}


## Debugging NUL-Delimited Output

NUL bytes are invisible, which makes debugging tricky. A few techniques:

**Make NUL visible with `cat -v`:**
```bash
find . -name '*.txt' -print0 | cat -v
# Output: ./file1.txt^@./file2.txt^@
# (^@ represents NUL)
```

**Or use `od` for hex output:**
```bash
find . -name '*.txt' -print0 | od -c | head
```

**Count records:**
```bash
find . -type f -print0 | tr -cd '\0' | wc -c
# Counts NUL bytes = number of files
```


## When You Don't Need NUL Delimiters

NUL-safety is essential when:
- Processing user-provided filenames
- Handling files from untrusted sources
- Building tools that must work with any input
- Writing code that runs in production

But it's overkill when:
- You control the filenames completely (e.g., programmatically generated with strict naming)
- You're doing a quick one-off in a directory you know
- Performance is critical and filenames are trusted (NUL processing has slight overhead)

The key is knowing *when* you need the safety. Use your judgment.


## A Complete Example

Here's a script that safely processes files, demonstrating the full pattern. (The `die()` function is the error-first pattern from [a previous post](/posts/error-first-pattern-writing-self-documenting-bash/)—a Perl-inspired idiom for clean error handling.)

```bash
#!/usr/bin/env bash

die() { printf '%s\n' "$*" >&2; exit 1; }

[[ -n "${1:-}" ]] || die "Usage: $0 <directory>"
search_dir=$1

[[ -d "$search_dir" ]] || die "Not a directory: $search_dir"

# Safely collect all .log files
count=0
total_lines=0

while IFS= read -r -d '' file; do
  lines=$(wc -l < "$file")
  printf '%6d lines: %s\n' "$lines" "$file"
  (( count++ ))
  (( total_lines += lines ))
done < <(find "$search_dir" -type f -name '*.log' -print0)

printf '\nProcessed %d files, %d total lines\n' "$count" "$total_lines"
```

This handles any filename—spaces, tabs, newlines, quotes, globs, whatever. It's defensive by default.


## The Bottom Line

Newline-delimited filename processing is a landmine. It works fine until it doesn't, and when it breaks, the failure mode can be confusing or dangerous.

NUL-delimited processing is bulletproof. The pattern is slightly more verbose, but it eliminates an entire class of bugs. Once you learn `find -print0` and `read -d ''`, you'll use them everywhere.

Your future self (and anyone who inherits your scripts) will thank you.


