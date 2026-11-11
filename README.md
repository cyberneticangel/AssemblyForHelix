# AssemblyForHelix

Shows the assembly the compiler generates for the function under your cursor,
or for the selected lines, in [Helix](https://helix-editor.com).

Each source line is shown as a comment, with the instructions it produced
below it. On x86 the output uses Intel syntax, and C++, Rust and Swift symbol
names are demangled.

![asm-view in Helix: assembly in a split and in popups for C, Rust and Go](assets/demo.gif)

([MP4 version](assets/demo.mp4))

```
; lib.c:3-8  -O2  (c)

sum:
  ; 3 | int sum(int *a, int n) {
  ; 5 | for (int i = 0; i < n; i++)
        test esi, esi
        jle .L4
  ; 6 | s += sq(a[i]);
        mov eax, DWORD PTR [rdi]
  ; 1 | static int sq(int x) { return x * x; }
        imul eax, eax
  ...
```

## Supported languages

| Language       | Extensions                          | Needs on `PATH`                 |
| -------------- | ----------------------------------- | ------------------------------- |
| C              | `.c`                                | `cc` (gcc or clang)             |
| C++            | `.cc .cpp .cxx .c++ .C`             | `c++`                           |
| Objective-C/++ | `.m .mm`                            | `clang` / `clang++`             |
| Fortran        | `.f .f90 .f95 .f03 .f08 .F .F90`    | `gfortran`                      |
| Rust           | `.rs`                               | `rustc`, plus `cargo` in a crate |
| Zig            | `.zig`                              | `zig`                           |
| Go             | `.go`                               | `go`                            |
| D              | `.d`                                | `ldc2`                          |
| Swift          | `.swift`                            | `swiftc`                        |

You also need `python3`. `c++filt` (from binutils) is optional and demangles
C++ and Rust names. `swift-demangle` is also optional and does the same for
Swift.

Notes:

- **C / C++ / Objective-C:** if a `compile_commands.json` exists (next to the
  file, in a parent directory, or in `build/`, `out/` or `builddir/` under the
  project root), the file is compiled with the flags listed there.
- **Rust:** inside a Cargo project, the crate is built with `cargo rustc`. At
  `-O1` and above, a plain `pub fn` in a library produces no assembly until
  something calls it. Mark it `#[no_mangle]` or use `-O0`.
- **Zig:** only code reachable from an `export fn` (or `main`) is compiled.
- **Go:** the whole package is built. `-O0` passes `-N -l`, and every other
  level uses the normal optimised build. The output is Go's Plan 9 assembly,
  with each instruction's PC offset in front of it.

## Setup

### 1. Install the `asm-view` script

```sh
git clone https://github.com/cyberneticangel/AssemblyForHelix ~/.local/share/AssemblyForHelix
mkdir -p ~/.local/bin
ln -s ~/.local/share/AssemblyForHelix/bin/asm-view ~/.local/bin/asm-view
```

Make sure `~/.local/bin` is on your `PATH`, then check that the script runs:

```sh
asm-view --help
```

### 2. Add keybindings to Helix

This needs Helix 25.07 or newer, which provides the
`%{selection_line_start}` and `%{selection_line_end}` expansions. Add the
following to `~/.config/helix/config.toml`:

```toml
# in a source file:   \a  asm in a popup       \v  asm in a vertical split
# in that split:      \0 \1 \2 \3 \s  re-run at that -O level
[keys.normal."\\"]
a = ":sh asm-view %{buffer_name} %{selection_line_start} %{selection_line_end}"
v = ":vsplit %sh{asm-view --tmp %{buffer_name} %{selection_line_start} %{selection_line_end}}"
0 = ":open %sh{asm-view --tmp --rerun %{buffer_name} -O 0}"
1 = ":open %sh{asm-view --tmp --rerun %{buffer_name} -O 1}"
2 = ":open %sh{asm-view --tmp --rerun %{buffer_name} -O 2}"
3 = ":open %sh{asm-view --tmp --rerun %{buffer_name} -O 3}"
s = ":open %sh{asm-view --tmp --rerun %{buffer_name} -O s}"

[keys.select."\\"]
a = ":sh asm-view %{buffer_name} %{selection_line_start} %{selection_line_end}"
v = ":vsplit %sh{asm-view --tmp %{buffer_name} %{selection_line_start} %{selection_line_end}}"
```

Restart Helix, or run `:config-reload`. You can change `\` to any key you
don't already use.

## Use

- **Function under the cursor:** put the cursor anywhere in a function and
  press `\a` (popup) or `\v` (split). When the selection covers a single line,
  the whole function around it is shown, including code inlined into it.
- **Selected lines:** select several lines (for example with `x`, pressed more
  than once), then press `\a` or `\v`. Only the code for those lines is shown.
- **Other optimisation levels:** in the split, press `\0`, `\1`, `\2`, `\3` or
  `\s`. The same function or lines are compiled again at that level and shown
  in the same split.
- **Close the split** with `Ctrl-w q`.

The compiler reads the file on disk, so save first (`:w`). If the file
doesn't compile, the compiler's error is shown where the assembly would be.
Each split run writes a new file under `/tmp/asm-view-$UID/`. Files older
than a day are deleted.

## Command line

`asm-view` works without Helix:

```
asm-view FILE [LINE [END]] [-O 0|1|2|3|s] [--exact] [--lang LANG] [-o OUT | --tmp] [--rerun]
```

- With one `LINE` (or with `LINE` equal to `END`), the range grows to the
  function that contains it. `--exact` turns this off.
- `-O` sets the optimisation level. The default is `2`.
- `--lang` sets the language when the file extension doesn't give it.
- `-o OUT` writes the output, or the compiler error, to `OUT` and prints the
  path. `--tmp` does the same with a new file each time. Both always exit with
  status 0.
- If `FILE` is an earlier `--tmp` output, asm-view compiles the same source
  and lines again, using the new `-O` if you give one. With `--rerun`,
  asm-view fails if `FILE` isn't such an output.
