# Alloy language specification

This repository holds the source of truth for the Alloy language - everything
that must stay consistent between compiler implementations:

- `LANGUAGE_SPEC.md` - the language specification.
- `std/` - the standard library sources.
- `samples/` - the end-to-end probe programs every implementation must run.
- `bin/alloyc.exe` - the latest working build of the alloyc compiler (Windows, ReleaseSafe).

It is consumed as a git submodule by the compiler repositories (the Zig
implementation and the self-hosted mirror), so both track the exact spec
revision they implement and the mirror can bootstrap from the packaged
compiler.

The packaged `bin/alloyc.exe` resolves `std::` imports through the search
order of specification section 5.4 (the current directory, the compiler
executable's directory, then `$ALLOY_STDLIB`). The standard library lives at
the repository root rather than next to the executable, so run it from this
directory or with `ALLOY_STDLIB` pointing here.
