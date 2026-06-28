+++
title = "FreeBSD's Base System Ships a Test Framework, and It's the Same One the OS Tests Itself With (ATF + Kyua)"
date = 2026-06-16
draft = false
summary = "/usr/tests exists on a stock install. The OS ships its own regression suite, run by a framework that's also in base, and nothing stops you pointing it at your own code. C tests, shell tests, isolation, cleanup, reporting, no packages."
freebsd_version = "15.1"
+++

*Part of the "FreeBSD Base: Things You Didn't Know It Could Do" series. Examples run on FreeBSD 15.1-RELEASE.*

Run this on a stock FreeBSD box:

```sh
ls /usr/tests
```

That directory is the operating system's own regression suite, shipped with the install. Thousands of tests for libc, the kernel, the userland tools. And the framework that runs them, ATF plus Kyua, is also in base, with nothing stopping you from pointing it at your own code. The OS tests itself with a test framework you can use for your projects, at the same version, documented in real man pages, with no `pip install`, no "which test runner," no dependency to pin.

I came to this the way most people probably do: I needed a test harness for a C project, started surveying the usual third-party options, and then noticed the machine already had one that the base system itself trusts. This post is what I wish I'd known at the start.

## Two layers: ATF writes, Kyua runs

The thing people miss at first is that this is two pieces with a clean split, and the split is the good part.

**ATF** (the Automated Testing Framework) is the set of libraries you *write* tests with. There's a binding per language: `atf-c` for C, `atf-c++` for C++, `atf-sh` for POSIX shell. You write test cases against one of those APIs and compile (or just mark executable, for shell) a test program.

**Kyua** is the separate runtime that *runs* those test programs. It doesn't just execute them. Per the ATF manual, the runtime engine "isolates the test programs from the rest of the system and ensures some common side-effects are cleaned up," and it's also "responsible for gathering the results of all tests and composing reports." That isolation-and-cleanup is the thing your hand-rolled `assert`-and-a-shell-script harness doesn't give you: each test case runs in its own controlled subprocess, and Kyua collects structured results rather than you scraping exit codes.

The separation matters because it means your tests describe *what* to check (ATF) while something else owns *how* to run them safely and report on them (Kyua). You write assertions; you don't write a test runner. There's a nice cross-Unix lineage here too: ATF began as a 2007 NetBSD Summer of Code project and grew OS-independent, which is why both FreeBSD and NetBSD ship it in base today.

## A real C test program

Here's a complete `atf-c` test program. It has four test cases: one that passes, one that checks string output, one that asserts on `errno`, and one that deliberately demonstrates the "known bug" workflow. Save it as `math_test.c`.

```c
#include <atf-c.h>
#include <fcntl.h>
#include <stdio.h>

/* A test case with a head (metadata) and a body (the actual checks). */
ATF_TC(addition);
ATF_TC_HEAD(addition, tc)
{
	atf_tc_set_md_var(tc, "descr", "Tests the addition operator");
}
ATF_TC_BODY(addition, tc)
{
	/* CHECK records a failure but keeps going. */
	ATF_CHECK_EQ(2, 1 + 1);
	ATF_CHECK_EQ(300, 100 + 200);
}

/* String comparison, with a custom failure message. */
ATF_TC(formatting);
ATF_TC_HEAD(formatting, tc)
{
	atf_tc_set_md_var(tc, "descr", "Tests snprintf output");
}
ATF_TC_BODY(formatting, tc)
{
	char buf[64];
	snprintf(buf, sizeof(buf), "a %s", "string");
	ATF_CHECK_STREQ_MSG("a string", buf, "got '%s' instead", buf);
}

/* errno checking: the second arg is the failing call. */
ATF_TC(open_missing);
ATF_TC_HEAD(open_missing, tc)
{
	atf_tc_set_md_var(tc, "descr", "open() of a missing file sets ENOENT");
}
ATF_TC_BODY(open_missing, tc)
{
	ATF_CHECK_ERRNO(ENOENT, open("does-not-exist", O_RDONLY) == -1);
}

/* The known-bug workflow: marked expect_fail, so the failure is
   reported as an *expected* failure rather than a red test. When
   someone fixes the bug, this starts failing loudly, which is the
   signal to update the test. */
ATF_TC(known_bug);
ATF_TC_HEAD(known_bug, tc)
{
	atf_tc_set_md_var(tc, "descr", "Reproduces bug #1234");
}
ATF_TC_BODY(known_bug, tc)
{
	atf_tc_expect_fail("bug #1234: off-by-one in the thing");
	ATF_CHECK_EQ(3, 1 + 1);
	atf_tc_expect_pass();
}

/* Register the cases. You never write main() yourself; this macro
   generates it. */
ATF_TP_ADD_TCS(tp)
{
	ATF_TP_ADD_TC(tp, addition);
	ATF_TP_ADD_TC(tp, formatting);
	ATF_TP_ADD_TC(tp, open_missing);
	ATF_TP_ADD_TC(tp, known_bug);

	return (atf_no_error());
}
```

A few things in there that are worth pulling out, because they're the parts that make ATF more than `assert.h`:

**`CHECK` vs `REQUIRE` is the fatal/non-fatal distinction.** The manual is precise: a `CHECK` "reports a failure as soon as it is encountered ... but the execution of the test case continues as if nothing had happened," while a `REQUIRE` will "immediately abort the test case as soon as an error condition is detected." Use `REQUIRE` when continuing makes no sense (a NULL pointer you're about to dereference), `CHECK` when you want to collect several independent failures in one run. Every macro comes in both flavors: `ATF_CHECK_EQ` / `ATF_REQUIRE_EQ`, `ATF_CHECK_STREQ` / `ATF_REQUIRE_STREQ`, and so on.

**The macro family is typed and specific.** Beyond plain `ATF_CHECK(expr)` there's `_EQ` for equality, `_STREQ` for strings, `_INTEQ` for integers, `_MATCH` for unanchored regex against a string, and `_ERRNO` for the very common "this call should fail and set this errno" pattern. Each has a `_MSG` variant taking a `printf`-style failure message. You're not writing `ATF_CHECK(strcmp(a,b) == 0)` and getting a useless "expression was false"; you use `ATF_CHECK_STREQ(a, b)` and the framework reports both strings.

**The known-bug workflow is genuinely clever.** `atf_tc_expect_fail("reason")` flips the test into a mode where a failure is reported as an *expected failure*, not a red build. The manual spells out why this is useful: when someone later fixes the underlying bug, the test "will start reporting a failure, signaling the developer that the test case must be adjusted." So a reproducer for a known bug becomes a tripwire that tells you the moment the bug is fixed. Setting the reason to a bug number makes it self-tracking.

Build and run it directly (the test program is a normal executable):

```sh
cc -o math_test math_test.c -latf-c
./math_test            # lists/runs cases with ATF's own interface
./math_test addition   # run a single case
```

## Shell tests: the faster on-ramp

For integration-style checks, the `atf-sh` binding is often quicker to reach for, and it's where ATF gets surprisingly pleasant. A shell test program starts with a specific shebang and follows the same head/body/register shape, using `_head`/`_body` functions named after each case:

```sh
#! /usr/bin/env atf-sh

atf_test_case greeting
greeting_head() {
	atf_set "descr" "the greet script prints a greeting"
}
greeting_body() {
	# atf_check runs a command and asserts on exit code + output.
	atf_check -s exit:0 -o match:"^Hello" greet --name world
}

atf_init_test_cases() {
	atf_add_test_case greeting
}
```

Note `atf_set` here where C used `atf_tc_set_md_var` (the shell names are shorter throughout). The star of the shell binding is `atf_check`, which wraps a command and asserts on its exit status, stdout, and stderr in one line. The matcher syntax is expressive:

```sh
# exit 0, nothing on stdout or stderr
atf_check -s exit:0 -o empty -e empty true

# exit 1, silent
atf_check -s exit:1 -o empty -e empty false

# stdout must equal the contents of a file
echo foo >expout
atf_check -s exit:0 -o file:expout -e empty echo foo

# stdout must match a regex
atf_check -s exit:0 -o match:"^foo$" -e empty ls

# save stdout to a file for later inspection
atf_check -s exit:0 -o save:stdout -e empty ls
```

For testing a command-line tool, this is close to ideal: one line per behavior, asserting exactly the exit code and output you expect, with the framework producing a clear diff when reality disagrees. Mark the script executable and it runs the same way the C program does.

## The payoff: point Kyua at your own project

Everything above runs a single test program by hand. Kyua is what turns a directory of them into a suite with real reporting, and crucially, it does not care whether the code is part of the FreeBSD base system or your own project sitting in `~/src`. You opt in with one file.

In the directory holding your test programs, drop a `Kyuafile`:

```
syntax(2)

test_suite("myproject")

atf_test_program{name="math_test"}
atf_test_program{name="greeting_test"}
```

Then:

```sh
kyua test                    # run everything, isolated, with cleanup
kyua report                  # human-readable summary of the last run
kyua report --verbose        # include details on failures
kyua report-junit            # JUnit XML, for CI ingestion
```

`kyua test` runs each program's cases in isolation, captures results into a database, and reports pass/fail/skip/expected-failure counts. `kyua report-junit` emits the JUnit XML that essentially every CI system understands, so your in-base test suite drops into GitHub Actions, GitLab, or Jenkins with no extra tooling. The same `kyua test` / `kyua report` you'd run by hand is exactly what the FreeBSD project runs against `/usr/tests` to validate the OS, so you're driving your project with the OS's own test driver.

This is the part that converted me. I didn't have to choose a test framework, vendor it, pin its version, or teach CI about it. The framework was already present, the same on every box, and the workflow I use on my own code is the workflow the base system uses on itself.

## Tour /usr/tests to learn by example

Because the OS ships its tests, you have thousands of worked ATF examples on disk. When you're unsure how to structure something, go read how libc tests it:

```sh
ls /usr/tests/lib/libc/
kyua test -k /usr/tests/lib/libc/stdio/Kyuafile   # run a slice of the OS's own suite
```

Real, non-toy test cases, written by the people who wrote the code under test, using the exact API you're using. It's the best documentation ATF has, and it ships with the system.

## The other-Unix contrast

Linux has no canonical base test framework, and the absence is structural rather than accidental. There's kselftest in the kernel tree, the Linux Test Project (LTP) as a separate large suite, and then a different harness per project and per distro. All of them are real and useful; none of them is *the* test runner that ships and versions with the system, that every base tool's tests are written against, that you can assume is present. The closest spiritual comparisons are language-specific (`cargo test`, `go test`), which are excellent but bound to one language and one toolchain. ATF's bet is orthogonal: a language-agnostic test format (C, C++, shell, with the same case/head/body/cleanup model across all three) plus one OS-level runner, shipped in base for any code on the box.

## Why "in base" matters here

A test framework you can rely on being present, identically, on every install changes what you build. There's no "add a testing dependency" step, no version skew between your test runner and CI's, no question of whether a fresh machine can run the suite. The OS trusts ATF and Kyua enough to gate its own releases on them, they're documented as real man pages (`atf-c(3)`, `atf-sh(3)`, `kyua(1)`), and the `/usr/tests` tree is a living example library. When the test harness is part of the audited base system rather than a dependency you chose, "can I test this here?" stops being a setup question and becomes a given.

---

*Man pages: `atf(7)` for the overview and cross-references, `atf-c(3)` and `atf-sh(3)` for the C and shell APIs, `atf-test-case(4)` for the language-independent test-case model (metadata variables like `require.user`, `timeout`, `require.progs`), `atf-check(1)` for the full `atf_check` matcher syntax, `kyua(1)` for the runner, and `tests(7)` for how the base test suite is organized. ATF and Kyua have been in FreeBSD base since late 2014.*
