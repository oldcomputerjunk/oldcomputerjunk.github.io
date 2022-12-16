---
layout: post
slug: valgrind-cross
title:  "Notes on coaxing valgrind to work on arbitrary embedded targets"
date:   2022-12-16 18:00:00
categories:
- infosec
- howto
tags:
- embedded
- hacking
- reversing
---

# Introduction

There are many tools in the kit for exploring embedded systems when reverse engineering. On Linux for example, you might use `strace` to watch system calls made by a program. `valgrind` is another useful tool (https://valgrind.org), yet one I’ve seen less commonly referenced in regard to reverse engineering. `valgrind` would be familiar to many C/C++ developers, where it is commonly used for testing for memory leaks and cache performance; indeed it has helped me track down subtle bugs in situations ranging from third party library integration, to network programming, to high performance GPU programming in the past. Like `strace`, you run `valgrind` to run another program that it controls - `valgrind` instruments the heap and can produce stack traces identifying the location of memory errors, and will track and pinpoint the source of memory leaks.

During reverse engineering, you are typically unlikely to find `valgrind` on a reverse engineering target. As it is you might not find `strace`, depending on how tight the memory constraints are.  On a Raspberry Pi of course, you can just apt-get install it. There is also a package for OpenWRT. Otherwise, you are faced with cross compiling it; and then, getting it to actually work as intended, even if you get it to run on the target.

The rest of this article focusses on arm embedded Linux but the principles should apply equally to MIPS or x86 with relevant changes.

# A birds eye view

To make valgrind run on a non-typical target, you will need to:
- install a cross compiler that can at a minimum produce static binaries that run on the target; valgrind by its very nature is not a static binary, but if we can’t pass this hurdle then the feasibility of the rest of the process drops off sharply
- get the source code or clone the valgrind git repository
  - depending on the age of your target, you might need to wind back a few versions, to work around otherwise insurmountable incompatibility changes as development has progressed
- get the compiled valgrind software, which is not just a binary but a bunch of supporting data files, onto the target
- deal with library versioning errors
    - this may require setting up a virtual machine with an older version of Linux and the cross compiler toolchain to compile against, or even finding a toolchain from another source, such as https://developer.arm.com/Tools%20and%20Software/GNU%20Toolchain
- deal with `valgrind` refusing to work due to some apparently mandatory rules baked into the software
- deal with other errors related to missing files or version mismatches

In practice, some of the above can be anticipated and avoided by using an older valgrind version and older build environment from the start!

# Before doing this

If you are very lucky, the target might match an environment you already have running. For example my Raspberry Pi 400 runs raspbian, and is the arm7 armhf architecture; if your target is the same, you might be able to `apt-get install busybox-static` and then copy that to your target and if it runs, you may be able to avoid cross-compiling at all and simply copy from your raspberry pi. But see the section about needing to find all the valgrind supporting files...

# Typical cross compiling process - environment

Assumptions:
- we are targeting an embedded device with firmware dating back to 2017 running Linux 4.4
- with this in mind, the following cross compile was done using a Ubuntu 16.04 environment, to avoid GLIBC_2.2x version errors
- these commands were tested using Ubuntu 1604 running in WSL, but whether you do that or native or in a VM shouldn’t matter
- what you need to install could well differ if you are using a newer distro or flavour (and of course the package names are likely different in CentOS or others, too)
- I haven't shown commands for transferring files to the target as this will depend on your situation
- if you install your toolchain from another source or on another distro, the gcc command used could be different to what is shown, and you may need to update your $PATH

Process:
- install cross compiler tools
    ```
    sudo apt-get install gcc-arm-linux-gnueabihf ca-certificates autoconf libc6-dbg-armhf-cross
    ```
    Note that `libc6-dbg-armhf-cross` may be needed for the target
- compile a static and dynamic test binary
    ```
    echo -e ‘#include <stdio.h>\nint main(int argc, char* argv[]) { printf(“Hello world”); }’ > hello.c
    arm-linux-gnueabihf-gcc hello.c -o hello
    arm-linux-gnueabihf-gcc hello.c -static -o hello-s
    ```
- copy the resulting files `hello` and `hello-s` to the target and run them. Remember to make them executable!

If the static binary `hello-s` runs, but the dynamically linked binary `hello` does not, then the C library files in your toolchain are (probably) too new for the target, or have some other version mismatch, or the `ld-linux.so*` in the target is mismatched. Try again with an older or a different toolchain. It might be that the toolchain is for a different architecture or C library, e.g. aarch64, or musl instead of libc, etc. Understanding and dealing with this is, which can include patching the linker processing is beyond the scope of this article; but basically, when you run `file` on a binary from the target, the ELF information should be as close as possible to what you compiled. By way of example, the following two binaries are not compatible in each others intended environment; the former is from my pi400, the latter from an OpenWRT cross compilation - the linker interpereter is obviously different:
```
/usr/bin/bash: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux 3.2.0, BuildID[sha1]=3e5e2847bbc51da2ab313bc53d4bdcff0faf2462, stripped
``` 
vs
```
hello: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-musl-arm.so.1, with debug_info, not stripped
```

A typical error to watch for might look like:
```
./hello: /lib/libc.so.6: version `GLIBC_2.5' not found (required by ./hello)
```


# Typical cross compiling process - valgrind

Having established your toolchain can build code that runs on the target, obtain the source code then change into the valgrind  source working copy and cross compile from a clean build directory.

You could download and unzip a specified release version of valgrind from the project home page, or clone the projects git repository (neither operation is shown here)

```
cd path/to/source-of/valgrind
mkdir build
cd build
export CROSS_COMPILE=arm-linux-gnueabihf-
../configure --target=arm-linux-gnueabihf \
    --host=armv7-linux-gnueabihf \
    --prefix=$(pwd)/out \
    CC=${CROSS_COMPILE}gcc \
    CPP=${CROSS_COMPILE}cpp \
    CXX=${CROSS_COMPILE}g++ \
    LD=${CROSS_COMPILE}ld \
    AR=${CROSS_COMPILE}ar \
    --enable-only32bit \
    --enable-tls \
    --without-x \
    --without-mpicc \
    --without-uiout \
    --disable-valgrindmi \
    --disable-tui \
    --disable-valgrindtk \
    --without-included-gettext \
    --with-pagesize=4
make -j5 # 5 is optional, this was 1+threads on my computer
make install
```

Note that the purpose of the `--prefix` argument is to allow `make install` to copy the result into the `build/out` directory, allowing us to pick out the files to transfer to the target

Bundle them up as follows. Then, transfer to the target and unpack.

```
tar czf out.tar.gz out/bin/valgrind* out/libexec/valgrind/{arm-*.xml,32bit-core-*.xml,32bit-linux-*.xml,memcheck-*,none-*,vgpreload_core*,vgpreload_memcheck*}
```

(Here, I worked out the minimum needed set of files through trial and error. YMMV!)

# Testing valgrind on the target

Test you can run the `valgrind` binary on the target.
To do this requires setting an environment variable so that `valgrind` can find its data files, and optionally adding to the PATH

```
cd path/to/unpacked-folder/out
export VALGRIND_LIB=$(pwd)/libexec/valgrind # newer versions
export VALGRIND_LIB=$(pwd)/lib/valgrind # valgrind 3.13 or older
cd bin
export PATH=$PATH:$(pwd)
valgrind
```

Now the acid test: run another program using valgrind!

```
valgrind /bin/busybox
```

If this works, with no segfaults, and your program runs and exits, and you have the valgrind memory leak summary, all is well and you are done!

Example - note this was run on my pi400 so the version reported is different, I forgot to log the result from the target and was not nearby to try again when finishing this blog:
```
==30567== Memcheck, a memory error detector
==30567== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.
==30567== Using Valgrind-3.20.0 and LibVEX; rerun with -h for copyright info
==30567== Command: ./hello
==30567==
Hello world!
==30567==
==30567== HEAP SUMMARY:
==30567==     in use at exit: 0 bytes in 0 blocks
==30567==   total heap usage: 0 allocs, 0 frees, 0 bytes allocated
==30567==
==30567== All heap blocks were freed -- no leaks are possible
==30567==
==30567== For lists of detected and suppressed errors, rerun with: -s
==30567== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

# Typical problems and how to fix them

Here is where the “fun” starts.

You might get an error like the following:

```
valgrind:  Fatal error at startup: a function redirection
valgrind:  which is mandatory for this platform-tool combination
valgrind:  cannot be set up.  Details of the redirection are:
valgrind:  
valgrind:  A must-be-redirected function
valgrind:  whose name matches the pattern:      index
valgrind:  in an object with soname matching:   ld-linux-armhf.so.3
valgrind:  was not found whilst processing
valgrind:  symbols from the object with soname: ld-linux-armhf.so.3
valgrind:  
valgrind:  Possible fixes: (1, short term): install glibc's debuginfo
valgrind:  package on this machine.  (2, longer term): ask the packagers
valgrind:  for your Linux distribution to please in future ship a non-
valgrind:  stripped ld.so (or whatever the dynamic linker .so is called)
valgrind:  that exports the above-named function using the standard
valgrind:  calling conventions for this platform.  The package you need
valgrind:  to install for fix (1) is called
valgrind:  
valgrind:    On Debian, Ubuntu:                 libc6-dbg
valgrind:    On SuSE, openSuSE, Fedora, RHEL:   glibc-debuginfo
valgrind:  
valgrind:  Note that if you are debugging a 32 bit process on a
valgrind:  64 bit system, you will need a corresponding 32 bit debuginfo
valgrind:  package (e.g. libc6-dbg:i386).
valgrind:  
valgrind:  Cannot continue -- exiting now.  Sorry.
```

I have only seen this error trying to run valgrind against non-static binaries. To cut a long story short, there are `valgrind` command line args that let you turn on and off a lot of functionality, but they don't stop this one! It seems the developers really, badly, want to force `valgrind` to find several functions in the C library shared library on the target. (I speculate that where I encountered this error, there was a weird `LD_PRELOAD` hard coded into the readonly initramfs I hadn’t at that stage worked out how to bypass, which had the effect of masking what valgrind was looking for.)

Lots of googling later and I found that definitely, yes, other people had seen this error, (and I don’t know why this always seem to happen to me but) the responses were either along the lines of “WTF are you doing that for" or “we cant fix this”. This kind of response is so annoying, especially when it turns out there is a solution. I think part of the problem is that google is getting less and less useful at finding what you want and better and better at turning up junk SEO blogs. And the other just as important problem is where so many people say things without understanding or care. Anyway, in the end I had to go prospecting into the `valgrind` source code, and to cut a long story short, it turns out this problem can be fixed by commenting out these “mandatory” rules and recompiling, and in spite of the big scary warning, it turns out everything just works…

<div style="padding: 15px; border: 1px solid transparent; border-color: transparent; margin-bottom: 20px; border-radius: 4px; color: #3c763d; background-color: #dff0d8; border-color: #d6e9c6;">
Solution: comment out at least lines 1456-1498 of <b>coregrind/m_redir.c</b>, (numbers will vary depending on version you chose, and this is for the arm processor!) and for good measure I used version 3.13 or older (just a guess).

<br/>
See: https://sourceware.org/git/?p=valgrind.git;a=blob;f=coregrind/m_redir.c;h=d54cae7966397485b8b5977724bec86c18847658;hb=0dc5853b9ee33f9a02c1fb8894e91b89a379ee97
</div>

In particular, the lines to be commened look like the following:
```
1461       add_hardwired_spec(
1462          "ld-linux-armhf.so.3", "strlen",
1463          (Addr)&VG_(arm_linux_REDIR_FOR_strlen),
1464          complain_about_stripped_glibc_ldso
1465       );
```
Remove all of them for the arch you are targeting.

Example of cross-compiling a specific older version. Dont forget to patch `m_redir.c` first:
```
git clone https://sourceware.org/git/valgrind.git
cd valgrind
git checkout svn/VALGRIND_3_13_0
git clean -xfd
git checkout svn/VALGRIND_3_13_0
./autogen.sh
mkdir build
cd build
export CROSS_COMPILE=arm-linux-gnueabihf-
../configure --target=arm-linux-gnueabihf \
    --host=armv7-linux-gnueabihf \
    --prefix=$(pwd)/out \
    CC=${CROSS_COMPILE}gcc \
    CPP=${CROSS_COMPILE}cpp \
    CXX=${CROSS_COMPILE}g++ \
    LD=${CROSS_COMPILE}ld \
    AR=${CROSS_COMPILE}ar \
    --enable-only32bit \
    --enable-tls \
    --without-x \
    --without-mpicc \
    --without-uiout \
    --disable-valgrindmi \
    --disable-tui \
    --disable-valgrindtk \
    --without-included-gettext \
    --with-pagesize=4
make -j5
make install
```

Bundle up the files. Newer valgrinds:
```
tar czf out.tar.gz out/bin/valgrind* out/libexec/valgrind/{arm-*.xml,32bit-core-*.xml,32bit-linux-*.xml,memcheck-*,none-*,vgpreload_core*,vgpreload_memcheck*,*.supp}
```
Note, 3.13 or older has a different directory for supporting files:
```
tar czf out.tar.gz out/bin/valgrind* out/lib/valgrind/{arm-*.xml,32bit-core-*.xml,32bit-linux-*.xml,memcheck-*,none-*,vgpreload_core*,vgpreload_memcheck*,*.supp}
```

For good measure, here is a full set of commands on a target where I got `valgrind` to run on this target finally - YMMV depending on your setup, and the `lib` directory can change between versions:

```
cd /tmp
mkdir -p vg
busybox zcat path/to/vg.tar.gz | tar xf -
mv out vg
cd vg/out
export VALGRIND_LIB=/tmp/vg/out/libexec/valgrind
export VALGRIND_LIB=/tmp/vg/out/lib/valgrind  # 3.13
cd bin
./valgrind busybox
```

# More potential problems

Valgrind also needs a DEBUG version of the C shared library.
This may lead to mismatch errors or likely wont be present, in case you need to bring your own from the build toolchain. You can run valgrind with the following options to maybe work around this: `--extra-debuginfo-path=path/to/byo/dbgfolder  --allow-mismatched-debuginfo=yes` - these files come from the `libc6-dbg-armhf-cross` package above, copy the file `/usr/arm-linux-gnueabihf/lib/debug/lib/arm-linux-gnueabihf/ld-2.23.so` (or similar, depending on your toolchain) into the location on the target

If you get an error like `FATAL: can't open suppressions file "/data/vg/out/libexec/valgrind/default.supp"` then some supporting files are missing (see the tar command above)


You may get a lot of illegal instruction noise or fatal errors:
```
==28657== Your program just tried to execute an instruction that Valgrind
==28657== did not recognise.  There are two possible reasons for this.
==28657== 1. Your program has a bug and erroneously jumped to a non-code
==28657==    location.  If you are running Memcheck and you just saw a
==28657==    warning about a bad jump, it's probably your program's fault.
==28657== 2. The instruction is legitimate but Valgrind doesn't handle it,
==28657==    i.e. it's Valgrind's fault.  If you think this is the case or
==28657==    you are not sure, please let us know and we'll try to fix it.
==28657== Either way, Valgrind will now raise a SIGILL signal which will
==28657== probably kill your program.
```
these can be ignored using `--sigill-diagnostics=no` but too-old versions may not have this. Including as it turns out 3.7.0 on buster, so you really will probably need to focus on a narrow range of releases (I had most luck with 3.13)



Not specific tio 'making it work' but you can also trace into children processes using `--trace-children=yes`

# Postfix

In case it was not clear, this is not a complete tutorial, it assumes that the reader has quite a bit of understanding of working with embedded linux development; rather, it attempts to consolidate in one place various tactics and solutions that I either failed to find when I needed, or had to work out through experiment. The reader will need to be able to read between the lines to suit their own environment!
