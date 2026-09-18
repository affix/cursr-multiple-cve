# Cursr 1.7.3 Security Disclosure

Full disclosure of ten vulnerabilities in **Cursr**, a peer-to-peer software KVM by Bitgapp that shares a single mouse and keyboard across multiple machines on a local network.

All ten issues were reported to the vendor on 2 March 2026 and are published here following coordinated disclosure. Fixes first shipped in **1.7.4-prerelease.2** on 2 August 2026 and are carried by the 1.7.4 release.

## Affected software

| | |
|---|---|
| Product | Cursr |
| Vendor | Bitgapp |
| Homepage | https://cursr.app/ |
| Affected versions | 1.7.3, 1.7.4-prerelease.1 |
| Fixed in | 1.7.4 (first available in 1.7.4-prerelease.2) |
| Attack vector | Adjacent network (local LAN segment) |

Analysis was performed against the beautified `main.beautified.js` from the packaged Electron application. Line references in the advisories point at that file for the version named in each code block. Two advisories (CVE-2026-38686, CVE-2026-38692) additionally show the unchanged handler as it appeared in 1.7.4-prerelease.1, before the fixes landed in prerelease.2.

## Vulnerabilities

| CVE | CWE | Title |
|---|---|---|
| [CVE-2026-38685](advisories/CVE-2026-38685-unauthenticated-display-metadata-disclosure.md) | CWE-306 | Unauthenticated Display Metadata Disclosure |
| [CVE-2026-38686](advisories/CVE-2026-38686-unauthenticated-symmetric-key-disclosure.md) | CWE-306 | Unauthenticated Symmetric Key Disclosure |
| [CVE-2026-38687](advisories/CVE-2026-38687-udp-multicast-information-disclosure.md) | CWE-319 | UDP Multicast Information Disclosure |
| [CVE-2026-38688](advisories/CVE-2026-38688-unauthenticated-peer-ejection.md) | CWE-306 | Unauthenticated Peer Ejection |
| [CVE-2026-38689](advisories/CVE-2026-38689-udp-unpair-spoofing.md) | CWE-290 | UDP Unpair Spoofing |
| [CVE-2026-38690](advisories/CVE-2026-38690-unauthenticated-peer-injection.md) | CWE-306 | Unauthenticated Peer Injection |
| [CVE-2026-38691](advisories/CVE-2026-38691-udp-renounce-spoofing.md) | CWE-290 | UDP Renounce Spoofing |
| [CVE-2026-38692](advisories/CVE-2026-38692-unauthenticated-tls-certificate-disclosure.md) | CWE-306 | Unauthenticated TLS Certificate Disclosure |
| [CVE-2026-38693](advisories/CVE-2026-38693-unauthenticated-election-manipulation.md) | CWE-306 | Unauthenticated Election Manipulation |
| [CVE-2026-38694](advisories/CVE-2026-38694-man-in-the-middle-via-disabled-tls-certificate-verification.md) | CWE-295 | Man-in-the-Middle via Disabled TLS Certificate Verification |

## Overview

Cursr's peer-to-peer layer uses three channels, none of which authenticate their peers:

- **HTTPS on port 4440** — an Express server exposing the cluster control API. Every route relevant to this disclosure is registered with no authentication middleware. Unauthenticated `GET` requests retrieve the AES symmetric key (CVE-2026-38686), the TLS certificate (CVE-2026-38692) and licence metadata (CVE-2026-38685). Unauthenticated `POST` requests eject peers (CVE-2026-38688), inject rogue peers (CVE-2026-38690) and drive leader election (CVE-2026-38693).
- **UDP on port 28777, multicast group 224.0.0.236** — device discovery, in cleartext JSON with no signing. It broadcasts instance UUIDs, machine IDs, hostnames, IP ranges, OS and licence status to anyone listening (CVE-2026-38687), and it acts on inbound `renounce` (CVE-2026-38691) and `unpair` (CVE-2026-38689) messages on the strength of a caller-supplied `senderId` alone.
- **gRPC on port 4441** — the input event stream, carrying payloads encrypted with the symmetric key that port 4440 will hand to any caller.

Validation on inbound UDP amounts to four checks: the `tag` field equals `cursr`, the `senderId` is not the receiver's own, the source IP is in a private range, and the broadcast address matches. There is no HMAC, signature, nonce or shared secret anywhere in the protocol.

The individual weaknesses compose. CVE-2026-38687 supplies the instance and machine IDs that the spoofing and injection issues need as parameters, which makes passive sniffing the first step in nearly every chain. CVE-2026-38686 and CVE-2026-38694 combine into full plaintext access: certificate verification is disabled across every HTTPS client in the application (`rejectUnauthorized: false`), so a network-path attacker can intercept the control API, and the symmetric key retrieved from `/key` decrypts the gRPC and UDP payloads that interception exposes. The practical result is read and write access to a live KVM stream: keystrokes, mouse events and clipboard contents.

## Repository layout

```
advisories/     One Markdown advisory per CVE
README.md       This file
```

Each advisory covers summary, affected versions, CWE, impact, technical details with annotated decompiled source, reproduction steps and remediation guidance.

No exploit tooling is published in this repository. The advisories carry enough detail to reproduce each issue, including `curl` invocations for the unauthenticated HTTP endpoints and the full datagram schema for the UDP issues.

## Disclosure timeline

| Date | Event |
|---|---|
| 2 March 2026 | Initial discovery of the vulnerabilities |
| 2 March 2026 | Report sent to Bitgapp's Ragauskl |
| 2 March 2026 | Discussion with Bitgapp regarding the vulnerabilities |
| 13 March 2026 | Bitgapp responded with further vulnerability details and a remediation timeline |
| 14 April 2026 | CVE requests submitted |
| 24 April 2026 | CVEs reserved |
| 2 August 2026 | Release 1.7.4-prerelease.2 published, addressing the vulnerabilities |
| 18 September 2026 | Public disclosure |

Bitgapp were responsive throughout and engaged constructively on both the technical detail and the remediation plan.

## Remediation summary

Users should update to 1.7.4 or later. Anyone still on 1.7.3 or on the 1.7.4-prerelease.1 build should treat Cursr as safe to run only on network segments where every host is trusted, since all ten issues are reachable by any device on the same LAN.

The per-issue guidance in the advisories reduces to four changes:

- Authenticate the port 4440 control API, binding peer identity to material exchanged during pairing rather than to a self-asserted instance ID.
- Stop serving cryptographic key material and TLS certificates to unauthenticated callers.
- Authenticate UDP discovery messages with an HMAC over a pairing-derived secret, add replay protection, and cut the cleartext broadcast down to the minimum needed to establish a connection.
- Enable certificate verification and pin peer certificates at pairing time.

## Credit

Discovered and reported by Keiran "Affix" Smith.
