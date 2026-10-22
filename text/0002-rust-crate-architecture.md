# RFC 0002: Rust crate architecture standard

- **Status:** Accepted (2026-10-22, the maintainers' direction, written by Claude)
- **Date:** 2026-10-22
- **Affected repositories:** system, opendap-os, toolkit-cli (and the other toolkit repositories that ship Rust), desktop

## Summary

Every OpenDAP Rust package follows one architecture: crates first, layered (domain model, application core with ports, adapters, interfaces), one shared domain model per context, platform-agnostic core, so that everything the `opendap` CLI can do is shared with OpenDAP Desktop and other surfaces. The standard is [system/docs/specs/rust-crate-architecture.md](https://github.com/opendap-project/system/blob/main/docs/specs/rust-crate-architecture.md).

## Motivation

The maintainers asked for modular, crates-first Rust with clean abstraction layers and a domain model, to maximise reuse and portability (for example, OpenDAP Desktop sharing everything the CLI does). TK-009 and DESK-004 already require one domain model across tools.

## Proposal

- Adopt the standard for all Rust code.
- toolkit-cli is refactored to it first: logic now in the `opendap-cli` binary (virtual T1 control, SSH execution, QMP) moves into `opendap-core` and adapter crates; the binary keeps only argument parsing and rendering.
- OpenDAP Desktop consumes the same crates.

## Alternatives

- One crate per repository: simpler at first, but forces the desktop app to duplicate or depend on CLI internals.

## Open questions

- When the host crates move to their own repository (a third consumer is the trigger in the standard).
