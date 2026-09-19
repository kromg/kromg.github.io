---
title: "Let it speak: when code should speak for itself - but it does not"
date: 
description: "Good written code should be speaking to the human who reads it, like an open book. Sometimes, though, this book is written in a poor language. "
tags: ["bash", "clean code", "expressive code", "code bugs", "code improvement"]
categories: ["Shell Scripting", "Clean code"]
featured_image: "/images/shebang-cover.png"
---


I'm writing a book about how to write Bash code, and how to write it well. 

That book is all about the little things that I would have liked to know when I started putting together my first scripts, 
contraptions that did what they had to do, but were not pleasant to look to (well, not too much). 

One of those contraptions has made it to these days, buried deep down into one of my oldest backups, and I found it while 
spelunking in search for examples of what _not_ to do in a script. 

I must say, in my defence, that finding it has not been easy: I've always had a taste for well written code, even before 
I had any idea about what "well written" really means, so it has taken me a while to find something ugly enough to be worty of this
improvised hall of shame - yet, here we are: a small log rotation script for a Tomcat instance, that's full of things I wouldn't want 
to see in a script (I'll attach the whole script at the end of the article so you can contemplate it in all its magnificence). 

We'll dissect it here and enumerate all its sins, solemnly swearing never to repeat those mistakes again. Then, in the following 
articles of this series, we'll go on and refactor this poor little guy, until it will become a nice piece of good quality software, 
something we can proudly show to grandma (even though she won't probably understand what's going on).   

Let's take a look at this critter, analyzing all the things that we should have avoided. 


## \#0 Indenting with tabs

The whole script is indented with tabs - which was the style, at the time. While it looks good when you have tabs set at 2 or 4 characters,
it becomes rapidly unbearable to look at, if tabs get expanded to 8 chars. Definitely a hard no for me. 

![Tabs for indent](/images/tab_for_indent.png)


## \#1 Starting with the wrong foot

```bash
#!/bin/bash
```

Ok, this is good enough for the old systems where this script was meant to run onto, but `#!/usr/bin/env bash` would have been 
as effective, and much safer. 


## \#2 Naming conventions and how to break them

One of the most important things to write code that does not surprise the reader is to adhere to standard naming conventions... 
or, at least, to be consistent! 

```bash

  Ver="1.1"

dirlogs="/opt/tomcat/logs"
usr="appuser"
DATE="`date -d yesterday +%Y-%m-%d`"
num="30"
```

Here we have a bunch of configuration variables: 
- some are uppercase, others lowercase (I could at least have chosen one naming convention, and stuck to it!)
- one is capitalized (??) - for which reason, I don't know. Guess what? Since I was not happy enough, to add more entropy 
  that one also has its own indentation. Because: why not? Right?
- one of them has a name which clashes with the name of a well known directory in the filesystem, so the reader can be left 
  guessing what we're talking about...
- ...which is what happens with the `num` variable as well: any idea what that will be used for? No? How comes? 
  GIVE VARIABLES SELF-EXPLANATORY NAMES (FFS)! 

Bonus: 
- if you look at the rest of the script, you'll notice that `usr` variable is **never used**!  

I must say I'm pretty impressed with the amount of entropy I was able to squeeze into 5 lines of code. Kudos to me for 
giving myself such a wonderful example to work on, years later. 

Let's continue. 

## \#3 Call it first, define it later does **not** work

The script goes on checking a prerequisite. Good! This is wonderful practice: check requirements early, handle errors 
before they happen (we've talked about it [here](/posts/error-first-pattern-writing-self-documenting-bash/)). All good, right? Well... 

```bash
if [ ! -d "$dirlogs" ]; then 
	log "ERROR: directory $dirlogs does not exist, exiting."
	exit 1
fi
```

...except if you call a logging `log` function that has not been defined yet. That's not gonna work, no. 

The original sin, here, is mixing runtime code with definition code (the functions that follow). 

A well structured script always follows the same structure: 

1. shebang;
2. a brief comment header with the purpose of the script, the copyright attribution, and the license; 
3. imports - that is: sourcing external files. You may want to encase this block between a `set -e` and an optional `set +e`, 
   so if anything goes wrong in this phase, the script stops;
4. internally defined functions - which should be kept at the minimum: the more you push functions into libraries, the 
   less you'll be repeating yourself and the more you'll reuse your code;
5. global configuration variables - if necessary. Include here the script's prerequisites like executable you may want to 
   search for in the system, or configure statically. If the configuration is lengthy, consider exporting it into an external 
   file as well - parsed, not sourced;
6. the main runtime of the script - which includes the prerequites checks, early-exit on missing required executables/directories/etc.

This ensures that each block can build on the previous one, and that _you'll actually call functions **after** they have been defined_. 


## \#4 Useless use of `cat` - guilty as charged

The logging function is not bad in and of itself, it's ready to log any message passed to it _or_ the output produced
by a command, line by line. A bit overkilling, maybe, but it makes logs more consistent (it's a technique that's also useful
if you run commands that may take a while, because it adds a timestamp to the output - this might come in handy for 
specific applications):

```bash
log() {
	if [ "$#" != "0" ]; then 
		echo -e "`date '+%Y-%m-%d-%H:%M:%S.%N'` :: $*"
		return 0
	else
		cat /dev/stdin | while read line; do 
			echo -e "`date '+%Y-%m-%d-%H:%M:%S.%N'` :: $line"
		done
		return 0
	fi
	return 1
}
```

The `cat /dev/stdin`, though, is totally unwarranted. A plain `while read line` _already_ reads from STDIN, and has the advantage 
of _not_ spawning a subshell. If a function like this was called a lot of times, this could impact performance sensibly. 

The `return` statements are equally redundant, and I could have saved 3 lines and a bunch of keystrokes by just avoiding them. 

If we want to be absolutely picky, the `echo -e` could be better substituted by `printf`, which gives us more control on the 
output, and will not accidentally interpret escape sequences if they were present in the content of the `$line` variable. 


## \#5 Shadowing system command names - bad and dangerous

What follows is a series of logging functions: 

```bash
info() {
	log "INFO: $*"
}

# [...]
```

`info` is a system command - shadowing it is bad practice: it causes confusion at best, possible unwanted behavior at worst. 
Try to avoid clashing with system command names. 


## \#6 Inconsistent redirection, wrong parameter expansion, more redundancy

```bash
is_in_use() {
	/sbin/fuser -s $*
	return $?
}
```

This function is not consistent with the rest of the script: 
- do we want to log all the output? Then pipe to the `log` function, it was done on purpose;
- are we not interested in the output of the `fuser` command? Well, then discard it: `&> /dev/null`

The `return` statement is completely redundant, again.

If you think the function itself is redundant, though, think twice: `if is_in_use "$src"; ...` reads 
way better than `if /sbin/fuser -s $src &> /dev/null; ...`, and if you wanted to change the implementation 
of the `is_in_use` function, you would not need to change the places where it's used (provided that the 
semantics of it stayed the same). 

Parameter expansion, however, is incorrect: `"$@"` should be used in this case in place of `$*` - it may not be 
important in _this_ specific case, given how the function is used (and given the fact that the `$src` argument
passed to it is unquoted, which is another problem entirely), but writing a function in the correct way _upfront_ makes 
it easier to move it to a shared library later, if needed. Not to mention the fact that habits stick, 
so they better be _good_ habits, rather than _bad_ habits. Use `"$@"` (quoted, always). 


## \#7 Can you spot the bug?

```bash
rename() {
	local src="$1"
	local dest="$2"
	info "renaming $src to $dest"
	if is_in_use $src; then 
		warn "file $src is in use, cannot rotate it"
		return 1
	else
		mv -fv $src $dest 2>&1 | log
		res="$?"
		[ "$res" != "0" ] && err "cannot rename file $src" && return 1
		return 0
	fi
}
```

Here we go, again: 
- `rename` is a system command - this function shadows it, and it should not;
- capturing `$?` and checking it if it's `0` is pointless, a plain `if <command>` does the trick and reads way better, 
  especially if you name a function in the proper way, and you use it in the place of the `<command>`. 

Speaking of which, if you want to have access to a bunch of ready-made predicates you can source and use in your scripts, 
check out the predicates you can find in my toolkit: 


{{< book-hook >}}



But the real problem, here, is: can you spot the bug in the `rename` function? 

Here, I'll zoom in for you: 

```bash
		mv -fv $src $dest 2>&1 | log
		res="$?"
		[ "$res" != "0" ] && err "cannot rename file $src" && return 1
```

That (ugly) `res="$?"` is checking the exit value of the `log` function, not that of the `mv` command! 
This means that, if the `mv` failed, we would never handle the problem correctly. 

This is the kind of subtle bugs that hide themselves in plain sight, when your coding style is not 
consistent - and when you don't write tests for your code, but this is a story for a completely different 
series of articles (stay tuned!). 

The double `&&` is another recipe for disaster: the moment one of the calls in the chain behaves in 
an unexpected way, you can incur in unwanted behavior. Better to avoid that syntax at all costs. 


## \#8 Comments are the Evil

Comments are the ultimate evil. Every time you put a comment in your code, you should treat that as a 
failure to express yourself in code. Let it speak for itself! 

Comments like the `# MAIN` you meet below here may seem innocuous: they are just a visual clue to 
make it easier to spot the end of the "header" block of the script - where you do imports, definitions,
configuaration - and the start of the main runtime. 

However, **why should you need such visual clues, if you kept your scripts minimal and exported 
as much code as possible into reusable libraries?**

```bash
################################################################################
# MAIN
################################################################################

log "================================================================================"
log " `basename $0` :: Ver. $Ver"
log "================================================================================"
log ""
```

Anyway, this is not the worst... I would probably create a reusable `print_header` function instead 
of firing 4 `log` calls in a row, OK, but the worst is what follows:

```bash
# Step 1: first thing to do is rename catalina.out:
info "renaming catalina.out"
fname="catalina.out"
ftarget="catalina.${DATE}.out"
file="$dirlogs/$fname"
tgt="$dirlogs/$ftarget"

rename $file $tgt
```

**A full block of unreadable code preceded by a comment, which should have been a function name!**

But, wait, we're going to see this other 3 times, brace yourselves. 


## \#9 Useless use of perl, or grep, or both! (And more stuff that doesn't fit in a title)

This whole script could have been written in Perl, and I would not be here ranting about it. Right?

I used to love Perl (still do, but it's not that popular nowadays - I haven't written a single perl line in years), 
and abused of it into my scripts: 

```bash
# Step 2: look for other files without a date in the name and rename them
info "looking for other files to rename"
find $dirlogs -type f | perl -ne 'print unless /^.*\/.+?\.\d{4}-\d{2}-\d{2}\..*$/;' | grep -v '\.bz2$' | while read file; do
	fullname="`basename $file`" 
	name="${fullname%\.*}"	# remove the extension: the last "." and everything after it
	ext="${fullname#${name}\.}"	# retrieve the extension
	dir="`dirname $file`" 	# in case the file is in a subdirectory of $dirlogs
	newname="${name}.${DATE}.$ext"

	rename $file $dir/$newname

done
```

And, then, there are the comments...

![Comments everywhere](/images/comments_everywhere.png)


I don't even have words for this. 

No, I lied, I have: OMG 🙈 

Let's see what sins I have committed here: 
1. comments, comments everywhere. There are at least 3 different ways we could have avoided those comments, 
   but the bottom line is always the same: "Use the functions, Luke!";
2. `find | perl | grep` - I should have thrown in also `awk` and `sed` just to make it complete. A plain `find`
   is perfectly capable of handing all of that on its own, and wrapping it in a nice `find_logs_not_rotated_yet` function
   would have made clear what was going on;
3. bonus: moving the call to the end of the `while` loop with redirection, would have made the loop easier to spot: 
   `while read file; do ... ; done < <(find_logs_not_rotated_yet)`
4. wrapping the 3 commented lines in their own functions would have spared us the disgrace of placing those three comments inline;
5. second bonus: wrapping those 3 functions into a new function and calling it like this: 
   `rename "$file" "$(get_rotated_name_for "$file")"` would make the intent way more clear. The implementation details can go in the 
   function;
6. unquoted variables complete the list of deadly sins, here. It's true that the naming conventions of log files in this context 
   _imply_ they won't contain spaces or special characters, but why risking, when the cost is a mere `""` around variables?? 
7. `basename`, `dirname`, `perl`, `grep`: a lot of command substitution going on here, where Bash syntax would have done equally well, 
   and would have saved some subprocess spawning; to make this worse, I used backticks instead of `$(...)` for the command substitution. 


## And so on, and so forth...

The rest of the script is a repetition of the sins above: the last two blocks of code strongly resemble the previous one, 
with an additional `if/else` that could have been skipped by making a clever use of a checker function in combination with 
`continue` (e.g. `warn_if_in_use && continue`)

```bash
# Step 3: look for all uncompressed files in $dirlogs and compress them
info "proceeding with compression of files not yet compressed"
today="`date +%Y-%m-%d`"
find $dirlogs -type f ! -name '*.bz2' | grep -v "$today" | while read logfile; do
	if is_in_use $logfile; then 
		warn "file $logfile is in use, cannot compress it"
	else
		bzip2 -9 $logfile
		res="$?"
		if [ "$res" == "0" ]; then 
			info "compressed file $logfile"
		else
			err "cannot compress file $logfile"
		fi
	fi
done

# Step 4: remove old logs
info "proceeding with removal of logs older than $num days"
find $dirlogs -type f -mtime +$num | while read oldfile; do
	if is_in_use $oldfile; then 
		warn "file $oldfile is in use, cannot remove it"
	else
		rm -f $oldfile 2>&1 | log
		res="$?"
		if [ "$res" == "0" ]; then
			info "removed file $oldfile"
		else
			err "cannot remove file $oldfile"
		fi
	fi
done
```


## Conclusions

In conclusion, the script `# MAIN` block should have read like: 

```bash

# MAIN 

print_header
rotate_catalina
rotate_other_logs
compress_rotated_logs
cleanup_old_logs

```

This is the highest level of abstraction: at this level, the code should only tell the reader what's happening - at a bird's eye view. 

Then, each function should tell its own story, calling other functions, and so on and so forth, until we reach the lowest-level functions, that deal with 
the real commands and statements, and should not be longer than 3-5 lines at most. 

This is how you make the code speak for itself, and how you factor it in smaller and smaller chunks, until you get a collection of reusable pieces 
of code, that you can finally export into shared files that you can (re)use across multiple scripts. 

I understand this is perhaps not always achievable - I've been in places where each script must be self-contained and you 
can't rely on much infrastructure - but it's still the foundation of a mindset that allows you to write clear, clean, maintainable (and testable) code. 


---

In the next episodes, we'll start to put all of this into practice, until this script will have become a 
small piece of craft we will be produd of (and we'll want to show grandma!). Stay tuned! 

And, in the meantime, check out the toolkit! 

See ya next time!


{{< book-hook >}}

 
---


## The whole thing

Here's the whole thing together, for reference: 

```bash
#!/bin/bash

  Ver="1.1"

dirlogs="/opt/tomcat/logs"
usr="appuser"
DATE="`date -d yesterday +%Y-%m-%d`"
num="30"

if [ ! -d "$dirlogs" ]; then 
	log "ERROR: directory $dirlogs does not exist, exiting."
	exit 1
fi

log() {
	if [ "$#" != "0" ]; then 
		echo -e "`date '+%Y-%m-%d-%H:%M:%S.%N'` :: $*"
		return 0
	else
		cat /dev/stdin | while read line; do 
			echo -e "`date '+%Y-%m-%d-%H:%M:%S.%N'` :: $line"
		done
		return 0
	fi
	return 1
}

info() {
	log "INFO: $*"
}

warn() {
	log "WARNING: $*" > /dev/stderr
}

err() {
	log "ERROR: $*" > /dev/stderr
}

is_in_use() {
	/sbin/fuser -s $*
	return $?
}

rename() {
	local src="$1"
	local dest="$2"
	info "renaming $src to $dest"
	if is_in_use $src; then 
		warn "file $src is in use, cannot rotate it"
		return 1
	else
		mv -fv $src $dest 2>&1 | log
		res="$?"
		[ "$res" != "0" ] && err "cannot rename file $src" && return 1
		return 0
	fi
}

################################################################################
# MAIN
################################################################################

log "================================================================================"
log " `basename $0` :: Ver. $Ver"
log "================================================================================"
log ""

# Step 1: first thing to do is rename catalina.out:
info "renaming catalina.out"
fname="catalina.out"
ftarget="catalina.${DATE}.out"
file="$dirlogs/$fname"
tgt="$dirlogs/$ftarget"

rename $file $tgt


# Step 2: look for other files without a date in the name and rename them
info "looking for other files to rename"
find $dirlogs -type f | perl -ne 'print unless /^.*\/.+?\.\d{4}-\d{2}-\d{2}\..*$/;' | grep -v '\.bz2$' | while read file; do
	fullname="`basename $file`" 
	name="${fullname%\.*}"	# remove the extension: the last "." and everything after it
	ext="${fullname#${name}\.}"	# retrieve the extension
	dir="`dirname $file`" 	# in case the file is in a subdirectory of $dirlogs
	newname="${name}.${DATE}.$ext"

	rename $file $dir/$newname

done

# Step 3: look for all uncompressed files in $dirlogs and compress them
info "proceeding with compression of files not yet compressed"
today="`date +%Y-%m-%d`"
find $dirlogs -type f ! -name '*.bz2' | grep -v "$today" | while read logfile; do
	if is_in_use $logfile; then 
		warn "file $logfile is in use, cannot compress it"
	else
		bzip2 -9 $logfile
		res="$?"
		if [ "$res" == "0" ]; then 
			info "compressed file $logfile"
		else
			err "cannot compress file $logfile"
		fi
	fi
done

# Step 4: remove old logs
info "proceeding with removal of logs older than $num days"
find $dirlogs -type f -mtime +$num | while read oldfile; do
	if is_in_use $oldfile; then 
		warn "file $oldfile is in use, cannot remove it"
	else
		rm -f $oldfile 2>&1 | log
		res="$?"
		if [ "$res" == "0" ]; then
			info "removed file $oldfile"
		else
			err "cannot remove file $oldfile"
		fi
	fi
done

```bash