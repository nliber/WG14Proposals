---
title: "Removing `strncpy`"
document: D0000a
date: 2026-07-06
audience: WG14
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

`strncpy` is a terrible, error-prone, memory-unsafe, insecure function.  It's long past time we remove it.

## Experience

Back in the early 2000s (long before becoming a member of this committee),
this author was debugging a production issue directly caused by the use of `strncpy`.

The developer who used it didn't realize: 

- `strncpy(s1, s2, n)` *always* writes `n` bytes to the memory pointed to by `s1`.
In other words, even though `strcpy(s1, s2)` only writes `strlen(s2) + 1` characters,
`strncpy`, whose name only indicates that it takes an additional length parameter compared with `strcpy`,
may write *more* characters than `strcpy` for the same `s2`.

- `strncpy(s1, s2, n)` does not guarantee that the memory pointed to by `s1` is null-terminated.
In other words, even though `strncpy(s1, s2, n)` is in `<string.h>`,
it doesn't guarantee that the resultant `s1` (and by extension, what it returns) represents a string.

Because it was so error-prone and the cause of a field issue, the company policy changed to ban the use of `strncpy`.

## Common Vulnerabilities and Exposures (CVEs)

A simple search finds that Since its inception, `strncpy` has been the cause of at least a **dozen** CVEs:

[[CVE-2026-40334](https://nvd.nist.gov/vuln/detail/CVE-2026-40334)] (2026)
:   An insecure `strncpy` call in libgphoto2 (ptp_unpack_Canon_FE) leaves a 13-byte camera filename buffer unterminated when the source string matches the limit, causing an out-of-bounds read.

[[CVE-2026-23749](https://nvd.nist.gov/vuln/detail/CVE-2026-23749)] (2026)
:   An insecure `strncpy` call in the Golioth Firmware SDK blockwise transfer feature fails to force a trailing null byte on maximum-length paths, leading to an out-of-bounds read and device crash.

[[CVE-2025-23395](https://nist.gov)] (2025)
:   An insecure `strncpy` call in GNU Screen's logfile_reopen feature fails to account for maximum string boundary lengths, allowing local attackers to overwrite memory buffers and escalate privileges to root.

[[CVE-2026-9265](https://nvd.nist.gov/vuln/detail/CVE-2026-9265)] (2026)
:   An insecure `strncpy` call in Crypt::OpenSSL::PKCS12 for Perl copies a UTF8STRING ASN.1 attribute into a heap buffer without a NUL terminator, causing downstream `strlen` calls to read out-of-bounds heap memory.

[[CVE-2025-6334](https://nvd.nist.gov/vuln/detail/CVE-2025-6334)] (2025)
:   Critical vulnerability in D-Link DIR-867 1.0. The `strncpy` function in the Query String Handler causes a stack-based buffer overflow.

[[CVE-2024-54809](https://nist.gov)] (2024)
:   An insecure `strncpy` call in Red Hat's underlying target utilities copies localized environment strings into fixed arrays without adding a trailing NUL byte, resulting in a heap-based memory disclosure flaw.

[[CVE-2024-39480](https://nvd.nist.gov/vuln/detail/cve-2024-39480)] (2024)
:   An insecure `strncpy` call in the Linux Kernel debugger (kdb) tab-completion feature misuses the source buffer size instead of the remaining destination capacity, triggering a predictable buffer overflow during symbol completion.

[[CVE-2024-27224](https://nvd.nist.gov/vuln/detail/cve-2024-27224)] (2024)
:   A vulnerability in `strncpy.c` allows a possible out-of-bounds write due to a missing bounds check, potentially leading to local privilege escalation.

[[CVE-2024-3119](https://nvd.nist.gov/vuln/detail/CVE-2024-3119)] (2024)
:   A buffer overflow exists in sngrep (since v0.4.2) because `strncpy` is used to copy 'Call-ID' SIP headers without proper length checks, allowing remote code execution.

[[CVE-2020-13995](https://nist.gov)] (2020)
:   An insecure `strncpy` call within the extract75 NITF parser component improperly computes destination capacity vs source string length, causing an out-of-bounds array write when handling malicious image metadata headers.

[[CVE-2019-11365](https://nvd.nist.gov/vuln/detail/CVE-2019-11365)] (2019)
:   An insecure `strncpy` call in atftpd (atftp 0.7.1) allows a remote attacker to trigger a stack-based buffer overflow with a specially crafted 3-byte packet.

[[CVE-2019-9125](https://nvd.nist.gov/vuln/detail/CVE-2019-9125)] (2019)
:   An insecure `strncpy` call in D-Link DIR-878 routers results in an unauthenticated, stack-based buffer overflow vulnerability that can be remotely triggered via the HNAP_AUTH HTTP header.

[[CVE-2018-3878](https://nvd.nist.gov/vuln/detail/CVE-2018-3878)] (2018)
:   An insecure `strncpy` call in the credentials handler of the video-core HTTP server on Samsung SmartThings Hub devices overflows a fixed 16-byte stack buffer when an attacker passes an oversized "region" string.

[[CVE-2018-3874](https://nvd.nist.gov/vuln/detail/cve-2018-3874)] (2018)
:   An exploitable buffer overflow in Samsung SmartThings Hub firmware (0.20.17) caused by improper `strncpy` usage.

[[CVE-2012-2114](https://nist.gov)] (2012)
:   An insecure `strncpy` call in the LibTIFF library command line utilities does not ensure null-termination on large input TIFF directories, causing a segmentation fault or memory leak via crafted images.

[[CVE-2004-0500](https://nvd.nist.gov/vuln/detail/CVE-2004-0500)] (2004)
:   An insecure `strncpy` call in Pidgin (Gaim) MSN protocol plugins fails to verify array limits before copying data, allowing remote attackers to trigger a buffer overflow via peer-to-peer messages.

[[CVE-2003-0465](https://nvd.nist.gov/vuln/detail/cve-2003-0465)] (2003)
:   The kernel `strncpy` in Linux 2.4/2.5 did not null-pad the buffer on non-x86 architectures, leading to information leaks.


## Linux Kernel

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

***It is time that WG14 did the same.***








# Design Decisions

# Acknowledgements
Nevin Liber was supported by the Office of Science, U.S. Department of Energy, under contract DE-AC02-06CH11357.

# References
- [N5050](https://wg21.link/n5050) Working Draft, Programming Languages -- C++
- [N3406](https://wg21.link/n3406) A proposal to add a utility class to represent optional objects (Revision 2)
- [P0653](https://wg21.link/p0653) Utility to convert a pointer to a raw pointer
- [P2988](https://wg21.link/p2988) `std::optional<T&>`{.cpp}
- [P4189](https://wg21.link/p4189) `get()`{.cpp}ing the pointer from `optional`{.cpp}
- [P4261](https://wg21.link/p4261) Presentation for P4189 `get()`{.cpp}ing the pointer from `optional`{.cpp}
