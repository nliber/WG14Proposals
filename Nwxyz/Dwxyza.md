---
title: "Removing `strncpy`"
document: D0000a
date: 2026-07-06
audience: WG14
project: "Programming Language C"
author:
    - name: Nevin ":-)" Liber
      email: <nliber@anl.gov>
toc: true
toc-depth: 6
---
# Revision History
## D0000a
First proposed.

# Motivation and Scope

`strncpy` is a terrible, error-prone, memory-unsafe, poorly-named, insecure function.  It is long past time we remove it.


## History

It appears to have originated in Version 7 Unix (1979), 46 years ago...

While not in the first edition of
*[The C Programming Language](https://archive.org/details/cprogramminglang0000kern)*
by Brian W. Kernighan and Dennis M. Ritchie (1978),
it was added when ANSI C was first standardized:

> **4.11.2.4 The `strncpy` function**
>
> `strncpy` was initially introduced into the C library to deal with
> fixed-length name fields in structures such as directory entries. Such
> fields are not used in the same way as strings: the trailing null is
> unnecessary for a maximum-length field, and setting trailing bytes for
> shorter names to null assures efficient field-wise comparisons.
> `strncpy` is not by origin a "bounded `strcpy`," and the Committee has
> preferred to recognize existing practice rather than alter the function
> to better suit it to such use.
>
> — [*Rationale for American National Standard for Information Systems —
> Programming Language — C*,
> §4.11.2.4](https://www.lysator.liu.se/c/rat/d11.html#4-11-2-4)

### Personal Experience

Back in the early 2000s (long before having lofty aspirations of becoming a member of this committee),
this author was debugging a production issue directly caused by the use of `strncpy`.

The developer who used it didn't realize: 

- `strncpy(s1, s2, n)` *always* writes `n` bytes to the memory pointed to by `s1`.
In other words, even though `strcpy(s1, s2)` only writes `strlen(s2) + 1` characters,
`strncpy`, whose name only suggests that it takes an additional length parameter compared with `strcpy`,
may write *more* characters than `strcpy` for the same `s2`.

- `strncpy(s1, s2, n)` does not guarantee that the memory pointed to by `s1` is null-terminated.
In other words, even though `strncpy(s1, s2, n)` is in `<string.h>` and starts with the prefix `str`,
it doesn't guarantee that the resultant `s1` (and by extension, what it returns) represents a string.


Because it was so error-prone and the cause of a field issue, the company policy changed to ban the use of `strncpy`.

### Common Vulnerabilities and Exposures (CVEs)

A simple search finds that `strncpy` has been the cause of no less than **fifteen** CVEs:

- [CVE-2026-40334](https://nvd.nist.gov/vuln/detail/CVE-2026-40334) (2026):
  An insecure `strncpy` call in libgphoto2 (ptp_unpack_Canon_FE) leaves a
  13-byte camera filename buffer unterminated when the source string matches
  the limit, causing an out-of-bounds read.
- [CVE-2026-23749](https://nvd.nist.gov/vuln/detail/CVE-2026-23749) (2026):
  An insecure `strncpy` call in the Golioth Firmware SDK blockwise transfer
  feature fails to force a trailing null byte on maximum-length paths,
  leading to an out-of-bounds read and device crash.
- [CVE-2026-9265](https://nvd.nist.gov/vuln/detail/CVE-2026-9265) (2026):
  An insecure `strncpy` call in Crypt::OpenSSL::PKCS12 for Perl copies a
  UTF8STRING ASN.1 attribute into a heap buffer without a NUL terminator,
  causing downstream `strlen` calls to read out-of-bounds heap memory.
- [CVE-2025-6334](https://nvd.nist.gov/vuln/detail/CVE-2025-6334) (2025):
  Critical vulnerability in D-Link DIR-867 1.0. The strncpy function in the
  Query String Handler causes a stack-based buffer overflow.
- [CVE-2024-54809](https://nvd.nist.gov/vuln/detail/CVE-2024-54809) (2024):
  An insecure `strncpy` call in Red Hat's underlying target utilities
  copies localized environment strings into fixed arrays without adding a
  trailing NUL byte, resulting in a heap-based memory disclosure flaw.
- [CVE-2024-39480](https://nvd.nist.gov/vuln/detail/cve-2024-39480) (2024): In
  the Linux kernel, strncpy() was used incorrectly in kdb (kernel debugger)
  during tab-completion, using the source size instead of the destination
  size, leading to buffer overflows.
- [CVE-2024-27224](https://nvd.nist.gov/vuln/detail/cve-2024-27224) (2024): A
  vulnerability in strncpy.c allows a possible out-of-bounds write due to a
  missing bounds check, potentially leading to local privilege escalation.
- [CVE-2024-3119](https://nvd.nist.gov/vuln/detail/CVE-2024-3119) (2024): A
  buffer overflow exists in sngrep (since v0.4.2) because strncpy is used to
  copy 'Call-ID' SIP headers without proper length checks, allowing remote
  code execution.
- [CVE-2020-13995](https://nvd.nist.gov/vuln/detail/CVE-2020-13995) (2020):
  An insecure `strncpy` call within the extract75 NITF parser component
  improperly computes destination capacity vs source string length, causing
  an out-of-bounds array write when handling malicious image metadata
  headers.
- [CVE-2019-11365](https://nvd.nist.gov/vuln/detail/CVE-2019-11365) (2019): An
  insecure strncpy call in atftpd (atftp 0.7.1) allows a remote attacker to
  trigger a stack-based buffer overflow with a specially crafted 3-byte
  packet.
- [CVE-2019-9125](https://nvd.nist.gov/vuln/detail/CVE-2019-9125) (2019):
  An insecure `strncpy` call in D-Link DIR-878 routers results in an
  unauthenticated, stack-based buffer overflow vulnerability that can be
  remotely triggered via the HNAP_AUTH HTTP header.
- [CVE-2018-3878](https://nvd.nist.gov/vuln/detail/CVE-2018-3878) (2018):
  An insecure `strncpy` call in the credentials handler of the video-core
  HTTP server on Samsung SmartThings Hub devices overflows a fixed 16-byte
  stack buffer when an attacker passes an oversized "region" string.
- [CVE-2018-3874](https://nvd.nist.gov/vuln/detail/cve-2018-3874) (2018): An
  exploitable buffer overflow in Samsung SmartThings Hub firmware (0.20.17)
  caused by improper strncpy usage.
- [CVE-2004-0500](https://nvd.nist.gov/vuln/detail/CVE-2004-0500) (2004):
  An insecure `strncpy` call in Pidgin (Gaim) MSN protocol plugins fails
  to verify array limits before copying data, allowing remote attackers to
  trigger a buffer overflow via peer-to-peer messages.
- [CVE-2003-0465](https://nvd.nist.gov/vuln/detail/cve-2003-0465) (2003): The
  kernel strncpy in Linux 2.4/2.5 did not null-pad the buffer on non-x86
  architectures, leading to information leaks.

### Linux Kernel

Most recently (2026-06-19), the Linux kernel has removed `strncpy` from its codebase.
Why?  Linus Torvalds wrote in [Linux Kernel Commit: strncpy removal](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1a3746ccbb0a97bed3c06ccde6b880013b1dddc1):

> strncpy() has been removed from the kernel. All former callers have
> been migrated to safer alternatives.
>
> strncpy() did not guarantee NUL-termination of the destination buffer,
> leading to linear read overflows and other misbehavior. It also
> unconditionally NUL-padded the destination, which was a needless
> performance penalty for callers using only NUL-terminated strings. Due
> to its various behaviors, it was an ambiguous API for determining what
> an author's true intent was for the copy.

# Design Decisions

## Isn't this a breaking change?

Yes it is (but it isn't a quiet change), which means we have to weigh the tradeoff of that against doing nothing.
If we continue to leave it in, we *know* it leads to:

- Application vulnerabilities.
- Incorrectly reasoned about code with respect to security and reliability.
- Impediment to writing memory safe and functional safe code.
 
## Why not mitigate it via education?

We've had over 4½ decades attempting to educate the world on this.  Education hasn't worked as a general mitigation strategy.
There have been endless warnings in compiler diagnostics, static analysis tools, textbooks, coding standards, blog posts and conference talks.

- No amount of education can make up for a function that is easy to use incorrectly and hard to use correctly.
- Education doesn't make up for the implicit assumptions in its name.
The name `strncpy` implies it is safer to use than `strcpy`, even though it is anything but safer.
- Education doesn't scale.  Every new generation of C developer ends up learning this lesson for themselves.

## Why not add a new string type with a safe API?

While that would be nice, that is likely to take years (if not decades),
has no guarantee of actually being agreed upon and adopted by the members of WG14,
and still doesn't address this problem, as it will stick around as long as
`strncpy` is a WG14-blessed function.

Specifically:

- Timeline.  Doing so will take years.  In general, this is the right solution for changing an international standard
that millions of developers are reliant upon.  But the problems of `strncpy` are hazardous to production code
*today*, and we should both acknowledge and address that fact.

- No guarantee of adoption.  There are many hard questions around designing a string class
(or string classes, since it is unlikely we can come up with a universal one that meets all needs),
and no assurance that WG14 can ever come to consensus on a solution.  Things like memory ownership,
allocation strategies, interaction with existing APIs, etc will all be contentious and take
much effort to come to consensus.

- It does not solve the problem.  Unless and until `strncpy` is removed, people will continue to use it.
Developing a new string type or types is independent of removing `strncpy`.  If it is in the standard,
the world views it as WG14 endorsing it.

## We could just deprecate it.

We could, but that is a weak choice.  How many more years/decades should WG14 ignore that this has been a real world
problem for decades that should be addressed?

- How many more decades before removal?  If not now, at what point does WG14 remove it, acknowledging that
there are real, serious problems caused by `strncpy` and the time for half measures has passed?

- Deprecation alone does not work.  We saw that with `gets`.  Even though that was deprecated in C99,
it still showed up in new code until it was removed over a decade later in C11.

- Deprecation puts the burden on developers, not WG14.  We leave it up to every developer and every organization
to decide this is a flawed function, and we know the problems that causes.

We are not lacking evidence that this function should be removed;
the only thing lacking is our willingness to act on it.

## Shouldn't we deprecate it and replace it with something better?

Again, that would be nice, but really isn't much different than the problems we
have introducing a new string type.  

The Linux Kernel replaced `strncpy` with *seven* different functions:

- `strscpy()` when the destination must be NUL-terminated.
- `strscpy_pad()` when the destination must be NUL-terminated and 
  zero-padded (for example, for structs crossing privilege boundaries).
- `memtostr()` for NUL-terminated destinations copied from
  non-NUL-terminated fixed-width sources, with the `__nonstring`
  attribute on the source.
- `memtostr_pad()` for the same case, but with zero-padding.
- `strtomem()` for non-NUL-terminated fixed-width destinations, with
  the `__nonstring` attribute on the destination.
- `strtomem_pad()` for non-NUL-terminated destinations that also need
  zero-padding.
- `memcpy_and_pad()` for bounded copies from potentially unterminated
  sources where the destination size is a runtime value.

How long would it take WG14 to agree on something similar?  Certainly not in time for C29.

I'm not saying we shouldn't pursue a replacement (we should),
but we shouldn't wait for it.  Perfect is the enemy of good.

## Is this just a token effort?

This is a small, achievable step, which is its value.

- It is not insignificant.  This is one of the most misused functions in standard C.
- This is a small effort that has a big benefit.
Removing `strncpy` eliminates an entire class of misuse that has persisted for decades.
- It signals to the community that we are taking memory safety problems seriously.

## Implementors will keep it around anyway.

- They almost certainly will.  This is normal and expected.
But they can put it behind a compiler flag or equivalent should they so choose,
and point to the standard as a strong reason for code, especially new code, not to use it.
- Removal allows them to make it opt-in. Developers will have to deliberately choose to use it.
- Correct, portable code can no longer use it.  Developers and organizations that care about
those things can also point to the standard as a reason to stop using it.

## We have precedent with `gets`.

`gets` was removed, and the sky did not fall.  All the concerns about breaking existing code (it did),
education, what implementations would do, etc., were valid but ultimately proved to be manageable.

- The removal of `gets` is considered to be an unqualified success.
- The case to remove `strncpy`, in some ways, is even stronger.  It appears to be safer (because it is bounded),
but it turns out to be subtly unsafe.
- The objections are the same.  There are no new arguments being made against removing `strncpy` that weren't
being made against removing `gets`.
- WG14 established a principle with removing `gets` that it will not
perpetuate functions that actively cause harm.  That principle shouldn't just be a one-off.

## What about `strncpy_s`?

It is out of scope for this proposal, as it does not share the same defects as `strncpy`.

- It guarantees null-termination of the destination buffer.
- It validates its parameters at runtime.
- It returns an error code on failure.

The status of Annex K is an independent question that should be evaluated on its own merits.

# Proposed Wording

All edits are relative to [N3886](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3886.pdf).

## Modify §7.28.2 [Copying functions]

In §7.28.2, remove subclause 5:

::: rm

7.28.2.5 The `strncpy` function

Synopsis

```c
#include <string.h>
char *strncpy(char * restrict s1, const char * restrict s2, size_t n);
```

Description

The `strncpy` function copies not more than `n` characters (characters
that follow a null character are not copied) from the array pointed to
by `s2` to the array pointed to by `s1`.^329)^ If the array pointed to by
`s2` is a string that is shorter than `n` characters, null characters
are appended to the copy in the array pointed to by `s1`, until `n`
characters in all have been written.

Returns

The `strncpy` function returns the value of `s1`.

:::

Remove footnote 329:

::: rm

^329)^ Thus, if there is no null character in the first `n` characters of the array pointed to by `s2`,
the result will not be null-terminated.

:::

## Modify §B.27 [String handling `<string.h>`]

Remove the following declaration:

::: rm

```c
char *strncpy(char * restrict s1, const char * restrict s2, size_t n);
```

:::

## Modify §M.2 [Changes from previous revisions]

Add the following bullet to the list of changes:

::: add

The `strncpy` function has been removed.

:::

## Modify the Index

Remove the entry for `strncpy`.

# Conclusion

`strncpy` may have been a reasonable function for directory entries in 1979.
It is not a reasonable function for a security-conscious world in 2026, let alone 2029.

The cost of removal is a compiler error and a one-time code change. The
cost of inaction is measured in CVEs.

When the committee removed `gets`, it established a principle: the
standard should not endorse functions that are known to cause systematic
harm. `strncpy` meets that bar. The principle should be applied
consistently.

# Acknowledgements
Nevin Liber was supported by the Office of Science, U.S. Department of Energy, under contract DE-AC02-06CH11357.

All the (non-quoted) ideas are mine or based on discussions,
but I did use Claude Opus 4.6 and a thesaurus to inspire more compelling ways to phrase and present them.

