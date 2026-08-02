# Alloy language specification

This repository holds the source of truth for the Alloy language:

- `LANGUAGE_SPEC.md` - the language specification.
- `bin/alloyc.exe` - the latest working build of the alloyc compiler (Windows, ReleaseSafe).
- `bin/std/` - the standard library sources the compiler resolves next to its executable.

It is consumed as a git submodule by the compiler repositories (the Zig
implementation and the self-hosted mirror), so both track the exact spec
revision they implement and the mirror can bootstrap from the packaged
compiler.
