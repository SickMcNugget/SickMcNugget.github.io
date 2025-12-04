---
layout: post
title: "A bit about debuginfod"
date: 2025-12-04 16:30:05 +08:00
categories: linux tutorial observability elf binary
published: true
---

debuginfod is super useful when it comes to modern observability tools. Since shipping a binary with symbols and debug information isn't usually done, a solution that's been used for some time is to package that information inside of a separate shared library which can itself be used to fill in the blanks.

## Understanding debuginfod
I'm mainly going to use https://debuginfod.ubuntu.com as my source in this post, since it's my decision and I'm choosing to make it.
When you go to that page, it'll just show you some documentation about what debuginfod is and how to use it, but now how it works more deeply.

First, the [debuginfod man page](https://www.mankier.com/8/debuginfod#Webapi) is going to be of great help, use it. Specifically, the Web API is how we're going to collect this information for now.

## Obtaining our test data
I'm on Ubuntu 24.04, so you can follow along if you like.
First, we need a package for testing. Let's use libX11.
```bash
# Install the package
sudo apt install -y libx11-6
# Get the path for libX11.so.6.4.0
apt-file list libx11-6
```
I know for a fact that Ubuntu have a debuginfo entry for libX11.so.6.4.0, so let's grab that entry, using the API from the manpage listed above, and compare the two files. The `readelf` command will be very helpful to us.

To retrieve data from a debuginfod server, you need to know the build id of the binary file you're interested in. You can obtain it in two ways:
```bash
file /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0
# /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=4cb55b1a3e1fcb63bde78cbab338d576fc43e330, stripped
readelf -n /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0
# ...
#    Build ID: 4cb55b1a3e1fcb63bde78cbab338d576fc43e330
```
I prefer `readelf`, because we'll be using it more later, anyway.

I have a bash function in my `.bashrc` to grab the build ID from readelf:
```bash
function elf-buildid() {
        if [ $# -ne 1 ]; then
                printf "\x1b[33m%s\x1b[0m\n" "Please run with a file name" >&2
                return 1
        fi

        if [ ! -f "$1" ]; then
                printf "\x1b[31m%s\x1b[0m\n" "File: ${1} does not exist" >&2
                return 1
        fi

        readelf -n "$1" | sed -n 's/Build ID: *\(.*\)/\1/p' | tr -d '[:space:]'
}
```

Let's download the debuginfo for libX11.so.6.4.0:
```bash
buildid=$(elf-buildid /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0)
curl -fsSO "https://debuginfod.ubuntu.com/buildid/${buildid}/debuginfo"
```
Note that we use the `/buildid/BUILDID/debuginfo` path to get our file, which is called `debuginfo`, unless named otherwise.

Another way to grab this information is using the `debuginfod-find` tool, which comes with `debuginfod` if you install it locally. For example:
```bash
buildid=$(elf-buildid /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0)
DEBUGINFOD_URLS="https://debuginfod.ubuntu.com" debuginfod-find debuginfo ${buildid}
# OR (don't need build ID!)
DEBUGINFOD_URLS="https://debuginfod.ubuntu.com" debuginfod-find debuginfo /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0
```
Note that the path to the downloaded file is printed to stdout. The hexadecimal folder name corresponds to the build ID of `libX11.so.6.4.0`.

A quick diff between the `curl`ed file and the `debuginfod-find`ed (found?) file gives:
```bash
diff debuginfo /home/joren/.cache/debuginfod_client/4cb55b1a3e1fcb63bde78cbab338d576fc43e330/debuginfo
# 
```
Absolutely nothing, since they're the same file.

If we run:
```bash
readelf -w /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0
# Contents of the .gnu_debuglink section:
#
#  Separate debug info file: b55b1a3e1fcb63bde78cbab338d576fc43e330.debug
#  CRC value: 0xea73cd55
```
You can see that libX11.so.6.4.0 specifies a file name that it expects to receive debug symbols from. I suspect we can move our `debuginfo` file to `/usr/lib/debug/` in some fashion to make this work. We'll discuss this more at the end of this document. Also note that the hash of that separate file is just the original hash with `4c` removed from the front. Curious...

Time to analyse these files.

## What's the deal with debuginfo?
First, let's compare what `file` tells us:
```bash
file debuginfo
# debuginfo: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=4cb55b1a3e1fcb63bde78cbab338d576fc43e330, with debug_info, not stripped
file /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0
# /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=4cb55b1a3e1fcb63bde78cbab338d576fc43e330, stripped
```
We can see that everything is the same, except our debuginfo has `with debug_info, not stripped` and our original library has `stripped`. Interesting.

Let's compare how large the files are:
```bash
du debuginfo
# 1360K
du /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0
# 1268K
```
So... our debug file is larger than the original library it's based on. Keep in mind, that the debug file **only** holds debug information and symbols, and cannot *do* what libX11.so.6.4.0 does. Really shows you why so many developers compile without debug info and strip their binaries, it just adds excess file size, especially in production environments.

Let's diff the section headers for both of the files:
```bash
diff -y <(readelf debuginfo -S) <(readelf /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0 -S)
# There are 39 section headers, starting at offset 0x1526b8:    | There are 29 section headers, starting at offset 0x13c768:
# 
# Section Headers:                                                Section Headers:
#   [Nr] Name              Type             Address           O     [Nr] Name              Type             Address           O
#        Size              EntSize          Flags  Link  Info            Size              EntSize          Flags  Link  Info
#   [ 0]                   NULL             0000000000000000  0     [ 0]                   NULL             0000000000000000  0
#        0000000000000000  0000000000000000           0     0            0000000000000000  0000000000000000           0     0
#   [ 1] .note.gnu.pr[...] NOTE             00000000000002a8  0     [ 1] .note.gnu.pr[...] NOTE             00000000000002a8  0
#        0000000000000020  0000000000000000   A       0     0            0000000000000020  0000000000000000   A       0     0
#   [ 2] .note.gnu.bu[...] NOTE             00000000000002c8  0     [ 2] .note.gnu.bu[...] NOTE             00000000000002c8  0
#        0000000000000024  0000000000000000   A       0     0            0000000000000024  0000000000000000   A       0     0
#   [ 3] .gnu.hash         NOBITS           00000000000002f0  0 |   [ 3] .gnu.hash         GNU_HASH         00000000000002f0  0
#        0000000000002778  0000000000000000   A       4     0            0000000000002778  0000000000000000   A       4     0
#   [ 4] .dynsym           NOBITS           0000000000002a68  0 |   [ 4] .dynsym           DYNSYM           0000000000002a68  0
#        00000000000080e8  0000000000000018   A       5     1            00000000000080e8  0000000000000018   A       5     1
#   [ 5] .dynstr           NOBITS           000000000000ab50  0 |   [ 5] .dynstr           STRTAB           000000000000ab50  0
#        0000000000005a6c  0000000000000000   A       0     0            0000000000005a6c  0000000000000000   A       0     0
#   [ 6] .gnu.version      NOBITS           00000000000105bc  0 |   [ 6] .gnu.version      VERSYM           00000000000105bc  0
#        0000000000000abe  0000000000000002   A       4     0            0000000000000abe  0000000000000002   A       4     0
#   [ 7] .gnu.version_r    NOBITS           0000000000011080  0 |   [ 7] .gnu.version_r    VERNEED          0000000000011080  0
#        00000000000000c0  0000000000000000   A       5     1            00000000000000c0  0000000000000000   A       5     1
#   [ 8] .rela.dyn         NOBITS           0000000000011140  0 |   [ 8] .rela.dyn         RELA             0000000000011140  0
#        00000000000063f0  0000000000000018   A       4     0            00000000000063f0  0000000000000018   A       4     0
#   [ 9] .rela.plt         NOBITS           0000000000017530  0 |   [ 9] .rela.plt         RELA             0000000000017530  0
#        0000000000000c90  0000000000000018   A       4    24   |        0000000000000c90  0000000000000018  AI       4    24
#   [10] .init             NOBITS           0000000000019000  0 |   [10] .init             PROGBITS         0000000000019000  0
#        000000000000001b  0000000000000000  AX       0     0            000000000000001b  0000000000000000  AX       0     0
#   [11] .plt              NOBITS           0000000000019020  0 |   [11] .plt              PROGBITS         0000000000019020  0
#        0000000000000870  0000000000000010  AX       0     0            0000000000000870  0000000000000010  AX       0     0
#   [12] .plt.got          NOBITS           0000000000019890  0 |   [12] .plt.got          PROGBITS         0000000000019890  0
#        0000000000000010  0000000000000010  AX       0     0            0000000000000010  0000000000000010  AX       0     0
#   [13] .plt.sec          NOBITS           00000000000198a0  0 |   [13] .plt.sec          PROGBITS         00000000000198a0  0
#        0000000000000860  0000000000000010  AX       0     0            0000000000000860  0000000000000010  AX       0     0
#   [14] .text             NOBITS           000000000001a100  0 |   [14] .text             PROGBITS         000000000001a100  0
#        000000000008eede  0000000000000000  AX       0     0            000000000008eede  0000000000000000  AX       0     0
#   [15] .fini             NOBITS           00000000000a8fe0  0 |   [15] .fini             PROGBITS         00000000000a8fe0  0
#        000000000000000d  0000000000000000  AX       0     0            000000000000000d  0000000000000000  AX       0     0
#   [16] .rodata           NOBITS           00000000000a9000  0 |   [16] .rodata           PROGBITS         00000000000a9000  0
#        0000000000078c58  0000000000000000   A       0     0            0000000000078c58  0000000000000000   A       0     0
#   [17] .eh_frame_hdr     NOBITS           0000000000121c58  0 |   [17] .eh_frame_hdr     PROGBITS         0000000000121c58  0
#        0000000000003c3c  0000000000000000   A       0     0            0000000000003c3c  0000000000000000   A       0     0
#   [18] .eh_frame         NOBITS           0000000000125898  0 |   [18] .eh_frame         PROGBITS         0000000000125898  0
#        0000000000011928  0000000000000000   A       0     0            0000000000011928  0000000000000000   A       0     0
#   [19] .init_array       NOBITS           0000000000138140  0 |   [19] .init_array       INIT_ARRAY       0000000000138140  0
#        0000000000000010  0000000000000008  WA       0     0            0000000000000010  0000000000000008  WA       0     0
#   [20] .fini_array       NOBITS           0000000000138150  0 |   [20] .fini_array       FINI_ARRAY       0000000000138150  0
#        0000000000000010  0000000000000008  WA       0     0            0000000000000010  0000000000000008  WA       0     0
#   [21] .data.rel.ro      NOBITS           0000000000138160  0 |   [21] .data.rel.ro      PROGBITS         0000000000138160  0
#        0000000000000ad0  0000000000000000  WA       0     0            0000000000000ad0  0000000000000000  WA       0     0
#   [22] .dynamic          NOBITS           0000000000138c30  0 |   [22] .dynamic          DYNAMIC          0000000000138c30  0
#        00000000000001e0  0000000000000010  WA       5     0            00000000000001e0  0000000000000010  WA       5     0
#   [23] .got              NOBITS           0000000000138e10  0 |   [23] .got              PROGBITS         0000000000138e10  0
#        00000000000001c0  0000000000000008  WA       0     0            00000000000001c0  0000000000000008  WA       0     0
#   [24] .got.plt          NOBITS           0000000000138fe8  0 |   [24] .got.plt          PROGBITS         0000000000138fe8  0
#        0000000000000448  0000000000000008  WA       0     0            0000000000000448  0000000000000008  WA       0     0
#   [25] .data             NOBITS           0000000000139440  0 |   [25] .data             PROGBITS         0000000000139440  0
#        00000000000031e0  0000000000000000  WA       0     0            00000000000031e0  0000000000000000  WA       0     0
#   [26] .bss              NOBITS           000000000013c620  0 |   [26] .bss              NOBITS           000000000013c620  0
#        0000000000000708  0000000000000000  WA       0     0            0000000000000708  0000000000000000  WA       0     0
#   [27] .comment          PROGBITS         0000000000000000  0 |   [27] .gnu_debuglink    PROGBITS         0000000000000000  0
#        0000000000000026  0000000000000001  MS       0     0   |        0000000000000034  0000000000000000           0     0
#   [28] .debug_aranges    PROGBITS         0000000000000000  0 |   [28] .shstrtab         STRTAB           0000000000000000  0
#        0000000000000126  0000000000000000   C       0     0   |        0000000000000110  0000000000000000           0     0
#   [29] .debug_info       PROGBITS         0000000000000000  0 <
#        000000000008d1e7  0000000000000000   C       0     0   <
#   [30] .debug_abbrev     PROGBITS         0000000000000000  0 <
#        00000000000034c2  0000000000000000   C       0     0   <
#   [31] .debug_line       PROGBITS         0000000000000000  0 <
#        0000000000032a7d  0000000000000000   C       0     0   <
#   [32] .debug_str        PROGBITS         0000000000000000  0 <
#        0000000000008b8e  0000000000000001 MSC       0     0   <
#   [33] .debug_line_str   PROGBITS         0000000000000000  0 <
#        0000000000000c1c  0000000000000001 MSC       0     0   <
#   [34] .debug_loclists   PROGBITS         0000000000000000  0 <
#        0000000000040657  0000000000000000   C       0     0   <
#   [35] .debug_rnglists   PROGBITS         0000000000000000  0 <
#        0000000000004ed5  0000000000000000   C       0     0   <
#   [36] .symtab           SYMTAB           0000000000000000  0 <
#        00000000000333f0  0000000000000018          37   7371  <
#   [37] .strtab           STRTAB           0000000000000000  0 <
#        000000000000ceea  0000000000000000           0     0   <
#   [38] .shstrtab         STRTAB           0000000000000000  0 <
#        000000000000018a  0000000000000000           0     0   <
# Key to Flags:                                                   Key to Flags:
#   W (write), A (alloc), X (execute), M (merge), S (strings),      W (write), A (alloc), X (execute), M (merge), S (strings),
#   L (link order), O (extra OS processing required), G (group)     L (link order), O (extra OS processing required), G (group)
#   C (compressed), x (unknown), o (OS specific), E (exclude),      C (compressed), x (unknown), o (OS specific), E (exclude),
#   D (mbind), l (large), p (processor specific)                    D (mbind), l (large), p (processor specific)
```
So `debuginfo` has 10 more sections than what our original file has, with `.debug_info`, `.debug_line`, `.debug_loclists` and `.symtab` being the largest sections.
Also, nearly all the sections from the original file have been marked as `NOBITS` in the `debuginfo` file. If you check the contents of these sections, they're just empty; there's no data there at all. I suspect the addresses, size, offset, etc. are required for compatibility between the two files, but the `debuginfo` file strips away all the data since it shouldn't be a normal executable/shared object.

Now let's compare the difference is size of various sections in both files:
```bash
# Let's start by comparing the size of the symbol table for both files
diff <(readelf -s debuginfo | wc -l) <(readelf -s /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0 | wc -l)
# < 8749
# ---
# > 1378

# Let's also compare the size of the debug section for both files
diff <(readelf -w debuginfo | wc -l) <(readelf -w /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0 | wc -l)
# < 1110200
# ---
# > 29312
```
So we have more symbols and way more debug information in our `debuginfo` file, who would've guessed, huh? (me)

A final note with that last diff command, assuming you don't already have debug data available on your system for libX11.so.6.4.0, you should have gotten the following messages from `readelf`:
```bash
# readelf: Warning: could not find separate debug file 'b55b1a3e1fcb63bde78cbab338d576fc43e330.debug'
# readelf: Warning: tried: /lib/debug/b55b1a3e1fcb63bde78cbab338d576fc43e330.debug
# readelf: Warning: tried: /usr/lib/debug/usr/b55b1a3e1fcb63bde78cbab338d576fc43e330.debug
# readelf: Warning: tried: /usr/lib/debug//usr/lib/x86_64-linux-gnu//b55b1a3e1fcb63bde78cbab338d576fc43e330.debug
# readelf: Warning: tried: /usr/lib/debug/b55b1a3e1fcb63bde78cbab338d576fc43e330.debug
# readelf: Warning: tried: /usr/lib/x86_64-linux-gnu/.debug/b55b1a3e1fcb63bde78cbab338d576fc43e330.debug
# readelf: Warning: tried: /usr/lib/x86_64-linux-gnu/b55b1a3e1fcb63bde78cbab338d576fc43e330.debug
# readelf: Warning: tried: .debug/b55b1a3e1fcb63bde78cbab338d576fc43e330.debug
# readelf: Warning: tried: b55b1a3e1fcb63bde78cbab338d576fc43e330.debug
```

What happens if we move our debuginfo file to one of these locations? is that going to work? Only one way to find out
```bash
mv debuginfo b55b1a3e1fcb63bde78cbab338d576fc43e330.debug
diff <(readelf -w b55b1a3e1fcb63bde78cbab338d576fc43e330.debug | wc -l) <(readelf -w /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0 | wc -l)
# < 1110200
# ---
# > 1139514
```
