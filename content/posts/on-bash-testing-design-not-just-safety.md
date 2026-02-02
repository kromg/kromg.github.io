---
title: "On Bash Testing: Design, Not Just Safety"
date: 2026-02-02
description: "Testing Bash scripts isn't only about catching bugs—it's about writing better code. Discover how the act of testing forces you to design scripts that are modular, predictable, and maintainable."
tags: ["bash", "testing", "tdd", "design", "best-practices"]
categories: ["Shell Scripting", "Software Engineering"]
---

Testing as a topic is a minefield already when it comes to full-fledged programming languages, let's not even talk about it for shell scripting.

Or shall we?

You see, whether you write software to test other software, or you just fire up a script from the command line and keep running it over and over again until you nail it, that's still testing, right?

Have you ever been woken up at night because a script failed six months after you wrote it?
Or one year later? Or *five* years later? — *For the love of my life, that thing was supposed to be temporary!*
There's nothing more permanent than a temporary solution, they say.

The problem with testing is that there's only so much you can reproduce on the command line.

What if the script you're going to run will run in a different environment (remember *crontab*?),
under a different user, or at a different time of the day, or of the year (daylight saving time, anyone?).
There are an infinite number of things that can change, and possibly twice as many that can go **wrong**.

And then, how much time will it take to run the script through the correct environment to test it?
Is it idempotent, at least, or do you need to re-setup the running conditions every time you stop mid-test for an error?
Do you need to run it through a pipeline? How brittle is the test method?
What if you forgot to set up some pre-condition, and got the script apparently working, only for it to fail in production?

*Well*, you could think, *maybe I can create a script to set up the proper environment and emulate the conditions where my script will run...*

Now we're talking about testing!

Welcome to the testing jungle, where nothing is defined, and no one agrees on what method to use!


{{< signup-cta >}}


## The Testing Conundrum

Testing is: "running your own QA".
It's not only about making sure the car you're building won't explode the first time it's turned on.
It's more like making it safe even when you're hitting the brakes on a dark night, while driving on ice.

There are different depths and strategies for testing, especially to test Bash scripts, and they depend on multiple factors. Let's call them the **testing decision factors**:

- How **complex** is the software you're writing?
- Is it **idempotent**? (Can you run it twice without harm?)
- Does it depend on a specific **environment**, or is it meant to run as a subprocess of another system?
- Does it depend on **external resources**? (Network, database, cloud APIs, etc.)
- Does it depend on **dates and times**?
- How high are the **stakes** of the task it's carrying out? (Will people lose money if it fails? Can it fail X times without repercussion? Can it fail for a given amount of time without consequences?)
- How **critical** is the data it's manipulating? (Will it lose or garble user data? Logs? Maybe something you're legally required not to lose?)
- How likely is it that the code will be **changed in the future**?
- How long will this script **live**? (One-off or maintained for years?)

Every one of these aspects calls for its own reasons to test, or not to test. External dependencies are hard to mock, for example (calling for a manual test), but at the same time they may make it harder to replicate the exact state needed to properly test your script—which would make tests with mocks more desirable.

People don't usually agree upon these aspects: should we mock, or run integration tests with "the real thing"?
How much effort is justifiable in writing tests, when we can just run the software and see if it works?

My opinion here is clear: *it depends*.


## There's More to It Than Meets the Eye

Testing is not *only* "running your own QA".

Every TDD enthusiast will tell you that test-driven development helps you **write better code**.
On the other side, every TDD detractor will tell you that writing tests before writing the code feels unnatural, and that they need to try and prototype the code before they can write tests.

But *stopping and thinking* is the whole point of test-driven development.

So, let's add this question to our little conundrum:
- Can the software you're writing be made simpler to test, by splitting it into simpler chunks?


## When Code Is Hard to Test, It's Telling You Something

Here's the secret that TDD advocates don't always spell out: **code that's hard to test is usually code that's hard to maintain**.

Think about it. If you can't isolate a function to test it, that function is doing too many things. If testing requires elaborate mocks and stubs, the code is too entangled with its environment. If every small change breaks half your tests, the code has too many implicit dependencies.

The act of testing—or even just *thinking* about how you'd test something—shines a light on these problems.

Consider a function that does everything: reads a config file, validates the content, connects to a database, and writes a log entry. To test this, you'd need a real config file, a running database, and somewhere for logs to go. That's not a unit test—that's a deployment rehearsal.

Now imagine you break it apart:

```bash
# Pure logic - testable in isolation
validate_config_value() {
  local key=$1
  local value=$2
  [[ -n "$value" ]] || return 1
  [[ "$key" =~ ^[A-Z_][A-Z0-9_]*$ ]] || return 1
  return 0
}

# I/O operation - separate concern
read_config_file() {
  local file=$1
  [[ -f "$file" ]] || return 1
  cat "$file"
}

# Orchestration - ties it together
load_config() {
  local config_file=$1
  local content
  content=$(read_config_file "$config_file") || return 1
  # Parse and validate each line...
}
```

Suddenly, `validate_config_value` can be tested with three lines of code, no files involved:

```bash
validate_config_value "DATABASE_URL" "postgres://localhost" && echo "PASS"
validate_config_value "" "some_value" || echo "PASS: empty key rejected"
validate_config_value "good_KEY" "" || echo "PASS: empty value rejected"
```

The testing constraint forced a better design. The function does one thing. It's easy to understand. It's easy to reuse. It's easy to test.


## The Testing Pressure Test

Here's a practical exercise. Take any function you've written and ask yourself: "How would I test this?"

If your answer involves:
- Creating temporary files and directories
- Spinning up services or containers
- Mocking half the commands in `/usr/bin`
- Running in a specific environment with specific variables set

...then you've found a design problem, not a testing problem.

The solution isn't to write more elaborate tests. The solution is to **refactor until testing becomes trivial**.

Extract the pure logic. Separate the I/O. Reduce the surface area that touches the outside world. What remains should be small orchestration code that's either obviously correct or easy to verify with a quick manual run.


{{< signup-cta >}}


## Pure Functions: The Testing Sweet Spot

In Bash, "pure functions" aren't quite the same as in functional programming, but the principle holds. A function that:
- Takes input only through its arguments
- Produces output only through `stdout` (or by setting a nameref variable)
- Has no side effects (no file writes, no global variable mutations)

...is trivially testable.

```bash
# Pure: same inputs, same outputs, no side effects
to_uppercase() {
  local -n result_ref=$1
  local input=$2
  result_ref="${input^^}"
}

# Test it
to_uppercase output "hello"
[[ "$output" == "HELLO" ]] && echo "PASS" || echo "FAIL"
```

Compare that to:

```bash
# Impure: touches filesystem, reads environment, logs output
process_user_data() {
  local user=$1
  local data_file="/home/$user/.app_data"
  
  [[ -f "$data_file" ]] || { echo "ERROR: no data" >&2; return 1; }
  
  local processed
  processed=$(some_transformation < "$data_file")
  
  echo "$processed" >> "/var/log/app_$(date +%Y%m%d).log"
}
```

The second function is a testing nightmare. You need a real user directory, a specific file, writable log directories, and you're at the mercy of the current date. Every test run leaves traces. Cleanup is tedious. Isolation is impossible.

The irony? The *interesting* part—the transformation logic—is buried inside, impossible to test in isolation.


## Design for Testability, Reap the Benefits

When you design for testability, you get more than just tests. You get:

**Modularity.** Functions that do one thing are reusable. That validation logic you extracted? Now it can be used in three different scripts without copy-pasting.

**Readability.** Small, focused functions are easier to understand. The function name tells you what it does. The body shows you how.

**Debuggability.** When something breaks, you can test components in isolation. Instead of staring at a 200-line script wondering where it went wrong, you can verify each piece independently.

**Maintainability.** Changes are localized. Modify the validation rules? Only one function changes. The rest of the script doesn't need to know.


## When Not to Bother

Like we said at the beginning, not every script needs this treatment. Let's review our checklist: 

- **Complexity:** A five-line script doesn't need a test harness. A 500-line orchestration script probably does.
- **Idempotency:** If running twice is harmless, manual testing is easier. If running twice causes disaster, you want guarantees.
- **Environment dependencies:** The more your script depends on specific environments, users, times, or external resources, the more valuable tests become—but also the harder they are to write.
- **Stakes:** Will it lose money? Corrupt data? Violate legal requirements? The blast radius determines the investment.
- **Lifespan:** Throwaway scripts don't need tests. Long-lived automation does.

This is why I said "it depends." There's no universal rule. The economics of testing change with each factor.

If you're writing a one-liner to rename some files, *just run it*. If it works, you're done. The feedback loop is instant—you'll see immediately if something went wrong.

If you're writing a quick data extraction script you'll run once and throw away, the time spent making it "testable" is time wasted. Write it, run it, verify the output, move on.

But here's the twist: even when you don't *need* tests, thinking about testability still pays off:

- can you **split** your workflow? Retrying one phase will take less time than retrying the whole thing.
- can you extract something you'll be using in **other** scripts? Write it once, use it many times (if it's tested, you won't have to come back to fix it later)
- can you reduce complexity by solving **one problem at a time**? That'll make each chunk easier to test (even if your test is a manual run)
- can you write each little chunk as a **small function**? That's how you achieve all three above, _and_ your code becomes easier to read! 


## The Habit: Spotting Units of Reusability

Here's what separates developers who struggle with testing from those who find it natural: the ability to spot **units of reusability** while writing code.

Every time you write a script, ask yourself: *how much of this will I use again?*

Not the whole script—the pieces. The patterns. The little helpers that solve problems you've solved before and will solve again.

Consider what you might extract:

- **Logic helpers:** Predicates like `is_integer()`, `is_valid_email()`, `file_exists_and_readable()`. These are pure functions, trivially testable, and you'll use them everywhere. (I've built a small library of these that I give away for free—grab the [Bash Production Toolkit](https://lost-in-it.kit.com/bash-production-toolkit) if you want a head start.)

- **Logging:** How many times have you written `echo "ERROR: ..." >&2`? Extract it once, test it once, reuse it forever. (This is exactly what *log4b* does—a logging library I'm building alongside my book.)

- **Environment-specific patterns:** Connections to specific hosts, database access functions, cloud CLI wrappers. If you work in a particular environment, these patterns repeat constantly.

- **Data handling:** Configuration parsing, CSV/JSON transformations, path manipulation, filename normalization. The format may change, but the pattern is stable.

- **Validation:** Input sanitization, argument checking, precondition verification. These are the gatekeepers that prevent garbage from propagating through your scripts.

When you start seeing scripts not as monoliths but as assemblies of reusable parts, testing becomes obvious. You test the parts. The assembly is just wiring—often too simple to need tests, or easily verified with a quick manual run.


## Self-Testing Code: The Pragmatic Middle Ground

There's a path between "no tests" and "full test suite": make your code self-verifying.

- Add `--dry-run` flags that show what *would* happen without doing it.
- Add verbose modes that log every decision.
- Make operations idempotent so running twice is safe.
- Use meaningful exit codes that tell you exactly what went wrong.
- Structure scripts so you can source individual functions and poke at them interactively.

This isn't testing in the formal sense, but it achieves similar goals: confidence that the code behaves as expected, and quick feedback when something goes wrong.

The key insight is that *testable code is debuggable code*. Even if you never write a single test file, designing for testability makes your scripts easier to understand, modify, and trust.


## The Takeaway

The real value of testing isn't catching bugs—though that's a nice side effect. The real value is that *the discipline of testing makes you write better code*.

If you can't figure out how to test a function, that function probably does too much.
If testing requires elaborate setup, your code is too coupled to its environment.
If tests are brittle and break with every change, your design needs work.

But perhaps more importantly: if you keep the big picture in mind—always asking "what here is reusable?"—writing tested, modular code becomes second nature. You stop seeing testing as an extra chore and start seeing it as a natural consequence of good design.

Testing is a mirror. It reflects the quality of your design back at you, warts and all. You can look in that mirror and fix what you see, or you can avoid mirrors entirely. But the problems will still be there, whether you see them or not.

The choice is yours.

---
