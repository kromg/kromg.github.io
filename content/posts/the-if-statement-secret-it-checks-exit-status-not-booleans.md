---
title: "The if Statement Secret: It Checks Exit Status, Not Booleans"
date: 2026-03-21
description: "Bash doesn't have true and false the way other languages do. Understanding that if checks exit status—not boolean values—changes how you write conditionals forever."
tags: ["bash", "conditionals", "fundamentals", "scripting"]
categories: ["Shell Scripting", "Counter-Intuitive"]
featured_image: "/images/if-statement-cover.png"
---

Coming from almost any other programming language, you'd expect `if` to work on boolean expressions. You write something that evaluates to true or false, and the `if` decides which branch to take.

Bash works in a way that is almost, but not quite, entirely unlike that.

In Bash, `if` runs a command and checks its *exit status*. Zero means success (the "then" branch runs). Non-zero means failure (the "else" branch runs, if present).

This is subtle but fundamental. Once you understand it, a lot of Bash's behavior clicks into place.


## Commands, Not Expressions

Watch what happens here:

```bash
if true; then echo "yes"; fi
# Output: yes

if false; then echo "yes"; else echo "no"; fi
# Output: no
```

You might think `true` and `false` are boolean literals. They're not. They're *commands*—programs that do nothing except exit with a specific status:

```bash
true
echo $?   # 0 (success)

false
echo $?   # 1 (failure)
```

The `if` statement runs the command after it, checks `$?` (the exit status), and branches accordingly. That's all it does.


## Any Command Works

Since `if` just checks exit status, you can use any command:

```bash
if grep -q "error" logfile.txt; then
  echo "Errors found"
fi
```

Here, `grep -q` returns 0 if the pattern matches, non-zero otherwise. The `if` doesn't care what `grep` does—it only cares about the exit status.

This is powerful. You can test whether commands succeed directly:

```bash
if ping -c 1 google.com &>/dev/null; then
  echo "Network is up"
else
  echo "Network is down"
fi
```

```bash
if make -j4; then
  echo "Build succeeded"
else
  echo "Build failed"
  exit 1
fi
```

No need for temporary variables or complex checking—the command's exit status *is* the condition.


{{< book-hook >}}


## What About `[ ]` and `[[ ]]`?

Here's where it gets interesting. What are `[` and `[[`?

They're commands (well, `[` is a command; `[[` is a keyword, but it works similarly). They evaluate conditions and return exit status. The differences between them — quoting rules, pattern matching, regex support — are worth a dedicated post, and one is coming.

```bash
[ 5 -gt 3 ]
echo $?   # 0 (true/success)

[ 2 -gt 3 ]
echo $?   # 1 (false/failure)
```

Because they're just commands that return exit status, you can use them *standalone*—no `if` required. This is incredibly useful on the command line when testing assumptions:

```bash
[ -f "$config" ] && echo "Config exists"
[[ "$name" == z* ]] || echo "Name doesn't start with z"
```


So when you write:

```bash
if [ "$name" = "zaphod" ]; then
  echo "Hello, President"
fi
```

You're running the `[` command with arguments `"$name"`, `=`, `"zaphod"`, `]`. That command evaluates the string comparison and exits with 0 or 1. The `if` then checks that exit status.

Because `[` is a genuine command, the shell needs a space between the command name and its first argument — just as you'd write `grep -q pattern` rather than `grep-qpattern`. Writing `["$name" = "zaphod"]` looks plausible to anyone coming from a language where brackets are just syntax, but Bash sees `["$name"` as a single, unknown token and returns a terse `command not found` — a cryptic error that leaves many beginners bewildered. The same rule applies to the closing `]`: it is an argument to `[`, not a matching bracket, so a space before it is also mandatory.

```bash
# Wrong: no space after [ or before ]
if ["$name" = "zaphod"]; then echo "hello"; fi
# bash: ["zaphod": command not found

# Correct
if [ "$name" = "zaphod" ]; then echo "hello"; fi
```

`[[` works the same way—it evaluates its expression and exits 0 (condition true) or 1 (condition false).

This is why `if test -f "$file"` and `if [ -f "$file" ]` are equivalent: `test` and `[` are the same command (literally—check `man test`).


## The Inversion Trap

In most languages, 0 means false and non-zero means true. In shell exit codes, it's opposite: 0 is success/true, non-zero is failure/false.

This trips people up:

```bash
# WRONG mental model
if [ 1 ]; then echo "truthy"; fi   # This prints "truthy"
if [ 0 ]; then echo "truthy"; fi   # This ALSO prints "truthy"
```

Wait, what? Both print "truthy"?

Yes. Because `[ 1 ]` tests whether the string "1" is non-empty (it is). And `[ 0 ]` tests whether the string "0" is non-empty (it is). Both are true.

If you want to check a numeric value:

```bash
value=0
if [ "$value" -eq 0 ]; then echo "zero"; fi

# Or with arithmetic evaluation:
if (( value == 0 )); then echo "zero"; fi
```

The `[ ]` and `[[ ]]` constructs test *strings* and *files* and *numeric comparisons*—they're not evaluating the "truthiness" of a value the way Python or JavaScript would.


## Exit Status Arithmetic

You can chain commands based on exit status with `&&` and `||`:

```bash
# && runs the right side only if the left side succeeds (exits 0)
mkdir build && cd build && cmake ..

# || runs the right side only if the left side fails (exits non-zero)
cd /nonexistent || echo "Directory doesn't exist"
```

This is how the error-first pattern works:

```bash
[[ -f "$config" ]] || { echo "Config missing" >&2; exit 1; }
```

Read it as: "Test that config exists. If that fails, print error and exit."

**Caveat: Mixing `&&` and `||` in one line**

It's tempting to write composite logic like this:

```bash
command_a && command_b || fallback_command
```

But beware: `fallback_command` runs if *either* `command_a` fails *or* `command_b` fails. This is usually not what you want. The logic you probably meant is:

```bash
if command_a; then
  command_b
else
  fallback_command
fi
```

"But can't I just group them?" you might think. Nope—`{ command_a && command_b; } || fallback_command` still runs the fallback if `command_b` fails. You'd need something like `command_a && { command_b; true; } || fallback_command`, but now you're masking `command_b`'s failure, which defeats the point.

Just use `if`/`then`/`else`. It's clearer, it does what you expect, and you won't spend an hour debugging why your fallback runs when it shouldn't.


## Functions Return Exit Status Too

Functions in Bash return exit status, not values:

```bash
is_even() {
  local num=$1
  (( num % 2 == 0 ))
}

if is_even 4; then
  echo "4 is even"
fi

if is_even 7; then
  echo "7 is even"
else
  echo "7 is odd"
fi
```

The function doesn't return "true" or "false"—it returns 0 (success) or 1 (failure) based on whether the arithmetic expression is non-zero.

This makes functions natural to use in conditionals. Design them to return appropriate exit status, and `if` just works.


{{< book-hook >}}


## The `!` Operator

You can negate exit status with `!`:

```bash
if ! grep -q "error" logfile.txt; then
  echo "No errors found"
fi
```

The `!` flips the exit status: 0 becomes 1, non-zero becomes 0.


## Common Gotcha: Command Substitution

What about this?

```bash
result=$(some_command)
if [ "$result" ]; then
  echo "Got output"
fi
```

This tests whether `$result` is a non-empty string—which is probably what you want. But what if you want to check whether `some_command` *succeeded*, not whether it produced output?

A command can succeed but produce no output. A command can fail but produce output. They're independent.

To check success:

```bash
if result=$(some_command); then
  echo "Command succeeded, output: $result"
fi
```

To check both:

```bash
if result=$(some_command) && [ -n "$result" ]; then
  echo "Command succeeded AND produced output"
fi
```


## Thinking in Exit Status

Once you internalize that `if` checks exit status, you start writing more idiomatic Bash:

**Instead of:**
```bash
output=$(ls /some/dir 2>/dev/null)
if [ -n "$output" ]; then
  echo "Directory has files"
fi
```

**Consider:**
```bash
if ls /some/dir &>/dev/null; then
  echo "Directory is accessible"
fi
```

**Instead of:**
```bash
grep -q "pattern" file.txt
if [ $? -eq 0 ]; then
  echo "Found"
fi
```

**Just:**
```bash
if grep -q "pattern" file.txt; then
  echo "Found"
fi
```

The command *is* the condition. Let `if` do its job.


## Writing Conditions Worth Reading

Everything covered so far — `if` running commands, `[ ]` being a command, functions returning exit status — converges on one practical insight that shapes how you structure real code.

Because `if` calls any command, you can give your conditions *names*. Instead of scattering comparisons and file tests through your scripts, you encapsulate them once, in a function with a clear name, and call that function wherever you need it. What used to be a tangle of flags and operators becomes a sentence:

```bash
is_valid_port()    { (( $1 >= 1 && $1 <= 65535 )); }
is_readable_file() { [[ -f "$1" && -r "$1" ]]; }
is_root()          { (( EUID == 0 )); }
```

Guards — preconditions that must hold or the script stops — read naturally with `||`:

```bash
is_root                    || die "This script must be run as root"
is_readable_file "$config" || die "Cannot read config: $config"
is_valid_port "$port"      || die "Invalid port: $port (must be 1–65535)"
```

When there are two meaningful paths, `if`/`else` earns its place:

```bash
if is_root; then
  install_system_wide
else
  install_user_local
fi
```

In both cases, nobody at the call site needs to know whether the check involves arithmetic, a file test, or a regex — only what it *means*. `is_valid_port "$port"` is not a Bash expression. It is a sentence.

Named conditions also do something that comments cannot: they stay true. A comment saying `# check if port is in valid range` can drift away from the code it describes — the check changes, the comment doesn't, and now the comment lies. Robert C. Martin made this point forcefully in *Clean Code*: comments are a failure to express intent in the code itself. They become old, they become wrong, and the code keeps running as if nothing happened. The better approach is to encode the concept into a name, so the code and its meaning are inseparable. `is_valid_port "$port"` doesn't need a comment explaining what it does. It already says so.

This is the habit that separates scripts that work from scripts that can be read, reviewed, and trusted. The exit-status model makes it possible; good naming makes it pay off.

A library of small, focused predicate helpers is one of the most useful things you can build for your own scripting environment — and it's exactly what the Bash Production Toolkit provides. Subscribe below to receive it.


## The Bottom Line

Bash's `if` statement is simpler than it looks: it runs a command and checks the exit status. Zero means success. Non-zero means failure. That's it.

Everything else — `[ ]`, `[[ ]]`, `(( ))`, `test`, `true`, `false`, and your own functions — are just commands that produce exit codes. Once that model replaces the "boolean expression" intuition you brought from other languages, Bash stops feeling cryptic and starts feeling consistent.

