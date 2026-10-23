# RFC 0004: OpenDAP Protocol, version 0

- **Status:** Accepted (2026-10-23; decided by Claude under the maintainers' delegation, with the keystore decided by the maintainers). Bench checks remain, listed at the end
- **Date:** 2026-10-23
- **Affected repositories:** system (interfaces, services), blackbird-os (USB gadget, links, discovery, key storage, packages), desktop (client and background service), toolkit-cli (client), toolkit-verify (conformance suite), t1 (product requirements, LED and UI states), board (USB, radios, key holder options), ui-components (user-facing states)

## Summary

The **OpenDAP Protocol** is how a listener's devices work together: the T1, other OpenDAP devices, OpenDAP Desktop and the `opendap` CLI, and later a phone app. It carries:

- playback state in real time;
- the library and its index;
- playlists, history and audio profiles;
- files, updates, backups and diagnostics;
- pairing, between devices of the same owner or household.

It runs over IP on whatever link is present (the LAN, the T1's own access point, or USB networking), finds devices with DNS-SD, and authenticates every peer by a key recorded at pairing. On the device it exposes the same versioned `org.opendap.*` interfaces Blackbird OS already uses ([system ADR 0002](https://github.com/opendap-project/system/blob/main/docs/adr/0002-system-architecture.md)), so a feature is defined once and every surface reaches it. It never requires an account, an online service or Bluetooth. Version 0 is the first version, developed until the maintainers accept it.

## Motivation

Requirements this answers:

- **SYS-042** (must): real-time playback sync across all of the listener's devices.
- **OS-029** (must): everything synced through the protocol, down to the library index, with on-demand work on the device when no client exists.
- **DESK-011** (must): sync and connected features go through OpenDAP Desktop or the CLI.
- **SYS-043** (must) and **SYS-044** (should): the household and family members.
- **SYS-045** (must) and **SYS-047** (must): third parties as opt-in providers; nothing needs an account.
- **SYS-046** (should): multi-room.
- **T1-065** (Draft): plain USB storage by default.
- **SYS-048** (must): user-facing conventions people know.
- Proposed (Draft): SYS-049 to SYS-060, OS-030, OS-031, DESK-012 to DESK-020.

Background:

- the [ecosystem experience research](https://github.com/opendap-project/t1/blob/main/docs/research/ux/2026-10-ecosystem-experience.md): jobs, journeys, principles E1 to E20;
- the [protocol research](https://github.com/opendap-project/system/blob/main/docs/research/2026-10-device-ecosystem-protocol.md): the landscape, where no existing protocol covers the whole job;
- the [case studies](https://github.com/opendap-project/t1/blob/main/docs/research/ux/2026-10-ecosystem-case-studies.md): security, update and longevity failures;
- the [user-facing conventions spec](https://github.com/opendap-project/system/blob/main/docs/specs/user-facing-conventions.md).

Without one protocol, each surface would talk to the device its own way. Features would be duplicated per surface, and the KDE Connect lesson would repeat: CVE-2025-66270 is one ambiguous handshake shared by five implementations ([KDE advisory](https://kde.org/info/security/advisory-20251128-1.txt)).

## Proposal

### 1. Principles

1. **Local-first.** Every device holds a full replica of what it needs and works alone. The protocol exchanges changes; it never asks permission to act (E1).
2. **Never on the critical path.** No protocol work delays input, audio or the UI (SYS-036, SYS-037, T1-057). Heavy work waits for idle or charging on the T1.
3. **One owner, many devices.** Trust and sync are scoped to the owner's paired devices, extended to a household's members (SYS-043). No account (SYS-047).
4. **The device's own interfaces, carried off the device.** Remote peers call the same `org.opendap.*` interfaces as on-device clients, under per-peer permissions.
5. **Strict and versioned inside; standards at the edges.** OpenDAP peers negotiate versions and capabilities and fail loudly ([RFC 9413](https://www.rfc-editor.org/rfc/rfc9413)). Devices that know nothing about OpenDAP use standards (§9).
6. **Capabilities, not models.** Each peer advertises what it can do; what is missing is disabled, never a reason to refuse the connection (SYS-040, DESK-020).
7. **Bounded.** No message grows with the library; exchanges are incremental and resumable (T1-046, T1-047).

### 2. Links

| Link | Role | Notes |
|---|---|---|
| **Same LAN** | Primary | IP on Wi-Fi or Ethernet (through a HAT or dock) |
| **USB networking** | Primary with a cable | CDC NCM. The T1's device-mode port is USB 2.0 High-Speed only on the RK3566 (480 Mb/s shared by every USB function; `board` ADR 0003). Inbox NCM drivers: Windows 11 (`UsbNcm.sys`, [Microsoft](https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/supported-usb-classes)), macOS (`AppleUSBNCM.kext`, observed on macOS 27.0.1; first version **UNVERIFIED**), Linux (`cdc_ncm`). Windows 10 has no inbox NCM driver: USB networking there is unsupported or needs a driver |
| **The T1's access point** | No network around | The T1 runs a SoftAP for one peer; station and access point concurrently on one channel are supported by the mainline IW612 driver (`nxpwifi`, kernel 7.3) |
| **Wi-Fi peer-to-peer** | Reserved | Not available: `nxpwifi` advertises no P2P modes (7.3), and macOS offers no public Wi-Fi Direct or Wi-Fi Aware. A slot is kept, enabled when drivers expose it (SYS-040) |
| **Bluetooth LE** | Optional bootstrap | Presence and wake-up for discovery, and passing pairing data; never bulk data and never required. Whether a product enables it is a product decision |
| **A relay the owner chooses** | Optional reach | End-to-end encrypted traffic between the owner's devices across the internet; the relay sees ciphertext only. Any relay, including one the owner runs |
| **The card** | Offline carrier | A card prepared by OpenDAP Desktop carries files and state, applied when inserted |

The system chooses the link; the listener sees one state with the link as detail (conventions spec §4).

### 3. USB composite

The main USB-C port in device mode presents **one configuration** with IAD, so it installs with no driver on Windows and macOS. Windows loads its composite driver only for a single configuration ([Microsoft](https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/enumeration-of-the-composite-parent-device)).

| Function | Purpose | Default |
|---|---|---|
| NCM | The protocol, SSH for the CLI | On |
| UAC2 | The T1 as a USB DAC (T1-054) | When the listener selects USB DAC |
| ACM | The serial console (BRD-039) | On |
| Mass storage | Plain files for hosts without a client (T1-065) | On |

- **UAC2** (`f_uac2`): up to 10 discrete rates, containers up to 32 bits, asynchronous with explicit feedback steered from the DAC clock. At High Speed one 1024-byte microframe carries stereo well beyond 384 kHz. DoP passes unchanged; native DSD needs kernel work, carried as a patch until upstream (OS-023).
- **Mass storage:** the kernel forbids touching an exported backing store from the device ([kernel docs](https://docs.kernel.org/usb/mass-storage.html)). So the T1 never exports its live library filesystem read-write. The T1 **hands over its music volume** (the card, or a music partition of the internal storage) to the host exclusively:
  - while it is handed over, the T1 keeps playing what its cache levels already hold (OS-022) and shows "Connected to a computer" with an **Eject** action;
  - on release, it re-indexes the changes incrementally;
  - when OpenDAP Desktop or the CLI is present, they use NCM instead, and the volume is not handed over.

  Decided in [blackbird-os ADR 0013](https://github.com/opendap-project/blackbird-os/blob/main/docs/adr/0013-usb-device-mode-and-keystore.md): the card if present, otherwise a dedicated music partition.
- **Endpoints:** about 12 non-control endpoints for the four functions together; the RK3566 DWC3 endpoint count is **UNVERIFIED** and must be measured.

### 4. Discovery

- **DNS-SD over mDNS** on every IP link (LAN, SoftAP, NCM), with a service type in the `odap` namespace (for example `_odap._tcp`). It is native on macOS (`dnssd`, `NWBrowser`), Windows 10 and later (`DnsServiceBrowse`, [Microsoft](https://learn.microsoft.com/en-us/windows/win32/api/windns/nf-windns-dnsservicebrowse)) and Linux (Avahi, systemd-resolved).
- **Nothing personal is broadcast.** Advertisements carry no owner identifier, hashed or not (AirDrop's hashed identifiers were reversed with rainbow tables, case studies §4.2). A paired peer recognises its devices by a value only paired peers can compute; everyone else sees a generic OpenDAP device.
- **BLE advertising is policy-driven:** on demand or at an interval the listener can change. Its standby cost is not in the module datasheets and must be measured before defaults are set.

### 5. Identity and pairing

- **Identity is a key.** Each device generates a key pair. Its identity is the key's fingerprint, recorded by each peer at pairing; a claimed identifier is never trusted. Anything exchanged before the secure channel is re-verified inside it (KDE Connect CVE-2025-66270).
- **Pairing** follows the conventions spec §3:
  - the T1 shows a QR code and a short code that expires;
  - the other side scans or types it;
  - the listener confirms on the T1.

  The code feeds a password-authenticated key exchange, so a short code gives strong keys and an eavesdropper learns nothing:
  - the exchange is **SPAKE2** ([RFC 9382](https://www.rfc-editor.org/rfc/rfc9382));
  - the code is 8 characters of Crockford base32 shown as `XXXX-XXXX` (40 bits, no ambiguous letters);
  - each code is single-use, expires after 10 minutes, and allows few attempts before a new code is needed.
- **Per-peer keys and revocation.** Every pairing has its own keys; removing a device from any surface revokes it everywhere at the next sync.
- **Keys always go through a keystore** (OS-032, the maintainers' decision): no service handles key files.
  - The keystore is backed by a TEE whenever the platform offers one.
  - The RK3566 has no mainline OP-TEE and its OTP is read-only to Linux ([OP-TEE platforms](https://optee.readthedocs.io/en/latest/general/platforms.html)), so the keystore starts on protected filesystem storage ([blackbird-os ADR 0013](https://github.com/opendap-project/blackbird-os/blob/main/docs/adr/0013-usb-device-mode-and-keystore.md)).
  - It moves to hardware without changing its users: a key holder on the board (BRD-059, a discrete TPM 2.0 proposed), or an OP-TEE port.
  - Pairing stays safe even if a key is extracted, through per-peer keys and revocation.
- **Households.** A household is a set of owners' devices that share the collection. Members (SYS-044) get their own personal state. Permissions follow per member and per device.

### 6. Session

1. Mutually authenticated, encrypted channel with the pinned keys.
2. Exchange of protocol version and capabilities: the interfaces each side implements, with their versions and hashes (as ADR 0002 already defines them).
3. Permissions checked per call, per peer and per member.
4. Typed errors, the same dotted codes as on the device and in the CLI.

### 7. Services

The protocol carries these interfaces. Names are indicative; their definitions live in `system`.

| Service | What it carries | Requirements |
|---|---|---|
| **Player** | Live playback state on every device (track, position, queue, state), transport control and takeover. **One active renderer, many controllers**: the renderer fetches media itself, bit-perfect from its source, and broadcasts state to every peer (the Spotify Connect model, locally) | SYS-042, E13 |
| **Library** | The collection and its index as versioned, content-addressed data: entries, metadata, artwork at each device's display size, statistics. A peer that has computed them sends them; a device with no peer computes them on demand | OS-029, T1-046, T1-052 |
| **State** | The owner's state: playlists, ratings, events (plays and skips, each with its own time), bookmarks and resume points, audio profiles, device selections, settings. Convergent: concurrent changes merge, real conflicts keep both versions and are reported once | SYS-052, SYS-053, SYS-043 |
| **Transfer** | Files by device rules: resumable, verified by content, never by timestamp | DESK-013, SYS-049 |
| **Device** | Updates staged for a device, versioned backups and restore, diagnostics and logs the owner chooses to send, storage and battery state | DESK-017, OS-030 |
| **Devices** | Paired devices, household members, revocation, links in use | SYS-051, SYS-056 |
| **Notifications** | Between devices, by kind, off until the listener turns them on | Conventions spec §8 |
| **Measurement** | Signal path, meters, captures for engineering sessions | SYS-041 |

**Destructive calls** (erase, reset, remove all music) require confirmation on the target device itself (SYS-050).

### 8. Sync model

- **Changes, not snapshots.** Each device keeps a log of its changes, and peers exchange what the other lacks.
- **History is bounded.** It is compacted once every peer has acknowledged it, so a device unpaired for months resyncs from a snapshot plus newer changes.
- **The index is data.** Its format is versioned with the protocol, so a newer desktop can prepare it for an older T1 within the versions both support.
- **Deletions** pass through Recently Deleted for 30 days on every device (SYS-050).
- **State is visible.** Every exchange reports synced, syncing, waiting, paused or can't sync, with the reason (SYS-051).

### 9. At the edges

Devices without OpenDAP software use standards. None of them is part of the protocol:

| The other side | Standard |
|---|---|
| A computer with no client | Mass storage (§3), UAC2 |
| Headphones and cars | A2DP, AVRCP (target, metadata, browsing, cover art); LE Audio media and volume control (BlueZ 5.87 marks them experimental) |
| A phone with no OpenDAP app | Bluetooth source and sink; UPnP/DLNA renderer and server |
| Home audio | UPnP/DLNA; Sendspin (release candidate, Community Specification License 1.0) as an interoperation target; Snapcast as a client, which is not bit-perfect (it corrects sync by dropping or duplicating samples) |
| Multi-room among OpenDAP devices | SYS-046 (should); the clock and sync scheme is specified later, bit-perfect where the path allows |
| Proprietary ecosystems | Optional adapters, never in the image by default (OS-027, SYS-045) |

LE Audio's media control places the server on the source. It covers the T1 with headphones, but not a phone controlling the T1; phone support for that role is **UNVERIFIED**. So the phone app uses this protocol, not LE Audio, to control the T1.

### 10. What the protocol never requires

- **An account or an online service** (SYS-047, SYS-058).
- **Bluetooth.** Every core function works over IP links.
- **A matching version.** Peers negotiate; an old device and a new one always connect with what both support.

### 11. Names

In prose the protocol is the **OpenDAP Protocol**. Technical identifiers use the short form `odap`:

- the DNS-SD service type;
- a URI scheme for links and QR codes;
- crate and package names;
- interface namespaces where the device's `org.opendap.*` names do not apply.

The exact identifiers are fixed in the specification.

### 12. Deliverables, per repository

| Repository | Deliverable |
|---|---|
| `system` | Protocol specification: handshake, pairing, session, encoding, interfaces, sync model, with test vectors; interface definitions |
| `blackbird-os` | USB composite gadget; NCM, SoftAP, DNS-SD; mass-storage handover; key storage port; packages |
| `desktop` | Client and background service (DESK-006 to DESK-008) on the shared core |
| `toolkit-cli` | The same client in the CLI (DESK-004, TK-009) |
| `toolkit-verify` | The conformance suite every implementation passes, OpenDAP's first |
| `t1` | LED states (pairing, transferring, updating) for T1-044; UI states |
| `board` | USB endpoint count on the RK3566; measured BLE standby current; key holder option |
| `ui-components` | The user-facing states of the conventions spec |

## Alternatives

- **KDE Connect's protocol.** Open, with several implementations, pairing, media control and files. Not chosen as the base: it has no library or index sync, no hi-res audio, no device management, and its handshake needed a coordinated fix in 2025. It remains a precedent and possibly a bridge.
- **Sendspin for everything.** Its licence, Noise-based security and versioned roles fit well. But it covers playback, control and metadata, not library, state or device management, and it is still a release candidate. It stays the interoperation target for playback (§9).
- **UPnP AV / DLNA as the core.** It has a large installed base. Not chosen: weak security, no sync, and SOAP/XML verbosity. It stays at the edges.
- **Syncthing's protocol for all sync.** Excellent for files. Not chosen as the whole: it has no music semantics, playback state or device services. Its identity model (certificate fingerprints) and block-level transfer are adopted as patterns.
- **Matter.** Not an audio transport, and certification is for members. Possible later as a home integration at the edges.
- **Separate APIs per surface** (a desktop API, a phone API). Rejected: features would be duplicated and drift; the one-core rule (DESK-004) points to one protocol.

## Decisions on the open points (2026-10-23)

1. **Encoding.**
   - Control traffic carries the device's interfaces as ADR 0002 defines them: the same calls, types and typed errors, over the authenticated channel.
   - Bulk transfer and media streams use binary framing.
   - A compact encoding for control replaces JSON only if measurements on the RK3566 show it is needed: the same gate ADR 0002 set for the bus.
2. **Pairing:** SPAKE2 (RFC 9382) with an 8-character code (§5).
3. **Mass storage handover:** blackbird-os ADR 0013.
4. **Keys:** a keystore always, a TEE when available (OS-032).
5. **Bluetooth** stays optional for the protocol: every core function works over IP links, and Bluetooth only adds headphones, cars and an optional bootstrap.
6. **Licence of the specification:**
   - the text under CC-BY-SA-4.0, like all OpenDAP documentation;
   - interface definitions and schemas under MPL-2.0, like the project's other schemas;
   - proposed, to be confirmed by the maintainers because it is a legal commitment: a royalty-free patent non-assertion for conforming implementations, so anyone can implement it.

## Bench checks (implementation)

1. Driverless enumeration of the full composite on Windows 11 and macOS, and the RK3566 DWC3 endpoint count.
2. UAC2 isochronous behaviour on the RK3566 under asynchronous feedback, and host bit-exactness for DoP.
3. BLE advertising and sleep current of the Murata Type 2EL at candidate intervals, before discovery defaults are set.
4. Control-plane encoding cost on the RK3566 (decision 1).
