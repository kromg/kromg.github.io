---
title: "Have You Got the SKILLs? Teaching AI to Write Better Bash"
date: 2026-02-19
description: "Use GitHub Copilot Agent Skills to generate consistent, clean Bash scripts. A practical guide to instructing AI agents with professional coding standards."
tags: ["bash", "copilot", "ai", "best-practices", "automation"]
categories: ["Shell Scripting", "AI Tools"]
---

 — *"Elmo's got the moves"* playing in my head since I typed this article's title —

I wrote a book on Bash scripting that I'm about to publish. It teaches you how to write Bash, and more importantly, how to write it *right*. 

But cool kids don't write code anymore, do they?

![Back to the Future](/images/bttf1.gif)

Maybe the time will never come again when we'll need to read and understand code ourselves—AI agents will have us covered. Maybe it's fine. Maybe you don't need to know the basics.

But then Isaac Asimov's "[The Feeling of Power](https://en.wikipedia.org/wiki/The_Feeling_of_Power)" keeps echoing in my head (*kinda strange with "Elmo's got the moves" playing in the background. Oh, well*).<br/>
You know the story, right? Humanity offloads all the work to the machines and forgets how to do even basic arithmetic. Too unreal, right? *Right??*

If you don't want to start by letting AI write mystery code and end up entangled in a future mess, you need to understand what's going on behind the scenes. But here's the good news—you can have both. You can use AI to solve your problems *and* make the solutions clear enough to learn from.


## The Secret: Teach Your AI to Write Code for Humans

The strategy is simple: instruct your AI agents to write code the *right* way. Not code optimized for machines 
(machines will understand it anyway), but code written for the next human who needs to read and maintain it. 
That human could be you: today, tomorrow, six months from now.

What does "the right way" mean? It means talking to your agent about:

- **Single responsibility** — one function, one job
- **Clean Code** principles — self-explaining names, small functions, minimal comments that explain *why*, not *what*
- **Modularity** — code organized into reusable, testable pieces
- **Composition over monoliths** — small, composable scripts rather than sprawling mega-files
- **Testability** — code structured so you can verify it works

You'd be surprised how dramatically code quality improves when you simply tell the AI: "Write code that speaks for itself. Prefer descriptive names over comments."


{{< signup-cta >}}


## Enter Copilot Agent Skills

I use Copilot extensively—it's become my primary coding companion. My old workflow was to start each project by asking Copilot to interview me about my guidelines, then summarize everything into a `.github/copilot-instructions.md` file. That file would capture my persona, coding conventions, project details, and implementation plans. It saved me countless hours of repetitive prompting.

But there was a problem: I was repeating myself across projects. Every time I needed Bash scripts, I'd specify the same conventions. Every Python file, the same style guidelines. Java projects? There we went again.

[Agent Skills](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills) solve this elegantly. Skills are reusable instruction sets that Copilot loads when relevant. Need Bash scripts? Your Bash skill activates. Adding some Python? Your Python skill kicks in. No more copy-pasting the same guidelines between projects.

Skills live in `~/.copilot/skills/` (personal, shared across all your projects) or `.github/skills/` (project-specific). Each skill is a folder containing a `SKILL.md` file with YAML frontmatter and Markdown instructions.

Here's what I include in my Bash scripting skill:

````markdown
---
name: bash-scripting
description: >
  Guidelines for writing clean, maintainable Bash scripts.
  Use this skill when generating or modifying .sh files or
  any shell script with a bash shebang.
---

## Script Structure

Every script starts with:

```bash
#!/usr/bin/env bash
# script_name.sh — Brief description of what this script does
# Copyright (c) YEAR Author Name. License: MIT (or your choice)

set -uo pipefail
```

- Group related functions together; place the main logic at the bottom.
- Use `main()` function pattern for non-trivial scripts; call it at the end.

## Logging and reporting

Define helper functions for logging and error reporting:

```bash
_log() {
  local level="$1"; shift
  printf '[%s] | %-5s | %s\n' "$(date +%Y-%m-%d\ %H:%M:%S)" "$level" "$*" >&2
}

log()  { _log INFO "$@"; }
warn() { _log WARN "$@"; }

die() {
  local code=1
  if [[ "$1" =~ ^[0-9]+$ ]]; then code="$1"; shift; fi
  _log ERROR "$@"
  exit "$code"
}
```

Use `die 2` for usage errors; `die` (or `die 1`) for runtime errors.

## Error Handling: Error-First Pattern

Use explicit error checks rather than `set -e`. The error-first pattern
handles failures immediately and keeps the happy path flat:

```bash
[[ -f "$config_file" ]] || die "Config not found: $config_file"
[[ -r "$config_file" ]] || die "Config not readable: $config_file"
```

**Why not `set -e`?** It has surprising edge cases that can cause
scripts to continue when you expect them to fail, or fail when you
expect them to continue. Explicit checks are predictable. We do use
`set -u` (catch undefined variables) and `set -o pipefail` (catch
pipeline failures)—these are safe and predictable.

## Variables and Quoting

- **Always quote variable expansions**: `"$var"`, `"${array[@]}"`.
- **Use meaningful names**: `input_file`, `line_count`, `user_name`—not `f`, `n`, `u`.
  Exception: `i`, `j` for simple loop indices.
- **Prefer `${var:-default}`** for optional variables with defaults.
- **Use `local`** for all variables inside functions.

## Conditionals

- **Prefer `[[ ... ]]`** over `[ ... ]` for conditionals (safer, more features).
- **No need to quote inside `[[ ]]`** for simple variable tests, but quote elsewhere.
- **Write predicate functions** for repeated conditions:

```bash
is_readable_file() { [[ -f "$1" && -r "$1" ]]; }

is_readable_file "$input" || die "Cannot read: $input"
```

## Functions

- Define with `name() {` syntax (portable).
- Use `local` for all internal variables.
- Return status with `return N` (0 = success).
- **Single responsibility**: one function, one job.
- **Predicate functions** return only status, print nothing.

For returning data, prefer command substitution for readability:

```bash
get_timestamp() { date +%Y%m%d-%H%M%S; }
backup_name="backup-$(get_timestamp).tar.gz"
```

For performance-critical code (tight loops, large data), consider
named references (`local -n`) to avoid subshell overhead.

## Output and Formatting

- **Use `printf`** instead of `echo` for reliable output.
- **Send errors/logs to stderr**: keep stdout clean for data.
- **Indent with 2 spaces**; no tabs.
- **Closing keywords** (`fi`, `done`, `esac`) on their own line.

## Command Substitution

- **Use `$(...)`** instead of backticks.
- **Check exit status** when command substitution can fail:

```bash
output="$(some_command)" || die "Command failed"
```

## Iteration and Globs

- **Quote `"$@"`** when forwarding arguments.
- **Quote loop variables**: `for file in *.txt; do process "$file"; done`
- Handle empty globs safely (either check first or use `shopt -s nullglob`).

## Avoid

- **Parsing `ls` output** — use globs or `find` instead.
- **Unquoted expansions** — leads to word splitting bugs.
- **`eval`** — almost never necessary; use safer alternatives.
- **Single-letter variable names** — except `i`, `j` for indices.
- **Comments explaining *what*** — write self-explanatory code instead.
````


{{< signup-cta >}}



## Making It Your Own

The skill I've shared reflects patterns I've found valuable over years of shell scripting. Your preferences might differ—and that's fine. The point is to *have* explicit guidelines rather than accepting whatever the AI generates by default.

Some variations you might consider:

- **Stricter or looser quoting rules** depending on your environment
- **Different indentation** (4 spaces instead of 2)
- **Project-specific conventions** like naming prefixes for functions in libraries
- **Error handling philosophy:** The book covers both error-first (as shown here) and strict modes (`set -e`), explaining the tradeoffs of each approach in depth

Start with these guidelines, observe the code Copilot generates, and refine. The skill becomes a living document encoding your team's collective wisdom about what makes Bash code maintainable.

**Where these patterns come from:** The SKILL.md above distills principles from *Bash: The Developer's Approach*—error-first pattern, predicate functions, meaningful names, the `set -uo pipefail` decision. The book goes deeper into the *why* behind each choice, shows the failure modes they prevent, and teaches you to recognize when to apply them. Think of the skill as your AI's quick reference guide; the book is your comprehensive understanding.


## The Bigger Picture

Teaching AI to write better code isn't just about getting cleaner output—it's about staying in control. When you can read and understand the code your tools generate, you can:

- Catch bugs before they hit production
- Modify and extend the code confidently
- Learn patterns you can apply elsewhere
- Maintain systems long after the AI-assisted development session ends

The goal isn't to avoid learning Bash. It's to learn *while* getting work done, with an AI that teaches by example rather than writing mysterious incantations.

Agent Skills are one piece of this puzzle. They turn ad-hoc prompting into systematic instruction, ensuring consistency across projects and sessions. Your future self—debugging a script at 2 AM—will thank you.
