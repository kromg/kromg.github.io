---
title: "The Disappearing Script: Shebang Tricks and Magic Numbers"
date: 2026-04-30
description: "The shebang line is more than a convention — it's parsed by the kernel before any shell is involved. Understanding how it works reveals some surprising tricks, including a script that deletes itself before a single line of code runs."
tags: ["bash", "shebang", "scripting", "linux", "exec"]
categories: ["Shell Scripting", "Fun/Wildcard"]
featured_image: "/images/shebang-cover.png"
---

Open any script and the first line is probably `#!/usr/bin/env bash`. You've typed it so many times it's muscle memory. The cursor lands, you type the shebang, and you move on.

But here's a question worth pausing on: who actually reads that line?

Not the shell. Not Bash. The *kernel*.

## The Magic Bytes

Ok, ok... that's not completely true. If you run a script specifying the shell you want to use, as in: 
```bash
bash script.sh
```

or source it, then nothing special happens. The line starts with a `#`, so for any shell that's just a comment line, and it will 
happily ignore it. End of the story. 

But if you execute the script directly: 
```bash
./script.sh
```

Then the oeprating system is asked to execute the file via `execve()`. The very first thing it does is look at the first two bytes. 
If they're `0x23 0x21` — the ASCII codes for `#` and `!` — the kernel reads the rest of that first line to find the interpreter path. 
It then launches that interpreter, passing your script as an argument. 
At this point, we're back to the first case: the kernel calls `execve()` with whatever's in the shebang line, and the script path. 
If the script starts with: `#!/usr/bin/env bash`, then the kernel will invoke: 

```bash
/usr/bin/env bash ./script.sh
```

and Bash will ignore the first line and move on. 

{{< book-hook >}}

## `/bin/bash` vs `/usr/bin/env bash`

Two forms are in common use:

```bash
#!/bin/bash
```

```bash
#!/usr/bin/env bash
```

The first hardcodes the path to the Bash binary. If Bash lives somewhere else — a different Linux distribution, macOS with Homebrew, certain enterprise environments — the script fails at the shebang.

The second uses `env` to search your `PATH` for `bash`. Whatever version the user's environment has configured gets used,
without any assumption about directory layout — which is usually the right call, since it lets users point to their preferred
Bash version simply by managing their `PATH`.


## The One-Argument Problem

Shebangs look like they could accept multiple arguments:

```bash
#!/usr/bin/env bash -euo pipefail
```

On Linux, this silently doesn't work as intended. The kernel passes everything after the interpreter path as a *single string* to `env` — so `env` receives `"bash -euo pipefail"` as a filename to look up, which doesn't exist.

macOS (and some BSDs) split the argument string; Linux doesn't. The same shebang behaves differently on different platforms, which is its own kind of problem.

The fix: GNU `env` gained a `-S` flag that tells it to split the following argument on whitespace:

```bash
#!/usr/bin/env -S bash -euo pipefail
```

This works on Linux systems with a recent enough `coreutils`. In practice, most scripts avoid the problem entirely by setting options in the script body:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

This is the most portable and readable approach.

## Shebangs Beyond Bash

Since the shebang mechanism is handled by the kernel and works for any executable, it extends to any interpreter:

```bash
#!/usr/bin/env python3
print("This is Python, running with a shebang")
```

```bash
#!/usr/bin/env node
console.log("This is Node.js, running with a shebang");
```

AWK has a slightly different form, since AWK scripts are passed via the `-f` flag:

```bash
#!/usr/bin/awk -f
BEGIN { print "Hello from AWK" }
```

Here, the kernel launches `awk -f` and passes the script path as that file argument. Make it executable (`chmod +x`) and it runs directly like any other script.

{{< book-hook >}}

## The Disappearing Script

Here's where it gets genuinely fun.

The kernel doesn't care what the interpreter is. It reads the path, calls `execve()`, and hands off the script. So what happens
if you point the shebang at something... unexpected?

```bash
#!/bin/rm -v
# This script has a very short life expectancy.
echo "You will never see this."
```

Make it executable and run it:

```bash
chmod +x script.sh
./script.sh
```

The kernel reads the shebang, sees `/bin/rm -v`, and calls `execve()` with `rm`, the `-v` flag, and the script path. In other
words, the very first thing that happens is:

```bash
/bin/rm -v ./script.sh
```

The script deletes itself. Before any shell is involved. The `echo` line is never reached — `rm` is not a shell and has no
interest in the rest of the file. The process exits, and the file is gone.

This is not a pattern you'd use in production, but it's a perfect illustration of the mechanism: the shebang interpreter
receives the script path as an argument and can do absolutely anything with it. The kernel has no opinion on the matter.

## A Note on `source`

One quirk worth knowing: when you `source` a script (with `source myscript.sh` or `. myscript.sh`), the kernel is never involved. The script runs in the current shell process — there is no exec, no interpreter lookup, no shebang processing. The shebang line becomes an ordinary comment, ignored completely.

This is why sourced library files often omit the shebang or use a note in its place. It doesn't cause harm to leave it there, but it serves no purpose when the file is sourced rather than executed.

## What the Kernel Gives Us

The shebang is one of those Unix design decisions that's elegant in hindsight. The kernel doesn't need to understand scripting
languages or interpreters — it just needs a way to dispatch execution to the right one. Two bytes and a path are all it takes.

Understanding this gives you a cleaner mental model of what happens when you run a script: the kernel finds the interpreter,
calls `execve()`, and steps aside. The interpreter — whatever it is — takes it from there. Even if it's `rm`.
