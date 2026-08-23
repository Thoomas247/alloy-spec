# Alloy language specification

This repository holds the source of truth for the Alloy language - everything
that must stay consistent between compiler implementations:

- `LANGUAGE_SPEC.md` - the language specification.
- `std/` - the standard library sources.
- `samples/` - the end-to-end probe programs every implementation must run.
- `alloyc.exe` - the latest working build of the alloyc compiler (Windows, ReleaseSafe).

It is consumed as a git submodule by the compiler repositories (the Zig
implementation and the self-hosted mirror), so both track the exact spec
revision they implement and the mirror can bootstrap from the packaged
compiler.

The packaged `alloyc.exe` resolves `std::` imports through the search order of
specification section 6.4 - the entry module's directory, the compiler
executable's directory, then `$ALLOY_STDLIB`. Resolution never depends on the
shell's working directory.

## Compiler status

The spec is the source of truth; `alloyc.exe` is a snapshot that trails it.
The packaged build predates spec revision 2026-08-23 and does not yet parse
interface-declared receivers (`fn next(self: &var) -> Option<&T>;`, section
5.2). Because `std/iterable.alloy` uses that form, the packaged binary cannot
currently build anything that reaches `Iterable`/`Iterator`:

- builds: `samples/constructor_probe.alloy`, `samples/macro_probe.alloy`,
  `samples/run_demo.alloy`
- does not build: `samples/parse_demo.alloy`, `samples/stdlib_probe.alloy`

Both fail with `expected a parameter name, found ':'`. The next compiler drop
clears it; no source change is needed here.
