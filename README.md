# SPEX: SQLite Packaged Executables.

Complexity is a barrier to intention.

In late 2026, Faride Zakaria introduced the **SELF** file, the 
*Structured Executable & Linkable Format*: an SQLite based alternative 
to standard ELF files, including the binfmt_misc machinery required to
run those SELF files on Linux and NixOS. 

SQLite Packaged Executables - SPEX files - take that idea a step
further, integrating information from package managers, documentation
and other executable-adjacent resources to create a unified software
package that is trivial to deploy, explore and audit. 

As well - by taking advantage of the capabilities of SQLite - this
project has two other goals. First, approach lets us replace a set
brittle, high-complexity tools with much simpler and more easily
legible programs and processes. Second - and more importantly -
this approach will let creative developers easily include other
useful tools or expressive works with the software they ship.

# The longer story.

In a small part of a much longer set of discussions about coding
style and techniques, John Carmack of ID fame (among many other
things) has suggested that "... [If the work is close to purely
functional](https://cbarrete.com/carmack.html), with few references
to global state, try to make it completely functional."

I was thinking of this, and remembering [Greenspun's Tenth
Rule](https://en.wikipedia.org/wiki/Greenspun's_tenth_rule), that
"any sufficiently complicated C or Fortran program contains an ad
hoc, informally-specified, bug-ridden, slow implementation of half
of Common Lisp" - when Zakaria introduced his work.

And the two questions that I think emerge from that are, first, in
terms of the platonic ideal shapes and structures of computer science
the state machines, functional approaches, databases, and so on:
should Carmack's idea that if you're almost there consider going
all the way be elevated to a general principle? 

If something is most of the way to [being a thing] then should we
try to get it all the way to [being that thing] so that we can
leverage of all the research theory and benefits of [being that
thing]?

And second: what if every sufficently complex data format is
effectively an informally specified, bug-ridden, slow implementation
of half of a database?

If that's the case, if an executable is already most of the way to
being a database, what does getting it all the way to being a
database get us? I think we can start answering that question by
asking, what else do we have lying around next to our executables,
that are also almost a database?

I believe the answer to both of those question is: a lot.

If your data format is both an executable and a database, installers
barely exist; they're reduced to a download and a symlink. Strip(),
like many of the tools we use to interrogate binaries, stop being
fiddly multi-thousand-line exercises in bitbanging and turn into
declarative, structured queries that are just a few easily understood
lines long.  We can flatten the MxN arch/distro problem with per-arch
binaries and per-distro packaging manifests built in, after
compilation, the build step becomes a trivial transaction.

And you can put anything else you want in there with it. Sample
code, a whole VCS can come built-in. Community contributions,
alternative skins or themes... I mean, why not ship your art-project
software with the art - the fanfiction, music and art all baked
into a single container that happes to be the thing you run?

And I don't think that this comes at the cost of anything, or at
least anything that matters. Dr. Hipp has famously said that SQLite3
isn't competing with MySQL or Postgres; it's competing with fopen().

So hear me out: if an ELF is almost a database, what can we do, 
what do we get, if we make it all-the-way a database?

If SQLite is competing with fopen(), what if we just... let SQLite win?

# Background 

Zakaria's earlier work in this space -
[sqlelf](https://github.com/fzakaria/sqlelf)
([arXiv:2405.03883](https://arxiv.org/abs/2405.03883)) - introduced
the idea of interrogating the structure of executables via SQL.
SPEX branches off from his subsequent work, turning that idea on
its head; the rows of the database becoming the executable format,
by providing a  a `binfmt_misc` interpreter to execute them on top
of a nixpkgs/NixOS slice.

This project has changed foundations from the nix/guix environment
to Debian Trixie.

```console
$ file hello
hello: SQLite 3.x database, application id 0x53454c46, user version 1
$ ./hello
Hello, world!
$ sqlite3 hello 'SELECT soname FROM ldd'          # ldd, as a query
$ sqlite3 hello 'DELETE FROM sections; VACUUM'    # strip, as a transaction
$ ./hello                                         # still runs
Hello, world!
```

# Developing.

This section assumes you're starting from a generally standard
installation of Debian Trixie, like the author. "Works On My Machine",
isn't great, but it's where this project is right now.

## Setting up Debian-based systems 

On Debian you'll need to install these packages via apt to build
and install the binary-format loader part of the project.

- gcc,
- sqlite3-dev,
- sqlite3-0 and
- pkg-config packages

To work with the Python part of the project, you'll first need to install

- curl, to install the uv package manager
- pyinstaller to package the tools up into standalone executables,
- pandoc and python3-pandocfilters for the documentation consolidation, and
- sqlite3-tools for exploring and validating the results.


```console

$ sudo apt install gcc pkg-config sqlite3 libsqlite3-dev libsqlite3-0 \
       curl pyinstaller sqlite3-tools pandoc python3-pandocfilters
$ curl -LsSf https://astral.sh/uv/install.sh | sh

```

Note that the Astral installer does not run as root, and is only
available to the installing user in ~/.local/bin/. You should 
restart your terminal session after installing UV to be sure that
your environment includes ~/.local/bin in your $PATH.

## Setting up your Python working environment

```console
# After cloning this respository and cd'ing into its working directory:
$ uv venv                            # once, to create virtual environment
$ source .venv/bin/activate          # to activate the environment each time you start work
$ uv sync                            # to install packages (only needed once)
$ python -m selfconv -h              # each time you want to run the script
```







## What's here

- `selfconv/` — the `self` CLI (subcommands include `elf2self`) and `self2elf` (Python + LIEF).
- `loader/` — `self-exec`, the binfmt interpreter, with three modes:
  `memfd` (rebuild ELF → `execveat`), `native` (map segments + hand off to
  ld.so), `selfld` (be the dynamic linker, bind via SQL). Plus
  `libself-audit.so`, an `LD_AUDIT` library that makes **stock glibc** load
  `.self` shared libraries resolved through SQL.
- `nix/` — packages, a `selfify` hook, a NixOS module (`programs.self`), and
  `self-vm`.
- `schema/self.sql` — the format DDL (generated from `selfconv/schema.py`).
- `examples/server/` — **self-httpd**: a webserver whose pages, program and
  visitor log are one file. It opens `argv[0]` as a database and serves out of
  its own `routes` table; editing the live site is an `UPDATE`. Live at
  <https://selfdb.exe.xyz>.
- `bench/`, `tests/` — the evaluation harness and the test suite.

Read [DESIGN.md](./DESIGN.md); §13 tracks implementation status.

## Try it

```console
$ nix develop                        # dev shell (converter + loader + tools)
$ nix develop -c bash tests/all.sh   # run M0..M3b end to end
$ nix develop -c bash tests/showcase.sh
$ nix run .#self-vm                  # a NixOS VM where ./hello.self just runs
                                     # (login: root, empty password)

$ bash examples/server/build.sh ./server   # a website, inserted into a program
$ loader/self-exec ./server 8080           # http://localhost:8080
```
