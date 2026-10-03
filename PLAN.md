# Brainfuck Compiler: Build Plan

A learn-by-doing project: build a Brainfuck compiler from scratch and ship it as a **real, installable toolchain**. Anyone should be able to install it from the GitHub repo and use it from their terminal and code editor, the same way they use `gcc`, `clang`, or `python`.

Along the way you pick up the core ideas behind real compilers (lexing, parsing, IR, optimization, code generation, assembling, linking). You also learn what it takes to ship a language: a CLI, installers, releases, cross-platform builds, and editor support.

### The end goal, from a user's point of view

```powershell
# Install (Windows)
irm https://raw.githubusercontent.com/aashishaxo/Brainfck/main/install/install.ps1 | iex
```
```sh
# Install (Linux / macOS)
curl -fsSL https://raw.githubusercontent.com/aashishaxo/Brainfck/main/install/install.sh | sh
# ...or, with Rust installed
cargo install --git https://github.com/aashishaxo/Brainfck
```
```sh
bfc --version              # bfc 0.4.0
bfc run hello.bf           # run it directly, like `python hello.py`
bfc hello.bf -o hello      # build a standalone executable, like `gcc hello.c -o hello`
./hello                    # Hello World!
```
Then in VS Code: install the **Brainf** extension, open `hello.bf`, and you get syntax highlighting, bracket matching, red squiggles on errors, and a ▶ Run button.

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

Because every BF command maps directly to a line of C (see the table), **a BF compiler is the smallest possible "real" compiler**. You get to skip the hard language-design parts and focus on the compiler pipeline, and on the tooling around it.

---

## 1. How languages plug into your editor (the mental model)

People often assume VS Code "has C/Python built in". It doesn't. Every language toolchain is split into the same independent pieces, and this project builds each of them:

| Piece | What it does | For C | For Python | **For us** |
|-------|--------------|-------|------------|------------|
| **Compiler / runtime** | Turns source into a program, or runs it | `gcc`, `clang` | `python` | **`bfc`** (a CLI) |
| **Installation** | Gets the binary onto the user's machine and **on their `PATH`** | installers, package managers | python.org installer | **GitHub Releases + install scripts / `cargo install --git`** |
| **Syntax highlighting** | Colors the code | VS Code's C/C++ extension | Python extension | **TextMate grammar** in a VS Code extension |
| **Run / build button** | Calls the CLI for you | Code Runner, tasks | Python extension | **Extension command + `tasks.json`** |
| **Error squiggles** | Parses the compiler's errors and shows them inline | problem matcher, clangd | Pylance | **Problem matcher** (easy), later a **language server** (`bfc lsp`) |
| **Debugger** | Breakpoints, stepping, inspecting memory | gdb / lldb via DAP | debugpy via DAP | *(stretch)* **`bfc debug`** via DAP |

The key insight: **the editor never compiles anything itself. It just runs your CLI and reads its output.** So the CLI has to behave like a well-mannered compiler: predictable flags, proper exit codes, and errors in the standard `file:line:col: error: message` format that every editor already knows how to parse.

---

## 2. Prerequisites (what to know before starting)

You don't need all of this on day 1. Items are tagged by the phase where you'll need them.

### Must know (Phase 1–3)
1. **One general-purpose language, reasonably well.** Recommendation: **Rust**. Now that the goal is a distributable tool, Rust has big advantages:
   - `cargo install --git https://github.com/aashishaxo/Brainfck` installs straight from your repo, with no extra work.
   - It produces a single static binary per platform with no runtime dependencies.
   - It cross-compiles cleanly to Windows/Linux/macOS, and tools like `cargo-dist` automate GitHub Releases and installers.
   - Good crates exist for the later parts: `clap` (CLI), `tower-lsp` (language server).

   **C is still a valid choice** (closest to the metal, and BF translates straight into it), but you'll hand-roll more: a `Makefile` with `make install PREFIX=...`, a Windows build (MSVC or MinGW), and your own release workflow. Python is not a good fit here: users would need Python installed to use your compiler.
2. **Basic data structures:** arrays, **stacks** (used to match `[` with `]`), and structs/enums.
3. **ASCII / bytes:** characters are just numbers (`'A' == 65`), and a byte wraps around (`255 + 1 == 0`).
4. **Command-line basics:** arguments, stdin/stdout/stderr, **exit codes**, and the **`PATH` environment variable** (how typing `gcc` finds the program). Know how to edit `PATH` on Windows and in `~/.bashrc` / `~/.zshrc`.
5. **Git + GitHub basics:** commit, branch, push, write a README, **tags**, and **GitHub Releases**.

### Needed later (Phase 4–8)
6. **Packaging & releases:** semantic versioning (`MAJOR.MINOR.PATCH`), a changelog, GitHub Actions **build matrices** (one job per OS), and attaching binaries to a release.
7. **VS Code extension basics:** `package.json` "contributes" (languages, grammars, commands), TextMate grammars (JSON + regex), and a little **TypeScript** for commands.
8. **How a program becomes an executable:** source → compiler → assembly → assembler → object file → linker → executable. Also what an executable file actually *is* (PE on Windows, ELF on Linux, Mach-O on macOS).
9. **x86-64 assembly basics:** registers (`rax`, `rbx`, `rdi`...), `mov`, `add`, `sub`, `cmp`, `jmp`, `je`/`jne`, labels, and memory addressing like `byte [rbx]`.
10. **Calling conventions / syscalls:** the Linux syscall convention, *and* the **Microsoft x64 calling convention** (Windows has no stable syscall interface, so you call `kernel32.dll` functions like `WriteFile` / `ReadFile` / `ExitProcess`). Since your users will be on Windows too, Windows support is no longer optional.
11. **Basic optimization ideas:** folding repeated operations and recognizing patterns (Phase 5 teaches you this).

### Optional / stretch
12. **Language Server Protocol (LSP)** and **Debug Adapter Protocol (DAP)**, for rich editor support.
13. Memory-mapping executable memory (`mmap` / `VirtualAlloc`), for the JIT phase.
14. LLVM IR basics, for the LLVM backend phase.

---

## 3. Documentation & resources to refer to

### Brainfuck itself
- **Esolang wiki: Brainfuck.** The canonical spec, including edge cases (cell size, EOF behavior): https://esolangs.org/wiki/Brainfuck
- **Wikipedia: Brainfuck.** Friendly intro and the C translation table: https://en.wikipedia.org/wiki/Brainfuck
- **Daniel Cristofani's brainfuck.org.** High-quality BF programs and **test programs** for your compiler: http://brainfuck.org/
  - Portability and implementation notes: http://brainfuck.org/epistle.html
  - Test file for tricky cases: http://brainfuck.org/tests.b

### Compiler & optimization for BF specifically
- **Mats Linander, "brainfuck optimization strategies"**, *the* reference for the optimizer phase: http://calmerthanyouare.org/2015/01/07/optimizing-brainfuck.html
- **Eli Bendersky, "Adventures in JIT compilation"** (4-part series using BF, from interpreter to JIT to LLVM): https://eli.thegreenplace.net/2017/adventures-in-jit-compilation-part-1-an-interpreter/

### General compiler knowledge
- **Crafting Interpreters** (Robert Nystrom, free online). The best beginner compiler/interpreter book, and very readable: https://craftinginterpreters.com/
- **LLVM Kaleidoscope tutorial**, for the LLVM phase: https://llvm.org/docs/tutorial/
- *(Optional, heavy)* "Compilers: Principles, Techniques, and Tools" (the Dragon Book) or "Engineering a Compiler" (Cooper & Torczon).

### Building a good CLI
- **Command Line Interface Guidelines** (flags, help text, exit codes, stderr vs stdout): https://clig.dev/
- **GCC diagnostic message format** (the `file:line:col: error:` convention editors parse): https://www.gnu.org/prep/standards/html_node/Errors.html
- **`clap`** (Rust CLI parser): https://docs.rs/clap/

### Packaging, installing & releasing
- **The Cargo Book: `cargo install`** (including `--git`): https://doc.rust-lang.org/cargo/commands/cargo-install.html
- **`cargo-dist`** (builds per-OS binaries, GitHub Releases, and shell/PowerShell installers for you): https://opensource.axo.dev/cargo-dist/
- **GitHub Releases:** https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases
- **GitHub Actions build matrix** (build on Windows/Linux/macOS): https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs
- **Semantic Versioning:** https://semver.org/
- **Keep a Changelog:** https://keepachangelog.com/
- *(Later)* **Scoop** (Windows) and **Homebrew taps** (macOS/Linux), which let people `scoop install bfc` / `brew install bfc` straight from a repo you own: https://scoop.sh/ and https://docs.brew.sh/Taps

### Editor integration
- **VS Code: Your First Extension:** https://code.visualstudio.com/api/get-started/your-first-extension
- **VS Code: Syntax Highlight Guide** (TextMate grammars): https://code.visualstudio.com/api/language-extensions/syntax-highlight-guide
- **VS Code: Language Configuration Guide** (brackets, comments, auto-closing): https://code.visualstudio.com/api/language-extensions/language-configuration-guide
- **VS Code: Tasks & problem matchers:** https://code.visualstudio.com/docs/editor/tasks#_defining-a-problem-matcher
- **VS Code: Publishing extensions** (`vsce`, Marketplace, `.vsix` files): https://code.visualstudio.com/api/working-with-extensions/publishing-extension
- **Open VSX** (the extension registry for VSCodium, Cursor, etc.): https://open-vsx.org/
- **Language Server Protocol:** https://microsoft.github.io/language-server-protocol/ and **`tower-lsp`**: https://docs.rs/tower-lsp/
- **Debug Adapter Protocol:** https://microsoft.github.io/debug-adapter-protocol/

### Assembly, executables & low-level
- **NASM manual:** https://www.nasm.us/doc/
- **x86-64 instruction reference (Félix Cloutier):** https://www.felixcloutier.com/x86/
- **Linux x86-64 syscall table:** https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/
- **Microsoft x64 calling convention:** https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention
- **PE file format** (Windows executables): https://learn.microsoft.com/en-us/windows/win32/debug/pe-format
- **ELF format** (`man 5 elf`), plus the classic "A Whirlwind Tutorial on Creating Really Teensy ELF Executables": https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html
- **Compiler Explorer (godbolt).** Write C and see the assembly it produces; hugely useful for learning codegen: https://godbolt.org/
- **WSL install guide** (handy for testing the Linux build on your Windows machine): https://learn.microsoft.com/en-us/windows/wsl/install

### Git / GitHub
- **Pro Git book (free):** https://git-scm.com/book/en/v2
- **Conventional Commits** (clean commit messages): https://www.conventionalcommits.org/
- **GitHub Actions quickstart** (CI for tests): https://docs.github.com/en/actions/quickstart
- **Why aren't my contributions showing?** Your commit email must match your GitHub account: https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-github-profile/managing-contribution-settings-on-your-profile/why-are-my-contributions-not-showing-up-on-my-profile

---

## 4. Architecture (where we're heading)

### The compiler

```
 hello.bf
    │
    ▼
┌─────────┐   tokens    ┌─────────┐   AST/IR   ┌────────────┐  optimized IR  ┌───────────┐
│  Lexer  │ ──────────► │ Parser  │ ─────────► │ Optimizer  │ ─────────────► │  Backend  │
└─────────┘             └─────────┘            └────────────┘                └─────┬─────┘
                        (checks [ ] match,      (folds, patterns)                  │
                         emits diagnostics)                                        │
              ┌─────────────────────┬──────────────────────┬──────────────────────┼─────────────────┐
              ▼                     ▼                      ▼                      ▼                 ▼
         Interpreter         Bundled exe              C source             x86-64 asm         Direct PE/ELF
        (bfc run)        (runner stub + IR,       (→ gcc/clang/cl)     (→ nasm → linker)   (stretch: no tools
                          zero dependencies)                                                 needed at all)
```

The key idea: **one front-end and many back-ends.** Real compilers like GCC and LLVM are structured exactly like this.

### The product around it

```
            GitHub repo (aashishaxo/Brainfck)
                     │  git tag v0.x.y
                     ▼
           GitHub Actions release workflow
     ┌───────────────┼────────────────┐
     ▼               ▼                ▼
 bfc-windows-x64  bfc-linux-x64   bfc-macos-arm64      + brainf-x.y.z.vsix
     └───────────────┼────────────────┘
                     ▼
              GitHub Release  ◄── install.ps1 / install.sh download from here
                     │              cargo install --git builds from source
                     ▼
            user's machine: bfc on PATH
                     ▲
                     │ runs `bfc check/run/build`, parses errors
            VS Code + Brainf extension
```

### The CLI contract (design this early; editors and users depend on it)

```
bfc run <file.bf> [--tape-size N] [--eof=unchanged|zero|max]     run immediately (interpreter)
bfc <file.bf> -o <out> [-O0|-O1|-O2] [--backend=auto|bundle|c|asm]  build an executable
bfc build ...                                                      same as above, explicit form
bfc check <file.bf>                                                report errors only, run nothing
bfc <file.bf> --emit=ir|c|asm                                      print an intermediate stage
bfc fmt <file.bf>                                                  (later) normalize formatting
bfc lsp                                                            (later) start the language server
bfc --version / bfc --help
```

Rules:
- **Errors go to stderr** in GCC format: `hello.bf:3:14: error: unmatched '['`. Editors' problem matchers and `$gcc` parse this format for free.
- **Exit codes:** `0` success, `1` compile error, `2` bad usage, and when running, pass through the program's runtime error as non-zero.
- **`-o` defaults sensibly:** `hello.exe` on Windows, `hello` elsewhere (like `gcc` → `a.out`, but nicer).
- **Default backend must work on a fresh machine.** Users won't have `nasm` or `gcc`, so `--backend=auto` picks the fastest backend whose tools are installed and falls back to `bundle`, which needs nothing.
- **Document the language semantics** in the README, since users now depend on them: 8-bit wrapping cells, 30,000-cell tape (configurable), EOF behavior (configurable), pointer out of bounds = runtime error with a clear message.

### Suggested repo layout (Rust)
```
Brainfck/
├── README.md               # install, usage, editor setup, benchmarks
├── PLAN.md
├── LICENSE                 # MIT or Apache-2.0, so people can legally use it
├── CHANGELOG.md
├── Cargo.toml
├── src/
│   ├── main.rs             # CLI entry (clap): run / build / check / --emit
│   ├── diagnostics.rs      # error type + GCC-style formatting
│   ├── lexer.rs
│   ├── parser.rs           # builds IR, checks bracket matching
│   ├── ir.rs               # instruction definitions
│   ├── optimizer.rs
│   ├── interp.rs
│   ├── backend/
│   │   ├── bundle.rs       # runner stub + serialized IR → standalone exe
│   │   ├── c.rs
│   │   └── x86_64/         # linux.rs (syscalls), windows.rs (kernel32)
│   └── lsp.rs              # (later) language server
├── editors/
│   └── vscode/             # the VS Code extension
│       ├── package.json
│       ├── language-configuration.json
│       ├── syntaxes/brainfuck.tmLanguage.json
│       └── src/extension.ts
├── install/
│   ├── install.ps1         # Windows installer script
│   └── install.sh          # Linux/macOS installer script
├── examples/               # hello.bf, mandelbrot.bf, etc.
├── tests/                  # test programs + expected outputs + runner
└── .github/workflows/
    ├── ci.yml              # test on Windows/Linux/macOS for every push
    └── release.yml         # build + publish binaries on tag push
```
*(C equivalent: `src/*.c/.h`, a `Makefile` with `install`/`uninstall` targets, and everything else the same.)*

---

## 5. Phased roadmap

Each phase ends with something that **works** and, from Phase 3 on, something **users can install**. Every phase is also a natural batch of commits and a release.

### Phase 1: CLI skeleton + interpreter (the warm-up)
**Goal:** `bfc run hello.bf` prints `Hello World!`, and `bfc` is installed on *your own* `PATH` from day 1.
- [ ] Set up the repo: README, `LICENSE`, `.gitignore`, `Cargo.toml` (or Makefile)
- [ ] CLI skeleton with subcommands, `--help`, and `--version` (read the version from `Cargo.toml`)
- [ ] Install it locally: `cargo install --path .` (or `make install`), then run `bfc` from any folder. **Dogfood your own installer from the start.**
- [ ] Read a file into memory; handle "file not found" with a clean error and exit code
- [ ] Tape: `[u8; 30000]` plus a pointer
- [ ] Execute the 6 simple commands (`> < + - . ,`); flush stdout properly (and make sure output works in the Windows console)
- [ ] Loops: **precompute a jump table** with a stack (push the index on `[`, pop on `]`, record both directions)
- [ ] Decide and document edge cases: cell wrapping (255+1 = 0), EOF on `,` (leave unchanged / set 0 / set 255, behind an `--eof` flag), and pointer out of bounds (runtime error)
- [ ] Run `hello.bf` and some programs from brainfuck.org

**You learn:** program execution, the fetch/decode/execute loop, why a jump table beats scanning for brackets, and how `PATH` + installed binaries work.

### Phase 2: Proper front-end (lexer → parser → IR) + editor-grade diagnostics
**Goal:** turn text into a structured IR, and report errors the way real compilers do.
- [ ] **Lexer:** keep only the 8 command chars, and track line/col for every token
- [ ] **Parser:** build the IR. A simple flat IR works well:
  ```
  enum Op { Add, Move, Output, Input, JumpIfZero, JumpIfNotZero }
  struct Instr { op: Op, arg: i32, span: Span }   // arg = amount or jump target; span = source location
  ```
  (Alternative: a tree/AST where `Loop` contains child nodes. Try both if you're curious.)
- [ ] **Diagnostics:** `file:line:col: error: message` on stderr, report **all** errors rather than just the first, and optionally show the source line with a `^` caret under the problem
- [ ] Warnings too, e.g. `warning: loop at program start never runs`
- [ ] `bfc check file.bf` (errors only, nothing runs). This is what the editor will call.
- [ ] Rewrite the interpreter to run the IR instead of raw characters
- [ ] Add `--emit=ir` to print the IR (great for debugging)

**You learn:** why compilers separate the front-end from execution, what IR/AST mean, and why error-message format is a public API.

### Phase 3: First public release (v0.1.0) 📦
**Goal:** a stranger can install `bfc` from your GitHub repo on Windows, Linux, or macOS and run `bfc run hello.bf`.
- [ ] Make `cargo install --git https://github.com/aashishaxo/Brainfck` work, and document it
- [ ] **CI** (`ci.yml`): build and test on `windows-latest`, `ubuntu-latest`, and `macos-latest` on every push
- [ ] **Release workflow** (`release.yml`): on pushing a tag like `v0.1.0`, build per-OS binaries and attach them to a GitHub Release (hand-written, or generated with `cargo-dist`)
- [ ] **Install scripts:**
  - `install.ps1`: download the right `.zip` from the latest release, extract it to `%LOCALAPPDATA%\bfc\bin`, and add that folder to the user's `PATH`
  - `install.sh`: detect OS/arch, download, put it in `~/.local/bin`, and tell the user if that isn't on `PATH`
- [ ] Document **uninstalling** too (delete the folder, remove the `PATH` entry / `cargo uninstall bfc`)
- [ ] Start `CHANGELOG.md`, follow SemVer (`0.x` = "still changing"), and write proper release notes
- [ ] Test the install on a clean machine: a fresh Windows user account, WSL, or a GitHub Actions job that runs the install script
- [ ] Note in the README: unsigned Windows binaries may trigger a SmartScreen warning (code signing is a later, optional step)

**You learn:** cross-platform builds, CI/CD, release engineering, and what "it works on my machine" really costs.

### Phase 4: VS Code extension 🎨
**Goal:** opening a `.bf` file in VS Code feels like opening a `.c` or `.py` file.
- [ ] Scaffold an extension in `editors/vscode/` (`npx --package yo --package generator-code -- yo code`)
- [ ] **Register the language:** id `brainfuck`, extensions `.bf` and `.b`, plus an icon
- [ ] **Syntax highlighting** with a TextMate grammar: give `+-`, `<>`, `.,`, and `[]` distinct scopes, and color everything else as a comment
- [ ] **Language configuration:** bracket pairs `[ ]` (free bracket matching + auto-closing), and a comment style
- [ ] **Run / Build commands:** a ▶ button in the editor title bar that runs `bfc run ${file}` in a terminal, plus a "Build" command that runs `bfc ${file} -o ...`
- [ ] **Problem matcher:** a task that runs `bfc check ${file}` and turns `file:line:col: error:` lines into red squiggles and the Problems panel
- [ ] Setting `brainf.compilerPath`, with a friendly message ("bfc not found: install it from ...") if `bfc` isn't on `PATH`
- [ ] Package it as a `.vsix` with `vsce package`, attach it to GitHub Releases, and document `code --install-extension brainf-x.y.z.vsix`
- [ ] *(Optional)* Publish to the VS Code Marketplace and Open VSX so users can search "Brainf" in the Extensions panel
- [ ] *(Optional)* Snippets, e.g. `clear` → `[-]`, `print-char` → a template

**You learn:** how editors integrate languages, TextMate grammars, the VS Code extension API, and a bit of TypeScript.

### Phase 5: Optimizer (where it gets fun)
**Goal:** make programs run a lot faster. Benchmark with `mandelbrot.bf` (on brainfuck.org).
- [ ] **Run-length folding:** `+++++` becomes `Add 5` and `>>>` becomes `Move 3`, and they cancel, so `+-` becomes nothing
- [ ] **Clear loop:** `[-]` or `[+]` becomes `Set 0`
- [ ] **Scan loops:** `[>]` / `[<]` become `ScanRight` / `ScanLeft` (can use `memchr`)
- [ ] **Copy/multiply loops:** `[->+>++<<]` becomes `cell[1] += cell[0]*1; cell[2] += cell[0]*2; cell[0] = 0`
- [ ] **Offset ops:** `>+<` becomes `Add 1 at offset +1`, which avoids moving the pointer
- [ ] **Dead code:** remove a loop at the very start of the program (cells are all 0, so it never runs)
- [ ] `-O0` / `-O1` / `-O2` flags (default `-O1`, like most compilers), and a benchmark table in the README (before/after timings)

**You learn:** pattern matching on IR, optimization passes, and how to prove an optimization doesn't change behavior. Reference: Mats Linander's article.

### Phase 6: Producing executables, the zero-dependency way
**Goal:** `bfc hello.bf -o hello` produces a standalone executable on **any** machine, even one with no C compiler or assembler.
- [ ] **Bundle backend (the default fallback):** ship a small precompiled *runner* (the optimized interpreter) inside `bfc`. To "compile", copy the runner for the target OS and append the serialized, optimized IR plus a small footer (magic bytes + length). At startup the runner reads its own file, finds the footer, and runs the payload.
  - This is the same trick self-contained tools like PyInstaller use. It's not native code yet, but it's honest about that, and it **always works**.
- [ ] **C backend:** `--backend=c` emits C (`--emit=c` to just print it) and invokes `cc`/`gcc`/`clang` on Linux/macOS, or `cl.exe` / `clang` / `gcc` (MinGW) on Windows
  - Emit a prelude (`#include <stdio.h>`, `unsigned char tape[30000]; unsigned char *p = tape;`), one statement per IR instruction, and `while(*p){ ... }` for loops
- [ ] **`--backend=auto`:** detect which toolchains are installed and pick the fastest available, falling back to `bundle`. Print which one was used with `-v`.
- [ ] Compare speeds: interpreter vs bundle vs C

**You learn:** code generation (the C backend is your first *real* compiler, the "source-to-source" kind), executable file layout, and designing for users whose machines aren't yours.

### Phase 7: Native x86-64 backend (Windows + Linux) 🔥
**Goal:** `bfc hello.bf -o hello --backend=asm` emits real machine code via assembly, on both of the main platforms.
- [ ] Hand-write a tiny asm program that prints one char, then assemble and link it by hand. Do it **once for Linux** (`write` syscall, in WSL) and **once for Windows** (`GetStdHandle` + `WriteFile` + `ExitProcess` from `kernel32`).
- [ ] Reserve the tape in `.bss` (`tape: resb 30000`) and keep the data pointer in a register (e.g. `rbx`, which is callee-saved on both platforms)
- [ ] Map IR to asm:
  - `Add n` → `add byte [rbx], n`
  - `Move n` → `add rbx, n`
  - `Output`: Linux → `mov rax,1; mov rdi,1; mov rsi,rbx; mov rdx,1; syscall`; Windows → call `WriteFile` (mind the 32-byte shadow space and stack alignment)
  - `Input` → the same with `read` / `ReadFile`
  - loops → unique labels plus `cmp byte [rbx],0` / `je` / `jne`
  - exit → `mov rax,60; xor rdi,rdi; syscall` / `call ExitProcess`
- [ ] Automate: write the `.asm`, run `nasm -f elf64` + `ld` (Linux) or `nasm -f win64` + `link.exe` / `lld-link` / `gcc` (Windows)
- [ ] Hook it into `--backend=auto` (used only when `nasm` + a linker are found)
- [ ] Use godbolt to compare against what GCC generates from your Phase 6 C output
- [ ] *(Stretch)* macOS: `nasm -f macho64`, plus Apple Silicon (arm64) means a second instruction set

**You learn:** what compilers *actually* output, plus registers, labels, syscalls vs OS APIs, calling conventions, and the assembler/linker toolchain on two operating systems.

### Phase 8: Rich language tooling
**Goal:** editor support on par with "real" languages.
- [ ] **Language server** (`bfc lsp`, built with `tower-lsp`): live diagnostics as you type (no saving needed), plus hover (show the IR for a loop, or "net pointer movement: +3")
- [ ] Switch the VS Code extension from the problem matcher to the language server. LSP also gives you **Neovim, Helix, Zed, Sublime** support almost for free, so document how to configure each.
- [ ] **Formatter** (`bfc fmt`): normalize indentation of loops and strip or keep comments; hook it into "Format Document"
- [ ] **Debugger** (`bfc debug` speaking DAP): breakpoints, step, and a "Tape" view that shows cells and the pointer in the VS Code debug sidebar
- [ ] **More install channels:** a Scoop bucket (`scoop install bfc`) and a Homebrew tap (`brew install aashishaxo/tap/bfc`), both just repos you own

**You learn:** how modern editor tooling is protocol-based, and why that lets one server support every editor.

### Phase 9: Stretch goals (pick any)
- [ ] **Direct PE/ELF writer:** emit machine code bytes and write the executable file yourself, so no `nasm`/linker is needed and native code becomes the zero-dependency default. The hardcore option.
- [ ] **JIT compiler:** emit raw machine code into `mmap`/`VirtualAlloc`'d executable memory and jump into it (Eli Bendersky part 2). Could make `bfc run` blazing fast.
- [ ] **LLVM backend:** emit LLVM IR text (`.ll`) and compile it with `clang`, which gets you LLVM's optimizer for free
- [ ] **WebAssembly backend:** emit `.wat` and run BF in the browser (a web playground for the README!)
- [ ] **Code signing** for Windows binaries, so SmartScreen stays quiet
- [ ] **Reverse direction:** a tiny language that compiles *to* BF

---

## 6. Testing strategy
- [ ] `tests/` folder containing `name.bf`, `name.in` (optional stdin), and `name.out` (expected output)
- [ ] A test runner that runs **every backend** (interp, bundle, C, asm) on every test and diffs the outputs. All backends must agree.
- [ ] **Diagnostic tests:** `name.err` files with the exact expected stderr, so you notice if you break the error format editors rely on
- [ ] **CLI tests:** exit codes, `--help`, `--version`, missing file, bad flags
- [ ] Include brainfuck.org's `tests.b` and edge cases: unmatched brackets, empty program, deep nesting, cell wrap, EOF, pointer out of bounds
- [ ] Watch for **line endings**: Windows users will have `\r\n` files, and column numbers must still be right. Add a `.gitattributes` so test fixtures don't get rewritten.
- [ ] **CI on all three OSes** for every push, plus a job that runs the install scripts against the latest release, and a green badge in the README ✅
- [ ] Extension: at least a smoke test that the grammar loads and the commands are registered (`@vscode/test-electron`)

---

## 7. GitHub / commit plan 🟩

Small, **atomic, meaningful** commits make a great contribution graph *and* a readable history (recruiters do look). Aim for one logical change per commit, using Conventional Commit style:

```
chore: initial project setup with Cargo and MIT license
feat(cli): add run subcommand, --help and --version
feat(interp): implement tape and pointer movement
feat(interp): add bracket matching with jump table
fix(interp): handle EOF on input command
feat(diag): report errors in file:line:col format
feat(cli): add check subcommand
ci: test on windows, linux and macos
ci: publish binaries to GitHub Releases on tag
feat(install): add PowerShell and shell install scripts
docs: add installation and uninstall instructions
chore(release): v0.1.0
feat(vscode): add language registration and syntax highlighting
feat(vscode): add run command and problem matcher
feat(opt): fold consecutive +/- and </>
feat(opt): replace clear loops with Set 0
perf: add benchmark results for mandelbrot.bf
feat(backend): add zero-dependency bundle backend
feat(backend): add C backend with toolchain detection
feat(backend): add x86-64 backend for linux
feat(backend): add x86-64 backend for windows
feat(lsp): add language server with live diagnostics
```

Tips:
- Set your commit email to one linked to your GitHub account (`git config user.email ...`), or the commits won't count on your profile.
- Commits only count on the **default branch** (or once a PR is merged into it). Feature branches + PRs to yourself is good practice anyway.
- **Tag a release at the end of each phase** from Phase 3 on (`v0.1.0` installable interpreter, `v0.2.0` editor extension, `v0.3.0` optimizer, `v0.4.0` standalone executables, ...). Releases show up on your repo's front page and give users a reason to come back.
- Rough estimate: Phase 1 ≈ 8–12 commits, Phase 2 ≈ 6–10, Phase 3 ≈ 8–12, Phase 4 ≈ 8–12, Phase 5 ≈ 10–15, Phase 6 ≈ 8–10, Phase 7 ≈ 12–18, and tests/docs/CI add plenty more. That comes to **80–120+ commits** of real work.
- Write a good README, organized like a real tool's: one-line install, quick start, editor setup, language semantics, CLI reference, architecture diagram, benchmark table, and GIFs of the VS Code extension in action.

---

## 8. Suggested order of reading
1. Esolang wiki page (15 min). Then write `hello.bf` **by hand** to get a feel for the language.
2. Crafting Interpreters, chapters 1–2 (the big picture of compilers).
3. clig.dev (30 min skim) before designing the CLI.
4. Build Phases 1 & 2.
5. GitHub Actions quickstart + the `cargo-dist` docs, then build Phase 3 (ship v0.1.0!).
6. VS Code "Your First Extension" + "Syntax Highlight Guide", then build Phase 4.
7. Mats Linander's optimization article, then build Phase 5.
8. Eli Bendersky part 1 (compare his interpreter with yours), then build Phase 6.
9. Pick up NASM, Linux syscalls, and the Microsoft x64 calling convention, then build Phase 7.
10. LSP overview + `tower-lsp` examples for Phase 8; Eli Bendersky parts 2–4 for the JIT/LLVM stretch goals.

Have fun. Once someone you've never met installs `bfc` from your repo, opens `mandelbrot.bf` in VS Code, and hits ▶, you'll have built a real language toolchain, not just a compiler.
