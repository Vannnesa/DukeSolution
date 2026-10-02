# git-homework

A local, offline reimplementation of the Duke "Programming Practice" `grade`
tool, from the *Introductory C Programming* specialization on Coursera.

Work through the assignments the way the course intended: get one, write it,
run `grade`, and on a pass the next one appears.

```console
$ grade init ~/duke-homework
$ cd ~/duke-homework/00_hello
$ bun grade
```

No daemon, no git push, no network.

---

## How it works

Two things that are deliberately kept apart:

| | what it is | where |
|---|---|---|
| **package** | this checkout — a read-only *index* of configs, templates, reconstructed grader drivers and reference solutions | `data/` |
| **workspace** | wherever you are doing the work | anywhere else |

The original course did the same: `/git-homework` was the bare repository the
grader read from, and your home directory was the workspace. Nothing is ever
written into the package, so you can have many workspaces and the index stays
pristine.

## Install

Requires [Bun](https://bun.sh).

```bash
git clone <this repo> ~/DukeSolution
cd ~/DukeSolution
bun install          # optional: only needed for typechecking and editor support
grade install        # not yet available — see below
```

The last step cannot work yet, because `grade` does not exist until it has been
installed. Run it through Bun the first time:

```bash
cd ~/DukeSolution
bun src/index.ts install
```

That writes a small launcher into the first writable directory on your `PATH`
(`~/.local/bin`, then Homebrew's `bin`, then `/usr/local/bin`). It reports
exactly where it went, and if that directory is not on your `PATH` it prints the
one line to add to your shell's rc file.

`bun install` is genuinely optional: the tool has no runtime dependencies, so
`bun src/index.ts` works in a fresh clone with no `node_modules` at all. Install
it only if you want `bunx tsc --noEmit` to run.

## Use

Two commands, from anywhere:

```console
$ cd ~
$ grade init
You are in your home directory, so the workspace goes in ~/duke-homework instead.

Workspace ready: /Users/you/duke-homework
Your first assignment is 00_hello/:
  README
  hello.txt

$ cd ~/duke-homework/00_hello
$ grade
```

`grade init` with no argument uses the current directory, except when that is
your home directory — 42 assignment folders loose in `~` would be a mess, so it
uses `~/duke-homework` and says so. Pass a directory to override:

```bash
grade init ~/school/duke
```

Once `grade` is on your `PATH`, the workspace also carries a `./grade` launcher,
and `bun grade` works inside it too (the workspace gets its own `package.json`).
All three are equivalent.

| command | what it does |
|---|---|
| `grade` | grade the assignment you are in, or the workspace's current one |
| `grade init [dir]` | create a workspace and hand out the first assignment |
| `grade status` | which assignments are done / current / locked |
| `grade list` | every assignment and its strategy |
| `grade new <id>` | copy one assignment into the workspace, untouched |
| `grade check` | validate every config in the index |
| `grade install` | put `grade` on your `PATH` |
| `grade uninstall` | remove the launcher |
| `grade doctor` | report anything that would stop grading working |

`grade` works from the assignment directory, from the workspace root, or from
any subdirectory of either — it walks up to find the workspace and falls back to
the recorded current assignment.

Useful flags: `--workspace <dir>`, `--id <id>`, `--dir <dir>`, `--explain`,
`--quiet`, `--no-answer`, `--no-scaffold`.

If something is wrong, start here:

```console
$ grade doctor
  ok    package          /Users/you/DukeSolution
  ok    bun              /opt/homebrew/bin/bun
  ok    C compiler       /usr/bin/cc
  ok    grade on PATH    /opt/homebrew/bin/grade
  ok    leak checker     /usr/bin/leaks (ASan + leaks)
  ok    line endings     .gitattributes present (no EOL conversion)
```

### What a pass gives you

1. `grade.txt` in the assignment directory.
2. `answer/user_code/` — your files.
3. `answer/reference/` — the reference solution.
4. `answer/diff.txt` — a unified diff of the two (`-` is the reference).
5. The next assignment, copied from `data/templates/`.

A failure writes only `grade.txt`, naming the failing case with
`Expected:` / `Got:` side by side.

### Exit status

| code | meaning |
|---|---|
| 0 | `PASSED` / `A` |
| 1 | `FAILED` |
| 3 | `NOT GRADED` — the assignment cannot be checked on this platform |
| 2 | usage or configuration error |

`NOT GRADED` exists because *"we cannot check this here"* is not the same as
*"you got it wrong"*. It never unlocks anything.

## Assignments

42 assignments across seven strategies:

| strategy | how it grades | count |
|---|---|---|
| `compile_and_run` | build, run with fixed argv/stdin, diff stdout | 16 |
| `memory_check` | as above, plus a memory-safety verdict (two passes) | 8 |
| `predict_output` | run a provided program; check your predicted output | 6 |
| `test_identify` | your test harness must catch every broken variant | 0 (see below) |
| `source_check` | required / forbidden declarations, then behaviour | 4 |
| `file_compare` | your file's bytes must match | 3 |
| `unsupported` | cannot be graded here; says why and returns 3 | 5 |

Five assignments are `unsupported`: the teacher programs they test against
(`/usr/local/l2p/...` on the original grading VM) did not survive into the
upstream snapshot this index was reverse-engineered from.
`REVERSE_ENGINEERING.md` §5.5 has the details.

Adding an assignment means adding one config and one template. `src/` contains
no assignment-specific knowledge — see `SCHEMA.md`.

---

## Platform support

| | status |
|---|---|
| **macOS** (Apple Silicon or Intel) | fully working; this is what it was developed and verified on |
| **Linux** | expected to work — and `memory_check` uses **valgrind**, the engine the original course used, which is better than what macOS can offer |
| **Windows** | works only with a gcc-compatible toolchain on `PATH`. MSVC's `cl` is not supported: its flags and conventions differ from the gcc this course assumes. |

Two things are handled for you automatically.

**Line endings.** `data/configs/expected/` fixtures, the templates and the
flattened symlink files are all compared or interpreted as raw bytes, so an
LF → CRLF conversion on checkout would silently break every one of them. A
`.gitattributes` with `* -text` disables that conversion in both directions —
do not remove it. If you author a file with Windows line endings anyway, the
grader says so explicitly rather than printing an `Expected:`/`Got:` pair that
looks identical:

```
Your file did not match the expected output
Your file uses Windows line endings (CRLF) on 1 line(s); this assignment
expects Unix line endings (LF). Re-save it with LF endings, or run the file
through `dos2unix`.
```

**A missing C compiler.** Most assignments compile C, so one is required.
Rather than a raw `posix_spawn ENOENT`, the grader names the problem and the
fix for your platform:

```
grade: no C compiler found (tried "cc").
Most assignments compile C, so one is required.

Install the Xcode command line tools:  xcode-select --install

Set CC=<path> to point at a specific compiler.
```

Only `*.sh` dev tooling under `tools/` is Unix-specific; the `grade` command
itself is TypeScript and has no shell dependency.

## Platform notes (read these)

This was reverse-engineered from a completed student workspace and runs on
macOS/arm64, while the original course ran Ubuntu 20.04 / gcc 9.3.0. Three
consequences:

1. **The teacher's `.o` files cannot be used.** They are Linux x86-64 ELF.
   Every grader driver was reconstructed as C source under `data/drivers/`.
   `REVERSE_ENGINEERING.md` §3 explains the method; each reconstruction is
   validated against artifacts that already existed.

2. **valgrind does not exist on Apple Silicon**, and neither obvious substitute
   is complete. `memory_check` builds each program **twice** — AddressSanitizer
   for out-of-bounds/use-after-free/double-free, then a plain build under
   Apple's `leaks` for leaked allocations. Measured limitations: neither
   reports uninitialised reads (valgrind did), and `leaks` misses leaks whose
   pointer is still on the exiting stack. So `  - Valgrind was clean …` means
   "clean under ASan + `leaks`", not "valgrind-clean".

3. **Some capstone sources are flattened git symlinks** — a file whose entire
   content is `../c2prj1_cards/cards.c`. The build expands them automatically.

## Guarantees

- **Every child process has a wall-clock timeout** and is killed (SIGTERM,
  then SIGKILL). A student's infinite loop cannot hang grading.
- **Builds happen in a temp directory** that is always removed, so no `.o` file
  or executable is left in the assignment folder.
- **Comparison is byte-exact on buffers**, never on decoded strings, because
  student C programs legitimately emit non-UTF-8 bytes.
- **Cases run serially** — peak memory stays flat.
- **Output capture is bounded**, so a runaway `printf` loop cannot exhaust RAM.
- **`init` refuses to create a workspace inside the package**, and never
  overwrites an existing assignment directory.
- **Never fakes a pass.** Where something cannot be checked faithfully, the
  config says `unsupported` and names the missing artifact.

---

## Development

```bash
bun test tests/          # 56 unit + end-to-end tests
bun run check            # validate every config
bunx tsc --noEmit        # typecheck
bun run build            # optional: single-file binary at dist/grade

./tools/check.sh         # all of the above, plus corpus integrity
./tools/check.sh --full  # also re-grade all 42 against the 2024 transcripts
./tools/demo.sh 05_squares
```

| path | what |
|---|---|
| `data/configs/` | one JSON per assignment (the index) |
| `data/templates/` | the skeleton each assignment starts from |
| `data/drivers/` | reconstructed grader drivers, in C |
| `data/reference/` | source-only answer key |
| `re/` | reverse-engineering evidence: ELF reports, original transcripts |
| `src/` | the tool |
| `tools/` | RE and verification tooling |
| `upstream/` | the 42 completed snapshot directories (gitignored, binary-heavy) |

## Licence

MIT — see `LICENSE`. The assignment texts and reference solutions originate
from the Duke University Coursera course and are included for study.
