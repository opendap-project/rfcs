# RFCs

Proposals for changes that span more than one OpenDAP repository, or that change an interface others build against: the HAT Case Standard, the faceplate interface, the component export contract, the bus API.

A change inside one repository does not need an RFC: it goes straight to that repository's requirements or ADRs.

## Process

1. Copy `0000-template.md` to `text/NNNN-short-title.md` with the next free number, status **Draft**, and open a pull request.
2. Every affected repository's domain reviews it (electronics, mechanics, firmware, software, OS).
3. The maintainers accepts or rejects it. An accepted RFC is merged with status **Accepted**.
4. Each affected repository records the outcome in its own `docs/` (requirements, and an ADR where a decision was taken), linking back to the RFC. The RFC becomes **Implemented** when all of them have.

RFCs that only concern private services live in the private `workspace` repository under `rfcs/`, with the same template.

## License

All content is **Creative Commons Attribution-ShareAlike 4.0** (`CC-BY-SA-4.0`), following the [REUSE](https://reuse.software) layout.

Copyright © 2026 The OpenDAP Project.
