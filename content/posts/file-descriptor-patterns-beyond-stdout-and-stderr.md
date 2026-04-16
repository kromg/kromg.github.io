---
title: "File Descriptor Patterns: Beyond stdout and stderr"
date: 2026-04-16
description: "Every process starts with three I/O channels. But Bash lets you open more — and knowing how to use them directly unlocks patterns that eliminate awkward workarounds."
tags: ["bash", "file-descriptors", "redirection", "io", "scripting"]
categories: ["Shell Scripting", "Deep Dive"]
featured_image: "/images/file-descriptors-cover.png"
---

You've already used the STDIO redirection operators: `<`, `>`, `2>`, `2>&1`, and so on.
You may also have used the appending operators: `>>` and `2>>`, and maybe the most modern ones: `&>` and `&>>`, to redirect both STDOUT and STDERR at the same time.

These operators cover most of what you need day-to-day. But have you ever wondered what they actually do — why `>` affects output, `<` affects input, and `2>` affects errors? The answer is *file descriptors*, and understanding them turns a collection of punctuation into a coherent system.

## The Three Channels — and the Numbers Behind the Operators

A file descriptor is just a number. When the kernel opens a file, a pipe, or a device for your process, it assigns the lowest available integer and hands it back. Your shell process starts with three already open, each pointing at a specific I/O channel by convention:

| Number | Name   | Default target | Operator shorthand |
|--------|--------|----------------|--------------------|
| `0`    | stdin  | keyboard       | `<`                |
| `1`    | stdout | terminal       | `>`  (same as `1>`) |
| `2`    | stderr | terminal       | `2>`               |

The operators are shorthand for *close the default target for this number and point it somewhere else*. So `> file` is exactly `1> file`, and `2>/dev/null` is "send fd 2 to the bit bucket." The number is always there, even when you don't write it.

This also explains something that is easy to take for granted: every read in your script — `read`, a pipeline's left side, a bare command waiting for input — implicitly uses fd `0`. Every write — `echo`, `printf`, the output of any command — goes implicitly to fd `1`. Redirecting those two numbers is what gives `< file` and `> file` their effect. The programs involved don't change; only where their standard channels point changes.

Stderr (fd `2`) is a different story. Nothing in the kernel *requires* programs to write errors there — it is a convention 
that Unix tools follow. 
When every tool respects it, you can separate output from errors cleanly: `./script.sh > output.txt 2> errors.txt`¹.  
When a tool doesn't, its error messages bleed into stdout, and it becomes impossible to discard the error messages if you don't need them.

Numbers `3` and above are yours to open as needed. How many? The per-process limit is controlled by `ulimit -n`; on most modern Linux systems the default is `1024`, though it can be raised. For scripting purposes you'll rarely use more than a handful of extra descriptors.

> ¹ Separating STDOUT and STDERR into different files may seem like a clever idea, until you realize you lose the correlation between the errors and the point in the
execution flow where they happen. I've warned you, do it at your own risk.

## Opening and Closing File Descriptors with `exec`

The shell built-in `exec` is the mechanism for opening and closing file descriptors in the current shell. When used without a command — just a redirection — it applies that redirection to the running shell process itself, permanently, until you close it or the shell exits:

```bash
exec 3> debug.log    # open fd 3, pointing at debug.log
exec 3>&-            # close fd 3
```

Bash also supports an *anonymous fd* form, where Bash picks a free number and stores it in a variable of your choosing:

```bash
exec {log_fd}> debug.log    # Bash picks a number, stores it in log_fd
echo "detail" >&"$log_fd"   # write to it
exec {log_fd}>&-            # close it when done
```

The anonymous form is preferable in almost every case: it avoids accidentally reusing a number you've already opened, and the variable name makes the purpose self-documenting. Reading works the same way, with `<` instead of `>`, and you close a read fd with `<&-`:

```bash
exec {input_fd}< data.txt
read -r line <&"$input_fd"
exec {input_fd}<&-
```

**Close file descriptors when you are done with them.** An unclosed fd keeps the underlying file or pipe open: it can delay log file finalization, prevent the read end of a pipe from receiving EOF (leaving the other process waiting indefinitely), or slowly exhaust the fd limit in long-running scripts. The close syntax is symmetric and easy to remember:

```bash
exec {my_fd}>&-    # close a write fd
exec {my_fd}<&-    # close a read fd
```

There is another reason closing matters: file descriptors are inherited by child processes. Every command your script runs, every subshell it spawns, starts with a copy of the parent's fd table. This is precisely why commands inside your script can read from stdin and write to stdout without any setup — they inherit those connections automatically. Any extra fds you open behave the same way: subprocesses will inherit them too, unless you explicitly close them first. A subprocess holding an inherited fd it knows nothing about can silently keep a pipe open or hold a lock on a file far longer than intended. That makes closing descriptors promptly not just good hygiene, but a correctness concern.

{{< book-hook >}}

## Saving and Restoring Output

The most broadly useful fd pattern is temporarily redirecting stdout or stderr and then restoring them — without spawning a subshell.

Subshells are the common approach: wrap a block in `(...)` and the redirection disappears when the subshell exits. But subshells have their own scope, so any variable assignments inside are lost. Using saved descriptors avoids that entirely:

```bash
# Save fd 1 before redirecting
exec {saved_stdout}>&1
exec 1>/dev/null

echo "This goes nowhere"

# Restore
exec 1>&"$saved_stdout"
exec {saved_stdout}>&-

echo "This appears normally"
```

The key step is `exec {saved_stdout}>&1`: it duplicates fd 1 into a new descriptor. After redirecting stdout to `/dev/null`, the saved copy still points at your terminal. Restoring it is a single line.

This pattern is especially useful inside functions that need to silence their own output without affecting the caller's environment, since functions run in the current shell (not a subshell) and share fd state with the rest of the script.

The same idea composes naturally into a reusable wrapper. Here is a `silently` function that suppresses all output from any command passed to it, while still propagating its exit status to the caller:

```bash
silently() {
  exec {saved_stdout}>&1 {saved_stderr}>&2
  exec 1>/dev/null 2>/dev/null
  "$@"
  local status=$?
  exec 1>&"$saved_stdout" 2>&"$saved_stderr"
  exec {saved_stdout}>&- {saved_stderr}>&-
  return "$status"
}

# usage
silently some_command --with args || echo "command failed" >&2
```

The two `exec` lines before `"$@"` save both channels and redirect them to `/dev/null` in one step. After the command returns, the saved descriptors are restored and then closed. Capturing the exit status in `status` before restoring the fds ensures the `return` reflects the command's outcome, not the result of the `exec` calls. The caller can then handle success or failure normally, as if `silently` were not there.

{{< book-hook >}}

## The Verbose Mode Pattern

Here is a pattern worth keeping in your toolkit: a togglable verbose channel that requires no `if` guards scattered through your code.

The idea is simple. At startup, open a custom fd pointing at `/dev/null` (quiet mode) or at stdout (verbose mode). Then write all diagnostic output to that fd throughout the script. The single choice at startup controls everything:

```bash
#!/usr/bin/env bash

verbose=0
[[ "${1-}" == "--verbose" ]] && verbose=1

if (( verbose )); then
  exec {verbose_fd}>&1        # verbose: point at stdout
else
  exec {verbose_fd}>/dev/null # quiet: swallow everything
fi

echo "Starting process..."
printf 'DEBUG: pid=%d, user=%s\n' "$$" "$USER" >&"$verbose_fd"

# ... rest of the script ...

exec {verbose_fd}>&-
```

Run without flags: only "Starting process..." appears. Run with `--verbose`: the debug line appears too. The fd does the switching — no conditionals required at each message site.

This scales cleanly: adding more diagnostic messages means writing one more `>&"$verbose_fd"` line, nothing else.


## Writing to Terminal and File Simultaneously

Another practical pattern: you want output to appear on the terminal *and* be saved to a log file at the same time. Process substitution with `tee` makes this a one-liner:

```bash
#!/usr/bin/env bash

log_file="run-$(date +%Y%m%d-%H%M%S).log"

# Redirect stdout through tee: one copy to terminal, one to file
exec 1> >(tee -a "$log_file")

echo "This appears on the terminal and in $log_file"
echo "So does this"
```

The `>(tee -a "$log_file")` is a *process substitution* — it opens a write-end connected to `tee`'s stdin. After `exec 1> >(...)`, every write to stdout flows through `tee`, which copies it to both its own stdout (your terminal) and the log file.

One thing to keep in mind: `tee` runs as a background process. If your script exits immediately after the last write, the final output might not have been flushed to the log yet. Closing fd `1` explicitly before exit signals to `tee` that the input stream is done, giving it time to flush and exit cleanly — this is exactly the habit introduced earlier. Process management and background jobs deserve their own post; for now, just remember that explicitly closing an fd is always the right move before your script finishes.

## The Bigger Picture

File descriptors are one of the cleaner pieces of Unix's design: every I/O resource gets a uniform interface, and every tool uses the same small integers to represent open connections. Bash exposes them directly — and once you're comfortable using them, patterns like verbose mode, save-and-restore, and dual logging stop feeling like tricks and start feeling like natural tools.

The underlying chapter in *Bash: The Developer's Approach* covers fd mechanics in full — including `exec`-based persistent redirections, the `<>` read-write operator, anonymous fds, and the subtleties of file offsets.
