# RFCs

Proposals for changes that span more than one OpenDAP repository, or that change an interface others build against: the OpenDAP HAT Standard, the faceplate interface, the component export contract, the bus API.

A change inside one repository does not need an RFC: it goes straight to that repository's requirements or ADRs.

## Process

1. Copy `0000-template.md` to `text/NNNN-short-title.md` with the next free number, status **Draft**, and open a pull request.
2. Every affected repository's domain reviews it (electronics, mechanics, firmware, software, OS).
3. The maintainers accept or reject it. An accepted RFC is merged with status **Accepted**.
4. Each affected repository records the outcome in its own `docs/` (requirements, and an ADR where a decision was taken), linking back to the RFC; a repository without `docs/`, such as the workspace root or this one, records it in its `README.md` or `CONTRIBUTING.md`. The RFC becomes **Implemented** when all of them have; `odapkit docs rfc-status` lists, for each RFC, the affected repositories that have recorded it and those that have not.

RFCs that only concern services outside the public repositories are kept elsewhere, with the same template.

## Index

Generated from each RFC's title and status (`odapkit docs index text --readme README.md`); never edited by hand.

<!-- index:start -->
| Document | Status |
|---|---|
| [RFC 0001: Faceplate interface v0 pinout, defined by function](text/0001-faceplate-interface-v0-pinout.md) | Accepted (2026-10-22). |
| [RFC 0002: Rust crate architecture standard](text/0002-rust-crate-architecture.md) | Accepted (2026-10-22). |
| [RFC 0003: Rename OpenDAP OS to Blackbird OS](text/0003-blackbird-os.md) | Accepted (2026-10-23). |
| [RFC 0004: OpenDAP Protocol, version 0](text/0004-opendap-protocol.md) | Accepted (2026-10-23). Bench checks remain, listed at the end. |
| [RFC 0005: Rename ui-components to Blackbird UI](text/0005-blackbird-ui.md) | Accepted (2026-10-23). |
| [RFC 0006: Development workflow: build on the host, run on the device](text/0006-development-workflow.md) | Accepted (2026-10-23). |
| [RFC 0007: Virtual devices in their own repository](text/0007-virtual-devices.md) | Accepted (2026-10-23). |
| [RFC 0008: Repositories by audience and domain](text/0008-repository-restructure.md) | Accepted (2026-10-23). |
| [RFC 0009: odapkit, the maintainers' command, and its boundary with opendap](text/0009-odapkit.md) | Accepted (2026-10-23). |
| [RFC 0010: Quality levels Q0–Q4, incremental and ratcheted](text/0010-quality-levels.md) | Accepted (2026-10-24). |
| [RFC 0011: Feature blocks and hardware variants](text/0011-feature-blocks-and-variants.md) | Accepted (2026-10-24). |
| [RFC 0012: The composer and the OS Shell](text/0012-composer-and-os-shell.md) | Accepted (2026-10-24). |
| [RFC 0013: Faceplate interface v1](text/0013-faceplate-interface-v1.md) | Accepted (2026-10-24). |
| [RFC 0014: HAT interface v2: HATs are USB only](text/0014-hat-interface-v2-usb-only.md) | Accepted (2026-10-25), with [hat-standard ADR 0006](https://github.com/opendap-project/hat-standard/blob/main/docs/adr/0006-hats-are-usb-only.md). Recorded in hat-standard, board and hat-camera; the other repository changes below are listed, not applied. |
| [RFC 0015: HAT connector: pogo field, Mill-Max 0965, and the USB-C port HAT](text/0015-hat-connector-pogo-field-and-usb-c-port-hat.md) | Accepted (2026-10-25). Applied in hat-standard, board and t1; the case changes are listed, not applied. |
| [RFC 0016: Outdoor HAT repository](text/0016-outdoor-hat.md) | Accepted (2026-10-25). The repository is created with its requirements; the changes in other repositories are listed, not applied. |
| [RFC 0017: Naming and topology: one repository per root, names from the hierarchy](text/0017-naming-and-topology.md) | Accepted (2026-10-27). The rules apply to each root at its cut (§7); until a root is cut, its repositories keep today's names. |
| [RFC 0018: The digital thread: records, owners and schemas](text/0018-digital-thread.md) | Accepted (2026-10-27). The records are applied one document at a time (§4 rule 4); the repository changes in §8 are listed, not applied. |
| [RFC 0019: Versioning and releases](text/0019-versioning-and-releases.md) | Accepted (2026-10-27). Nothing is tagged yet; the repository changes in §11 are listed, not applied. |
<!-- index:end -->

## License

All content is **Creative Commons Attribution-ShareAlike 4.0** (`CC-BY-SA-4.0`), following the [REUSE](https://reuse.software) layout.

Copyright © 2026 The OpenDAP Project.
