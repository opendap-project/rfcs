# RFC 0005: Rename blackbird-ui to Blackbird UI

- **Status:** Accepted (2026-10-23).
- **Date:** 2026-10-23
- **Affected repositories:**
  - blackbird-ui (becomes blackbird-ui);
  - workspace;
  - system, t1, rfcs, blackbird-os, case;
  - toolkit-verify, toolkit-hardware, blender-lib.

## Summary

The repository that holds the Engineered Matter design system, its tokens, icons, templates and components is renamed **`blackbird-ui`**: the UI of Blackbird OS, as `blackbird-os` is the OS. "Engineered Matter" stays the name of the design system it contains.

## Motivation

`blackbird-ui` described one part of the repository's content. It now holds:

- the design system specification;
- the tokens and OpenDAP Icons;
- the host's templates and the components contributed through the registry.

It is the product UI of Blackbird OS (system ADR 0005, ADR 0006), and its name should say so, next to `blackbird-os` (RFC 0003).

## Proposal

- **Repository:** `opendap-project/blackbird-ui` becomes `opendap-project/blackbird-ui`; GitHub redirects the old URL. The workspace folder `public/blackbird-ui` becomes `public/blackbird-ui`, and `repos.tsv` follows.
- **References:** every link and path to `blackbird-ui` in the affected repositories points to `blackbird-ui`. The design system keeps its title, "Engineered Matter — Design System".
- **Slint library name:** the design-system library that activities and templates import is `@blackbird-ui`. Crate names (`opendap-ui`, `opendap-*`) are unchanged, as RFC 0003 kept them.
- **Unchanged:** requirement IDs (`DS-`), token names, icon names, the licence layout.

## Alternatives

- **Keep `blackbird-ui`.** Not chosen: it undersells the repository and does not match `blackbird-os`.
- **Name it after the design system (`engineered-matter`).** Not chosen: the repository is the OS's UI, of which the design system is one part.

## Open questions

None.
