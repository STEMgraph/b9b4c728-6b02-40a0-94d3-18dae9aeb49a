<!---
{
  "id": "b9b4c728-6b02-40a0-94d3-18dae9aeb49a",
  "teaches": "GNU Make: Building and Installing a Library Into ~/.local",
  "depends_on": [
    "0b44c096-b285-4c06-b5ca-7480f72ca524"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-01",
  "keywords": [
    "make",
    "install",
    "uninstall",
    "PREFIX",
    "DESTDIR",
    "static library",
    "order-only prerequisite",
    "mkdir -p"
  ]
}
--->

# GNU Make: Building and Installing a Library Into `~/.local`

> In this exercise you will write a `Makefile` for a small C library that builds the archive, installs header and archive into a configurable prefix (default `~/.local`), can uninstall them again, and is used by a separate program's `Makefile`. Everything stays in your home directory, no `sudo` is required.

## Introduction

The lecture's `Makefile` installs into `/usr/local`, which needs root. Real projects solve this with a variable, conventionally called `PREFIX`, that defaults to `/usr/local` but can be overridden: `make install PREFIX=$HOME/.local`. Package builders add a second variable, `DESTDIR`, that is prepended to every installed path so that a package can be assembled in a temporary directory.

An `install` target is phony, but it should still only copy what has been built, so it depends on the build products. Directories that must exist before copying are a classic `make` problem: their timestamp changes whenever a file inside them changes, so listing them as normal dependencies causes needless rebuilds. The answer is an **order-only prerequisite**, written after a `|`.

### 1.1) Further Readings and Other Sources

* [GNU Make Manual: Standard Targets for Users](https://www.gnu.org/software/make/manual/html_node/Standard-Targets.html)
* [GNU Make Manual: Types of Prerequisites (order-only)](https://www.gnu.org/software/make/manual/html_node/Prerequisite-Types.html)
* [GNU Coding Standards: DESTDIR](https://www.gnu.org/prep/standards/html_node/DESTDIR.html)
* [install(1) man page](https://man7.org/linux/man-pages/man1/install.1.html)

## Tasks

### Task 1: Separate Library and Program

Split the project into two directories:

```bash
mkdir -p ~/vec2lib ~/vec2app
cp ~/make-intro/vec2.c ~/make-intro/vec2.h ~/vec2lib/
cp ~/make-intro/main.c ~/vec2app/
```

### Task 2: The Library `Makefile`

```bash
cd ~/vec2lib && vim Makefile
```

```make
CC      = gcc
CFLAGS  = -Wall -Wextra -O2
PREFIX ?= $(HOME)/.local
DESTDIR ?=

LIB  = libvec2.a
OBJS = vec2.o

all: $(LIB)

$(LIB): $(OBJS)
	ar rcs $@ $^

%.o: %.c vec2.h
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(LIB)

.PHONY: all clean
```

Build it with `make`. Note `?=`: the variable is only set if it is not already set, for example from the environment.

### Task 3: An `install` Target With Order-Only Directories

Append:

```make
INCDIR = $(DESTDIR)$(PREFIX)/include
LIBDIR = $(DESTDIR)$(PREFIX)/lib

install: $(LIB) vec2.h | $(INCDIR) $(LIBDIR)
	install -m 644 vec2.h $(INCDIR)/
	install -m 644 $(LIB) $(LIBDIR)/

$(INCDIR) $(LIBDIR):
	mkdir -p $@

.PHONY: install
```

```bash
make install
ls -l ~/.local/include/vec2.h ~/.local/lib/libvec2.a
make install
```

The second `make install` copies again (it is phony), but does not try to create the directories again. Replace the `|` with a space, run `touch ~/.local/lib/x; make -n install` and observe the difference.

### Task 4: `uninstall`

```make
uninstall:
	rm -f $(INCDIR)/vec2.h $(LIBDIR)/$(LIB)

.PHONY: uninstall
```

```bash
make uninstall
ls ~/.local/lib
make install
```

### Task 5: Use the Installed Library From Another Project

```bash
cd ~/vec2app && vim Makefile
```

```make
CC      = gcc
CFLAGS  = -Wall -Wextra -O2
PREFIX ?= $(HOME)/.local

main_program: main.o
	$(CC) $< -o $@ -L$(PREFIX)/lib -lvec2 -lm

main.o: main.c
	$(CC) $(CFLAGS) -I$(PREFIX)/include -c $< -o $@

clean:
	rm -f main.o main_program

.PHONY: clean
```

Change `#include "vec2.h"` in `main.c` to `#include <vec2.h>`, then:

```bash
make
./main_program
```

The application never sees `vec2.c`. It only knows the installed header and archive.

### Task 6: A Different Prefix, and `DESTDIR`

```bash
cd ~/vec2lib
make install PREFIX=$HOME/opt/vec2
ls -R ~/opt/vec2
cd ~/vec2app && make clean && make PREFIX=$HOME/opt/vec2 && ./main_program
```

Now simulate what a package builder does:

```bash
cd ~/vec2lib
make install DESTDIR=$HOME/pkgroot PREFIX=/usr/local
find ~/pkgroot
```

Nothing touched `/usr/local`. The whole tree that *would* be installed there now sits under `~/pkgroot`, ready to be packed into a `.deb` or a tarball.

### Task 7: Change the Library, Rebuild the App

In `~/vec2lib`, change `VEC2_len` to print a message, then:

```bash
make install
cd ~/vec2app && make && ./main_program
```

The application was not rebuilt. Why not? Which file would `main_program` have to depend on so that `make` notices the new archive? Add that dependency and try again.

## Questions

- What is the difference between `PREFIX = ...`, `PREFIX ?= ...` and `PREFIX := ...` when the user runs `PREFIX=/tmp/x make install`?
- Why is `install` phony, but still has dependencies?
- What goes wrong if `$(LIBDIR)` is a normal prerequisite of `install` instead of an order-only one? When would you notice?
- `install -m 644` versus `cp`: what does `install` do that `cp` does not?
- Why does `DESTDIR` exist in addition to `PREFIX`? Why is it wrong to bake `DESTDIR` into the path that the program uses at run time?
- In Task 7, the application `Makefile` has no way to know that `~/.local/lib/libvec2.a` changed unless told. How do larger build systems like CMake solve this?

## Advice

With `PREFIX ?= $(HOME)/.local` you can install and test software without ever typing `sudo`, and the same `Makefile` works for a system administrator who runs `make install PREFIX=/usr/local`. This is the convention almost every open-source C project follows, so once you can read it in your own file, you can read it everywhere.

The order-only prerequisite is a small thing, but the `mkdir -p` rule is the canonical example of it. If you ever see a target rebuilt for no reason, check whether a directory is listed as a normal dependency.

The last task points at the limit of hand-written `Makefile`s: dependencies on *installed* files of other projects. That is where `pkg-config` and CMake come in, which the next chapter covers.
