# Brainfuck Compiler: Build Plan

A learn-by-doing project: build a Brainfuck compiler from scratch, one small step at a time, and pick up the core ideas of how real compilers work along the way (lexing, parsing, IR, optimization, code generation, assembling, and linking).

---

## 0. What even is Brainfuck?

Brainfuck (BF) is an *esoteric* programming language created by Urban Müller in 1993. It's tiny but **Turing-complete**, which means it can compute anything any other language can, just very painfully.

### The machine model

Picture this:

- A **tape** of at least 30,000 cells, each one a byte (0–255), all set to `0` at the start.
- A **data pointer** that starts at cell 0.
- An **instruction pointer** that walks through your program one character at a time.

### The entire language (8 commands)

| Cmd | Meaning | C equivalent |
|-----|---------|--------------|
| `>` | move pointer right | `ptr++;` |
| `<` | move pointer left | `ptr--;` |
| `+` | increment current cell | `(*ptr)++;` |
| `-` | decrement current cell | `(*ptr)--;` |
| `.` | output current cell as a character | `putchar(*ptr);` |
| `,` | read one character into current cell | `*ptr = getchar();` |
| `[` | if current cell is 0, jump past the matching `]` | `while (*ptr) {` |
| `]` | if current cell is not 0, jump back to the matching `[` | `}` |

**Every other character is a comment** and gets ignored. That's the whole language.

### Example: print `A` (ASCII 65)

```
++++++++[>++++++++<-]>+.
```
Cell0 = 8. Loop: add 8 to cell1 and decrement cell0, which leaves cell1 = 64. Move to cell1, add 1 to get 65, print it: `A`.

Because every BF command maps directly to a line of C (see the table), **a BF compiler is the smallest possible "real" compiler**. You get to skip the hard language-design parts and focus on the compiler pipeline itself.

---

## 1. Prerequisites (what to know before starting)

You don't need all of this on day 1. Items are tagged by the phase where you'll need them.

### Must know (Phase 1–2)
1. **One general-purpose language, reasonably well.** Recommendation: **C** (closest to the metal, and BF translates straight into it) or **Rust** (modern, great tooling, very GitHub-friendly). Python works for the early phases but is awkward for the later ones.
2. **Basic data structures:** arrays, **stacks** (used to match `[` with `]`), and structs/enums.
3. **ASCII / bytes:** characters are just numbers (`'A' == 65`), and a byte wraps around (`255 + 1 == 0`).
4. **Command-line basics:** running programs, arguments, and stdin/stdout, e.g. `bfc hello.bf -o hello`.
5. **Git + GitHub basics:** commit, branch, push, write a README.

### Needed later (Phase 3–5)
6. **How a program becomes an executable:** source → compiler → assembly → assembler → object file → linker → executable.
7. **x86-64 assembly basics:** registers (`rax`, `rbx`, `rdi`...), `mov`, `add`, `sub`, `cmp`, `jmp`, `je`/`jne`, labels, and memory addressing like `byte [rbx]`.
8. **Calling conventions / syscalls:** how to call `putchar`/`getchar`, or how to make `write`/`read` syscalls directly.
   - ⚠️ You're on **Windows**. Linux assembly is *much* simpler to learn from (syscalls are easy, and there are more tutorials). **Strongly recommended: install WSL2 (Ubuntu)** and do the assembly phases there.
9. **Basic optimization ideas:** folding repeated operations and recognizing patterns (Phase 3 teaches you this).

### Optional / stretch
10. Memory-mapping executable memory (`mmap` / `VirtualAlloc`), for the JIT phase.
11. LLVM IR basics, for the LLVM backend phase.

---

## 2. Documentation & resources to refer to

### Brainfuck itself
- **Esolang wiki: Brainfuck.** The canonical spec, including edge cases (cell size, EOF behavior): https://esolangs.org/wiki/Brainfuck
- **Wikipedia: Brainfuck.** Friendly intro and the C translation table: https://en.wikipedia.org/wiki/Brainfuck
- **Daniel Cristofani's brainfuck.org.** High-quality BF programs and **test programs** for your compiler: http://brainfuck.org/
  - Portability and implementation notes: http://brainfuck.org/epistle.html
  - Test file for tricky cases: http://brainfuck.org/tests.b

### Compiler & optimization for BF specifically
- **Mats Linander, "brainfuck optimization strategies"**, *the* reference for Phase 3: http://calmerthanyouare.org/2015/01/07/optimizing-brainfuck.html
- **Eli Bendersky, "Adventures in JIT compilation"** (4-part series using BF, from interpreter to JIT to LLVM). Basically this whole project, written up as a blog series: https://eli.thegreenplace.net/2017/adventures-in-jit-compilation-part-1-an-interpreter/

### General compiler knowledge
- **Crafting Interpreters** (Robert Nystrom, free online). The best beginner compiler/interpreter book, and very readable: https://craftinginterpreters.com/
- **LLVM Kaleidoscope tutorial**, for the LLVM phase: https://llvm.org/docs/tutorial/
- *(Optional, heavy)* "Compilers: Principles, Techniques, and Tools" (the Dragon Book) or "Engineering a Compiler" (Cooper & Torczon).

### Assembly & low-level
- **NASM manual:** https://www.nasm.us/doc/
- **x86-64 instruction reference (Félix Cloutier):** https://www.felixcloutier.com/x86/
- **Linux x86-64 syscall table:** https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/
- **Microsoft x64 calling convention** (if you target native Windows): https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention
- **Compiler Explorer (godbolt).** Write C and see the assembly it produces; hugely useful for learning codegen: https://godbolt.org/
- **WSL install guide:** https://learn.microsoft.com/en-us/windows/wsl/install

### Git / GitHub
- **Pro Git book (free):** https://git-scm.com/book/en/v2
- **Conventional Commits** (clean commit messages): https://www.conventionalcommits.org/
- **GitHub Actions quickstart** (CI for tests): https://docs.github.com/en/actions/quickstart
- **Why aren't my contributions showing?** Your commit email must match your GitHub account: https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-github-profile/managing-contribution-settings-on-your-profile/why-are-my-contributions-not-showing-up-on-my-profile

---

## 3. Architecture (where we're heading)

```
 hello.bf
    │
    ▼
┌─────────┐   tokens    ┌─────────┐   AST/IR   ┌────────────┐  optimized IR  ┌───────────┐
│  Lexer  │ ──────────► │ Parser  │ ─────────► │ Optimizer  │ ─────────────► │  Backend  │
└─────────┘             └─────────┘            └────────────┘                └─────┬─────┘
                        (checks [ ] match)     (folds, patterns)                   │
                                                                ┌──────────────────┼──────────────────┐
                                                                ▼                  ▼                  ▼
                                                           Interpreter        C source          x86-64 asm
                                                          (runs it now)     (→ gcc/clang)    (→ nasm → ld → exe)
```

The key idea: **one front-end and many back-ends.** Real compilers like GCC and LLVM are structured exactly like this.

### Suggested repo layout (C example; Rust is similar with `src/*.rs`)
```
brainf/
├── README.md
├── PLAN.md
├── Makefile
├── src/
│   ├── main.c          # CLI: bfc <file> [--emit=c|asm|run] [-O0|-O1] [-o out]
│   ├── lexer.c/.h
│   ├── parser.c/.h     # builds IR, checks bracket matching
│   ├── ir.h            # instruction definitions
│   ├── optimizer.c/.h
│   ├── interp.c/.h
│   ├── codegen_c.c
│   └── codegen_x86.c
├── examples/           # hello.bf, mandelbrot.bf, etc.
└── tests/              # test programs + expected outputs + a test runner script
```

---

## 4. Phased roadmap

Each phase ends with something that **works**, so you always have a demo and a natural batch of commits.

### Phase 1: Interpreter (the warm-up)
**Goal:** `bfc run hello.bf` prints `Hello World!`.
- [ ] Set up the repo, README, `.gitignore`, and build system (Makefile / Cargo)
- [ ] Read a file into memory
- [ ] Tape: `uint8_t tape[30000]` plus a pointer
- [ ] Execute the 6 simple commands (`> < + - . ,`)
- [ ] Loops: **precompute a jump table** with a stack (push the index on `[`, pop on `]`, record both directions)
- [ ] Error on unmatched brackets, reporting line and column
- [ ] Decide and document edge cases: cell wrapping (255+1 = 0), EOF on `,` (leave the cell unchanged / set 0 / set 255), and what happens when the pointer goes out of bounds
- [ ] Run `hello.bf` and some programs from brainfuck.org

**You learn:** program execution, the fetch/decode/execute loop, and why a jump table beats scanning for brackets.

### Phase 2: Proper front-end (lexer → parser → IR)
**Goal:** turn text into a structured intermediate representation (IR).
- [ ] **Lexer:** keep only the 8 command chars, and track line/col for errors
- [ ] **Parser:** build the IR. A simple flat IR works well:
  ```
  enum Op { ADD, MOVE, OUTPUT, INPUT, JUMP_IF_ZERO, JUMP_IF_NOT_ZERO }
  struct Instr { Op op; int arg; }   // arg = amount, or jump target
  ```
  (Alternative: a tree/AST where `Loop` contains child nodes. Try both if you're curious.)
- [ ] Rewrite the interpreter to run the IR instead of raw characters
- [ ] Add `--dump-ir` to print the IR (great for debugging)

**You learn:** why compilers separate the front-end from execution, and what IR/AST mean.

### Phase 3: Optimizer (where it gets fun)
**Goal:** make programs run a lot faster. Benchmark with `mandelbrot.bf` (on brainfuck.org).
- [ ] **Run-length folding:** `+++++` becomes `ADD 5` and `>>>` becomes `MOVE 3`, and they cancel, so `+-` becomes nothing
- [ ] **Clear loop:** `[-]` or `[+]` becomes `SET 0`
- [ ] **Scan loops:** `[>]` / `[<]` become `SCAN_RIGHT` / `SCAN_LEFT` (can use `memchr`)
- [ ] **Copy/multiply loops:** `[->+>++<<]` becomes `cell[1] += cell[0]*1; cell[2] += cell[0]*2; cell[0] = 0`
- [ ] **Offset ops:** `>+<` becomes `ADD 1 at offset +1`, which avoids moving the pointer
- [ ] **Dead code:** remove a loop at the very start of the program (cells are all 0, so it never runs)
- [ ] Add `-O0` / `-O1` flags and a benchmark table in the README (before/after timings)

**You learn:** pattern matching on IR, optimization passes, and how to prove an optimization doesn't change behavior. Reference: Mats Linander's article.

### Phase 4: Backend #1, transpile to C
**Goal:** `bfc hello.bf --emit=c` outputs a `.c` file, then `gcc` compiles it to a real executable.
- [ ] Emit a C prelude (`#include <stdio.h>`, `unsigned char tape[30000]; unsigned char *p = tape;`)
- [ ] Emit one C statement per IR instruction, and `while(*p){ ... }` for loops
- [ ] Optionally shell out to `gcc`/`clang` automatically
- [ ] Compare its speed with your interpreter (it'll be much faster)

**You learn:** code generation. This is your first *real* compiler, the "source-to-source" (transpiler) kind.

### Phase 5: Backend #2, native x86-64 assembly 🔥
**Goal:** `bfc hello.bf -o hello` → emits `.asm` → `nasm` → `ld` → native binary. **No C compiler involved.**
*(Do this in WSL/Linux first.)*
- [ ] Hand-write a tiny asm program that prints one char with the `write` syscall, then assemble and link it by hand
- [ ] Reserve the tape in `.bss` (`tape: resb 30000`) and keep the data pointer in a register (e.g. `rbx`)
- [ ] Map IR to asm:
  - `ADD n` → `add byte [rbx], n`
  - `MOVE n` → `add rbx, n`
  - `OUTPUT` → `mov rax,1; mov rdi,1; mov rsi,rbx; mov rdx,1; syscall`
  - `INPUT` → the same with `rax=0, rdi=0`
  - loops → unique labels plus `cmp byte [rbx],0` / `je` / `jne`
  - exit → `mov rax,60; xor rdi,rdi; syscall`
- [ ] Automate: write the `.asm`, then run `nasm -f elf64` and `ld` from your compiler
- [ ] Use godbolt to compare against what GCC generates from your Phase 4 C output
- [ ] *(Stretch)* Support native Windows with `nasm -f win64`, linking against `kernel32` / the C runtime (see the Microsoft calling convention docs)

**You learn:** what compilers *actually* output, plus registers, labels, syscalls, and the assembler/linker toolchain.

### Phase 6: Stretch goals (pick any)
- [ ] **JIT compiler:** emit raw machine-code bytes into `mmap`'d executable memory and jump into it (Eli Bendersky part 2)
- [ ] **LLVM backend:** emit LLVM IR text (`.ll`) and compile it with `clang`, which gets you LLVM's optimizer for free
- [ ] **WebAssembly backend:** emit `.wat` and run BF in the browser
- [ ] **Emit your own ELF file directly** (no nasm/ld at all), the hardcore option
- [ ] **Debugger:** step through code and view the tape (nice TUI)
- [ ] **Reverse direction:** a tiny language that compiles *to* BF

---

## 5. Testing strategy
- [ ] `tests/` folder containing `name.bf`, `name.in` (optional stdin), and `name.out` (expected output)
- [ ] A test runner script that runs **every backend** (interp, C, asm) on every test and diffs the outputs. All backends must agree.
- [ ] Include brainfuck.org's `tests.b` and edge cases: unmatched brackets, empty program, deep nesting, cell wrap, EOF
- [ ] GitHub Actions CI running the tests on every push, plus a green badge in the README ✅

---

## 6. GitHub / commit plan 🟩

Small, **atomic, meaningful** commits make a great contribution graph *and* a readable history (recruiters do look). Aim for one logical change per commit, using Conventional Commit style:

```
chore: initial project setup with Makefile and README
feat(interp): implement tape and pointer movement
feat(interp): add bracket matching with jump table
fix(interp): handle EOF on input command
test: add hello world and cell-wrap tests
feat(parser): introduce flat IR
feat(opt): fold consecutive +/- and </>
feat(opt): replace clear loops with SET 0
perf: add benchmark results for mandelbrot.bf
feat(codegen): add C backend
feat(codegen): add x86-64 NASM backend
ci: add GitHub Actions test workflow
docs: explain optimization passes in README
```

Tips:
- Set your commit email to one linked to your GitHub account (`git config user.email ...`), or the commits won't count on your profile.
- Commits only count on the **default branch** (or once a PR is merged into it). Feature branches + PRs to yourself is good practice anyway.
- Rough estimate: Phase 1 ≈ 8–12 commits, Phase 2 ≈ 5–8, Phase 3 ≈ 10–15, Phase 4 ≈ 5, Phase 5 ≈ 10–15, and tests/docs/CI add plenty more. That comes to **50–80+ commits** of real work.
- Write a good README: what BF is, usage, architecture diagram, benchmark table, and GIFs.

---

## 7. Suggested order of reading
1. Esolang wiki page (15 min). Then write `hello.bf` **by hand** to get a feel for the language.
2. Crafting Interpreters, chapters 1–2 (the big picture of compilers).
3. Build Phase 1 & 2.
4. Mats Linander's optimization article, then build Phase 3.
5. Eli Bendersky part 1 (compare his interpreter with yours).
6. Pick up NASM + syscall basics, then build Phase 5.
7. Eli Bendersky parts 2–4 for JIT/LLVM stretch goals.

Have fun. Once `mandelbrot.bf` renders at native speed through your own compiler, you'll understand compilers far better than any lecture could teach you.
