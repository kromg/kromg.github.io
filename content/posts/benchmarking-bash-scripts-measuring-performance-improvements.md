---
title: "Benchmarking Bash Scripts: Measuring Real Performance Improvements"
date: 2026-01-01
description: "Learn how to accurately measure Bash script performance and discover why name references can be 10-20x faster than command substitution. Includes practical benchmarking techniques and a real-world case study."
tags: ["bash", "performance", "benchmarking", "optimization", "shell-scripting"]
---

Happy New Year! Let's start 2026 with a look at something that surprises many developers: Bash script performance actually matters, and you can measure and improve it systematically.

If you've ever written Bash scripts that process thousands of log entries, iterate over file lists, or run in tight loops, you've probably wondered: "Is this as fast as it can be?" The answer is usually no—but the trick is knowing what to measure and how to interpret the results.

This post shows you how to benchmark Bash code properly, reveals a common performance pitfall that can cost you 10-20x in speed, and demonstrates the techniques with a real logging library that went from sluggish to snappy with a single architectural change.

## Why Benchmark Bash?

Bash isn't Python or C. It's designed for gluing commands together, not CPU-intensive computation. But that doesn't mean performance is irrelevant. When your script runs once a day, a few extra milliseconds don't matter. When it runs in a loop processing 10,000 items, or when it's a library function called thousands of times, those milliseconds add up fast.

Consider a logging library. Every time you write `lb_info "Processing file $filename"`, the library formats the message, checks the log level, and writes output. If this happens once per file in a deployment script processing 5,000 files, the library's performance directly impacts the script's runtime. A slow library turns a 30-second deploy into a 5-minute wait.

The good news: Bash performance problems are usually architectural, not algorithmic. A small change in how you structure your code can yield dramatic improvements. But you need measurements to know what to fix.

## The Benchmarking Basics

Before we dive into specific techniques, let's establish the fundamentals of Bash benchmarking.

### Measuring Time: The Right Way

Bash provides the `time` builtin, which measures how long a command takes:

```bash
time for ((i=0; i<1000; i++)); do
  echo "Processing item $i" > /dev/null
done
```

Output:
```
real    0m0.125s
user    0m0.089s
sys     0m0.036s
```

Three numbers appear:

- **real**: Wall-clock time (what you experience as a human)
- **user**: CPU time spent in user space (your script's code)
- **sys**: CPU time spent in kernel space (system calls)

For most benchmarks, focus on **real** time—it reflects the actual duration. If you're comparing two implementations, the one with lower real time is faster in practice.

### Controlling Variables

A good benchmark isolates the thing you're testing. To compare two implementations fairly:

1. **Run both tests in the same environment**: Same shell, same system load, same input data.
2. **Run multiple iterations**: A single run might be affected by system noise (caching, background processes). Repeat the test several times and look at the average.
3. **Disable output**: Redirect output to `/dev/null` so you're measuring computation, not I/O.
4. **Use consistent iteration counts**: Compare apples to apples—same number of operations in both tests.

Here's a simple benchmark template:

```bash
#!/bin/bash

iterations=10000

echo "=== Benchmark: Implementation A ==="
time for ((i=0; i<iterations; i++)); do
  # Implementation A
  some_function
done > /dev/null

echo ""
echo "=== Benchmark: Implementation B ==="
time for ((i=0; i<iterations; i++)); do
  # Implementation B
  another_function
done > /dev/null
```

This structure gives you comparable numbers. Run it a few times to check consistency—if results vary wildly (more than 10-20%), your environment is too noisy or your iteration count is too low.

### Extracting Numeric Results

For automated testing or regression tracking, you want numeric data, not formatted output. Bash provides the `SECONDS` variable, which automatically tracks elapsed time:

```bash
SECONDS=0
# ... do work ...
duration=$SECONDS
echo "Duration: $duration seconds"
```

The `SECONDS` variable is a shell builtin—it increments automatically, requires no external processes, and provides sufficient precision for most benchmarking tasks. For sub-second precision, you can use `SECONDS` as a decimal: `printf '%.3f' "$SECONDS"` gives you milliseconds. This gives you a number you can compare programmatically or log to a file for trend analysis.


{{< signup-cta >}}


### Building a Benchmarking Library

Since benchmarking is a recurring task, it makes sense to extract the pattern into a reusable library. Here's a minimal `benchmark.sh` that encapsulates the measurement logic:

```bash
#!/usr/bin/env bash
# benchmark.sh - Simple Bash benchmarking utilities

# Run a command multiple times and measure duration
benchmark() {
  local description="$1"
  local iterations="$2"
  shift 2
  
  printf '%-50s ' "$description"
  
  SECONDS=0
  for ((i=0; i<iterations; i++)); do
    "$@" > /dev/null 2>&1
  done
  
  local elapsed=$SECONDS
  local throughput=$(( iterations / (elapsed + 1) ))
  
  printf '%6d ms  (%s ops/sec)\n' "$((elapsed * 1000))" "$throughput"
}
```

With this helper, benchmarking becomes one line:

```bash
source lib/benchmark.sh

benchmark "Level resolution test" 10000 level_to_num_nameref level "INFO"
```

This pattern eliminates repetition, makes benchmarks easier to read, and encourages consistent measurement across your codebase.

## The Subshell Problem: A Hidden Tax

Now let's get specific. One of the biggest performance traps in Bash is unnecessary subshells, particularly via command substitution: `$(...)`.

### What's a Subshell?

When you write `result=$(compute_hash "$file")`, Bash does this:

1. Forks a new subshell (a copy of the current process)
2. Runs `compute_hash "$file"` inside that subshell
3. Captures the output
4. Returns it to the parent shell as a string

Each fork is a system call. Each system call has overhead. In a tight loop, this overhead dominates. It's like paying a processing fee every time you move data between variables—except the fee is paid in microseconds instead of dollars, and it compounds just as aggressively.

### The Smoking Gun: A Comparison

Let's compare two implementations of a simple helper that converts a log level name to a number:

**Implementation A: Command Substitution**

```bash
level_to_num() {
  case "$1" in
    DEBUG) echo 0 ;;
    INFO)  echo 1 ;;
    WARN)  echo 2 ;;
    ERROR) echo 3 ;;
    *)     echo 1 ;;
  esac
}

# Usage
level=$(level_to_num "INFO")
```

Every call spawns a subshell to capture `echo` output.

**Implementation B: Name References**

```bash
level_to_num() {
  local -n result=$1  # Create a name reference
  local level_name=$2
  case "$level_name" in
    DEBUG) result=0 ;;
    INFO)  result=1 ;;
    WARN)  result=2 ;;
    ERROR) result=3 ;;
    *)     result=1 ;;
  esac
}

# Usage
level_to_num level "INFO"
# $level now contains 1
```

The `local -n result=$1` creates a **name reference**: a variable that acts as an alias for another variable. When the function assigns `result=1`, it's actually assigning to the variable named by the first argument. No subshell required.

### Benchmarking the Difference

Let's measure this concretely. First, we define both implementations:

```bash
#!/bin/bash

# Implementation A: Command substitution
level_to_num_subshell() {
  case "$1" in
    DEBUG) echo 0 ;;
    INFO)  echo 1 ;;
    WARN)  echo 2 ;;
    ERROR) echo 3 ;;
    *)     echo 1 ;;
  esac
}

# Implementation B: Name reference
level_to_num_nameref() {
  local -n result=$1
  local level_name=$2
  case "$level_name" in
    DEBUG) result=0 ;;
    INFO)  result=1 ;;
    WARN)  result=2 ;;
    ERROR) result=3 ;;
    *)     result=1 ;;
  esac
}
```

Now we can measure them using either the manual approach or our benchmarking library:

**Manual timing with `time`:**

```bash
iterations=10000

echo "=== Test A: Command Substitution ==="
time for ((i=0; i<iterations; i++)); do
  level=$(level_to_num_subshell "INFO")
done

echo ""
echo "=== Test B: Name Reference ==="
time for ((i=0; i<iterations; i++)); do
  level_to_num_nameref level "INFO"
done
```

**Using the benchmark library:**

```bash
source lib/benchmark.sh

test_subshell() {
  # We need this to pass the assignment into the benchmark function
  level=$(level_to_num_subshell "INFO")
}

benchmark "Command substitution" 10000 test_subshell
benchmark "Name reference" 10000 level_to_num_nameref level "INFO"
```

Typical results on a modern system:

```
=== Test A: Command Substitution (...) ===
real    0m1.890s
user    0m0.752s
sys     0m1.138s

=== Test B: Name Reference (local -n) ===
real    0m0.089s
user    0m0.087s
sys     0m0.002s
```

**Implementation B is over 20 times faster.** The difference is the fork overhead—thousands of subshell creations add up fast. That's not a rounding error; that's the difference between making coffee while your script runs and wondering if it crashed.

### When It Matters

This isn't premature optimization—it's avoiding stepping on performance landmines. It matters when:

- Your function is called in a loop (hundreds or thousands of times)
- Your script is a library sourced by many other scripts
- You're processing large data sets (log files, file lists, database dumps)
- You care about user experience (nobody likes watching scripts think really hard about simple tasks)

In a logging library, for example, level checking happens on every log call. If your application emits 5,000 log lines during a deployment, a 20x speedup means the difference between 500ms and 25ms spent on level resolution alone. That's the good kind of compound interest.

## Real-World Case Study: Optimizing the Logging Library

Last year, I wrote about [designing a modular Bash logging library](/posts/designing-modular-bash-functions-namespaces-library-patterns/), showing how to build reusable code with clean APIs and namespace discipline. The examples worked, but several readers pointed out a performance issue: the code used command substitution extensively, and they found that adapting it for heavy logging scenarios resulted in sluggish performance. Turns out, spawning thousands of subshells to log messages is like hiring a courier service to pass notes between adjacent cubicles.

This feedback prompted me to explore better patterns and measure the difference systematically.

### The Original Pattern

The initial examples used this pattern throughout:

```bash
_lb_resolve_level() {
  local token=${1^^}
  case $token in
    DEBUG) echo 0 ;;
    INFO)  echo 1 ;;
    WARN)  echo 2 ;;
    ERROR) echo 3 ;;
    *)     echo 1 ;;
  esac
}

# Caller code
level=$(_lb_resolve_level "INFO")
```

Every log call triggered multiple helper functions, each spawning a subshell. 
Testing revealed throughput of around 500-1,000 log lines per second—acceptable for light logging, but painful for verbose scripts.
If you work properly, you can probably avoid verbosity in the first place, but, sometimes, you just have to keep records.


{{< signup-cta >}}


### The Refactoring

The fix: replace all internal helpers with name reference-based functions:

```bash
_lb_resolve_level() {
  local -n __out=$1
  local token=${2:-}
  [[ -n $token ]] || return 1
  token=${token^^}
  case $token in
    DEBUG) __out=0 ;;
    INFO)  __out=1 ;;
    WARN)  __out=2 ;;
    ERROR) __out=3 ;;
    *)     __out=1 ;;
  esac
}

# Caller code
_lb_resolve_level level "INFO"
# $level now contains 1
```

The public API remained unchanged—users still called `lb_info "message"` exactly as before—but the internal implementation eliminated subshells entirely.

### Measuring the Improvement

Here's a simplified benchmark to measure the difference:

```bash
#!/bin/bash

# Assume the logging library is loaded here
# with functions: lb_info, lb_debug, etc.

iterations=10000
export LB_COLOR=never LB_LEVEL=INFO

echo "=== Benchmark: 10,000 INFO messages ==="
SECONDS=0
for ((i=0; i<iterations; i++)); do
  lb_info "Test message $i" > /dev/null 2>&1
done

duration=$SECONDS
throughput=$(( iterations / (duration + 1) ))

printf "Duration: %.3fs\n" "$duration"
echo "Throughput: ${throughput} lines/sec"
```

Results before and after the refactoring:

- **Before (subshell pattern)**: ~700 lines/sec
- **After (name reference pattern)**: ~8,500 lines/sec

**A 12x improvement** from a single architectural change. No algorithmic tricks, no C extensions—just eliminating unnecessary process forks.

### The Trade-Offs

Name references aren't always the right choice. They require Bash 4.3+ (released in 2014), so very old systems won't support them. They're also slightly less intuitive for beginners—command substitution reads more naturally at first glance.

But for library code and performance-critical paths, the trade-off is clear. Document the Bash version requirement and reap the benefits of modern features. If you're still supporting Bash 3.x in 2026, you have bigger problems than performance.

### From Example to Reality: The *log4b* Library

The optimized logging library isn't just a teaching example—it's the foundation of *log4b*, a production-ready Bash logging library currently in development. *log4b* applies these performance patterns throughout: name references for all internal helpers, minimal subshells, and careful benchmarking to validate every optimization.

The result is a library that handles thousands of log entries per second while maintaining clean APIs, structured output options (text and JSON), log rotation, and configurable formatting. The performance work described in this post made the difference between a toy example and a library ready for real-world use.

Building *log4b* required not just applying the optimization techniques, but also measuring them systematically—which is why benchmarking matters. Fast code that's hard to maintain is a false economy; *log4b* aims to be both performant and maintainable.

## Practical Benchmarking Checklist

When optimizing your own scripts, follow this workflow (and resist the urge to optimize everything at once):

1. **Identify the hot path**: Where does your script spend most of its time? Use `time` on different sections to find out.

2. **Write a focused benchmark**: Isolate the function or code block you suspect is slow. Create a minimal test that runs it 1,000 or 10,000 times.

3. **Measure the baseline**: Run the benchmark and record the time. This is your reference point.

4. **Refactor one thing**: Change the implementation (e.g., replace `$(...)` with name references). Keep changes small and targeted.

5. **Measure again**: Run the benchmark and compare. Did it improve? By how much?

6. **Validate correctness**: Fast but wrong is useless. Run your test suite to ensure behavior didn't change.

7. **Document the results**: Add a comment in your code explaining the choice. Future maintainers (including future you) will appreciate it.

## Wrapping Up: Measurement Matters

Bash scripting is often dismissed as "quick and dirty," but when scripts grow beyond throwaway one-liners, performance and maintainability become real concerns. Benchmarking isn't about micro-optimizing every loop—it's about making informed architectural choices. The difference between "it works" and "it works well" is usually just a few well-placed measurements.

The subshell vs. name reference comparison is just one example. There are other patterns worth exploring: avoiding `cat` in pipelines, using built-in string manipulation instead of `sed`, choosing the right loop construct, caching expensive lookups. Each has measurable impact.

The logging library example demonstrates that these techniques work in practice. A reusable Bash library can be both fast and clean—if you measure, iterate, and apply the right patterns.

## What's Next

If you found this useful, check out the [original post about designing modular Bash code](/posts/designing-modular-bash-functions-namespaces-library-patterns/). Together, they show how professional software engineering practices—modularity, benchmarking, iterative refinement—apply to shell scripting just as much as to any other language.

For those wanting to dive deeper into these topics—benchmarking, optimization patterns, testing strategies, and building production-ready Bash libraries—I'm currently working on a book, *Bash: The Developer's Approach*. It covers these techniques from first principles and uses *log4b* as a running example throughout, showing how to build robust, performant shell code that scales beyond quick scripts.

And now, go benchmark something! You might be surprised by what you find ;)

