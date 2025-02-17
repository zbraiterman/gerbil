# The Gerbil Bootstrap

Gerbil is fully self-hosted, 100% written in itself. So how does it
all fit together?

## The Chain of Trust

The key premise of Gerbil is simple: it is a meta-language, a
meta-dialect of Scheme that bootstraps from precompiled sources using
Gambit.

This has implications for the chain of trust regarding the security of
your software:
- There is _no precompiled binary involved_, nor will there ever be
  one. The bootstrap is purely source based, which means you can
  actually read and audit the bootstrap sources. It is not pretty, but
  it is readable; so if you feel so inclined I recommend taking a look
  at `GERBIL_SRCDIR/src/bootstrap`.
- Just like Gerbil bootstraps from precompiled Gambit code, Gambit
  bootstraps from precompiled C code. This is also auditable, albeit
  not an easy read.
- The bootstrap chain is anchored on the C compiler. Ultimately, If
  you trust your C compiler, then you can _verifiably_ trust the
  Gerbil bootstrap.

For the Gerbil core team, where we all use GCC, this can be
summarized in a quotable one liner:

> In GNU we trust; everyone else pays cash.

## The Long and Arduous History of Bootstrap

The first version of the Gerbil, let's call that the proto-Gerbil, was
bootstrapped by vyzo a long time ago using a hand-written unhygienic
interpreter for the core language.  Once that was done, vyzo wrote the
expander and the first version of the compiler, then the expander
expanded the compiler and the compiler compiled the expander and
itself... and Gerbil was born.

Initially, the runtime was written in Gambit with a set of macros;
that was called `gx-gambc`.  In the v0.18 release cycle, where Gerbil
became fully self hosted, all the traces have disappeared from the
source tree, as they are dead code. They still exist in the
repo's commit history if you want to do some historical research and
peek into the deep past to understand the evolution of Gerbil.


## How Gerbil Builds Itself

The build process can be summarized in the following steps:
1. `configure` configures the system and creates `build-env.sh`.
2. `make` invokes the [top level build script](https://github.com/mighty-gerbils/gerbil/blob/master/build.sh)
3. `build.sh` sources `build-env.sh` and dispatches to the [worker build script](https://github.com/mighty-gerbils/gerbil/blob/master/src/build.sh).
4. The worker `build.sh` sets up the build environment and proceeds in stages:
   1. the build directory structure is prepared
   2. Gambit is bootstrapped and built.
   3. the `boot-gxi` shim is compiled using `gcc`.
   4. the Gerbil bootstrap is compiled to object code using `gsc` (stage 0).
   5. the Gerbil core system and universal binary is compiled using the bootstrap compiler with `boot-gxi` (stage 1).
   6. the newly compiled `gerbil` binary compiles the rest of the system.


## Practical Matters

### Recompiling the Bootstrap
If you have made changes to the core runtime, prelude, expander, or
the compiler itself, then you may also need to update the precompiled
bootstrap modules (e.g. because of a bug fix).

This can be accomplished with the following incantations in `$GERBIL_SRCDIR/src`.

- To compile the bootstrap runtime:
```
gxc -d bootstrap -s -S -O gerbil/runtime/{gambit,system,util,loader,control,mop,error,thread,syntax,eval,repl,init}.ss gerbil/runtime.ss
```

- To compile the bootstrap core prelude:
```
gxc -d bootstrap -s -S -O -no-ssxi gerbil/prelude/core.ss
```

- To compile the bootstrap gambit prelude:
```
gxc -d bootstrap -s -S gerbil/prelude/gambit.ss
```

- To compile the bootstrap expander:
```
gxc -d bootstrap -s -S -O gerbil/expander/{common,stx,core,top,module,compile,root,stxcase}.ss gerbil/expander.ss
```

- To compile the bootstrap compiler:
```
gxc  -d bootstrap -s -S -O gerbil/compiler/{base,compile,optimize-base,optimize-xform,optimize-top,optimize-spec,optimize-ann,optimize-call,optimize,driver,ssxi}.ss gerbil/compiler.ss
```

- Finally, if you've made changes to it, you should also copy the core.ssxi.ss optimizer prelude:
```
cp gerbil/prelude/core.ssxi.ss bootstrap/gerbil
```

### Debugging

If you have been making changes in the core system and building a new
bootstrap, you might have some bugs and need to debug. The gerbil
build script is modular, which allows you to build certain stages at
will and supports serveral commands:
- `prepare` prepares the build environment.
- `gambit` bootstraps and builds Gambit.
- `boot-gxi` builds and bootstrap gxi shim.
- `stage0` builds the Gerbil bootstrap.
- `stage1` builds the Gerbil core modules and the `gerbil` universal
  binary (aka Bach, affectionally named after the great composer of
  timeless music).
- `stdlib` builds the Gerbil standard library.
- `libgerbil` builds the Gerbil (shared) library; this includes the
  Gerbil runtime, prelude, expander and compiler, together with the
  entiry of stdlib.
- `lang` builds the bundled alternative language preludes.
- `r7rs-large` builds the R7RS large libraries included in Gerbil.
- `srfi` builds the top level srfi package shims.
- `tools` builds the gerbil tools.
- `tags` builds a TAGS file for the Gerbil sources so that you can
  easily navigate code in emacs.
- `env` applies the arguments in the build environment.

So if you have made changes and want to rebuild gerbil, you don't have
to redo everything from scratch with `make`; you can simply build the
stage you want, and once you are satisfied you can move to the next
stage or push your branch so that CI does the job for you.

If you have introduced a bug and `gerbil` crashes, do not despair. You
can simply run with `gdb` and get the exact location of the crash to
pinpoint the bug.

Here is an example from debugging a real crash during development:
```
$ ./build.sh stdlib
[*] Building gerbil stdlib
Segmentation fault (core dumped)

$ ./build.sh env gdb gerbil
...
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from gerbil...
(gdb) r ./std/build.ss
Starting program: /home/vyzo/gerbil/build/bin/gerbil ./std/build.ss
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Program received signal SIGSEGV, Segmentation fault.
___H_gerbil____runtime____gambit (___ps=0x7ffff7eda000 <___gstate0>) at /home/vyzo/gerbil/build/lib/static/gerbil__runtime__gambit.scm:4
4	  (define ##max-char (let () (declare (not safe)) (##max-char-code))))
```

Once you have successfully debugged and finished the build, you should run the tests:
```
$ ./build.sh env gerbil test ./...
...
```

And that's it! Happy Hacking.
