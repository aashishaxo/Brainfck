# Brainf

A Brainfuck compiler, built from scratch as a way to learn how compilers work.

**Status:** planning. No code yet. The full roadmap is in [PLAN.md](PLAN.md).

## Brainfuck in one table

A byte tape (30,000 cells, all zero) and a pointer into it. Eight commands; everything else is a comment.

| Cmd | C equivalent |
|-----|--------------|
| `>` | `ptr++;` |
| `<` | `ptr--;` |
| `+` | `(*ptr)++;` |
| `-` | `(*ptr)--;` |
| `.` | `putchar(*ptr);` |
| `,` | `*ptr = getchar();` |
| `[` | `while (*ptr) {` |
| `]` | `}` |

## Planned pipeline

```
source.bf → lexer → parser (IR) → optimizer → backend
                                               ├─ interpreter
                                               ├─ C source   (→ gcc/clang)
                                               └─ x86-64 asm (→ nasm → ld)
```

## Roadmap

1. Interpreter with a precomputed jump table
2. Lexer and parser producing a flat IR
3. Optimizer: run-length folding, clear/scan/copy loops, offset ops
4. C backend
5. Native x86-64 backend (NASM, Linux)
6. Stretch: JIT, LLVM IR, WebAssembly

See [PLAN.md](PLAN.md) for details and references.
