---
id: RFP-XXX
title: Decentralised Naming Service
tier: XL
funding: $XXXXX
status: draft
category: Infrastructure
---

# RFP-XXX — Decentralised Naming Service

> _Sections above the appendix are TBD._

---

## Appendix A — Potential Architectures

### Two-Layer Design

The research across 8 naming systems (ENS, Handshake, GNS, IPNS, D3/Doma,
Unstoppable Domains, SPACE ID, Namecoin) reveals a fundamental tension:
DHT-based systems provide privacy and chain-agnosticism but lack
human-readable names; blockchain-based systems provide human-readable
names but sacrifice privacy and chain-independence. No single existing
system solves both.

The proposed architecture addresses this with two layers:

```
┌──────────────────────────────────────────────────┐
│  Layer 2: Human-Friendly Naming                  │
│  (on LEZ, or as a custom registrar zone)         │
│                                                  │
│  - Human-readable names (e.g. alice.logos)        │
│  - Anti-squatting via fees or auctions            │
│  - Dispute resolution                            │
│  - Optional: on-chain registry for persistence   │
│  - Maps: human name → Layer 1 zone key           │
└───────────────────┬──────────────────────────────┘
                    │ resolves to
┌───────────────────▼──────────────────────────────┐
│  Layer 1: DHT-Based Self-Certifying Names        │
│  (chain-agnostic, privacy-preserving)            │
│                                                  │
│  - Name = hash of public key (self-certifying)   │
│  - Records stored in DHT with key blinding       │
│  - No blockchain dependency for resolution       │
│  - Query privacy by design                       │
│  - Multi-transport: DHT, Waku, HTTP              │
│  - Record encryption at rest                     │
└──────────────────────────────────────────────────┘
```

**Layer 1 (DHT base)** provides the core naming infrastructure. It is
entirely chain-agnostic: any keypair can publish records to the DHT, and
any client can resolve them, without interacting with any blockchain.
Privacy is a first-class property; key blinding prevents DHT peers from
enumerating zones or correlating queries. This layer draws heavily from
GNS (RFC 9498) for its privacy model and from IPNS for its transport
flexibility.

**Layer 2 (human-friendly naming)** is an optional layer that maps
human-readable names to Layer 1 zone keys. This could be implemented as:

1. **A LEZ smart contract registry:** names are registered on-chain with
   anti-squatting fees. The registry maps `alice.logos` to a Layer 1
   zone public key. Resolution then proceeds via the DHT. This combines
   the Sybil resistance of blockchain registration with the privacy of
   DHT resolution.

2. **A custom registrar zone (GNS model):** a designated zone operator
   runs a first-come-first-served registrar (or auction-based registrar)
   that maps human-readable labels to zone keys within a GNS-style zone.
   Multiple registrar zones can coexist; users choose which to trust.
   This approach is fully chain-agnostic but depends on the registrar
   operator for availability.

3. **A hybrid:** the LEZ registry provides the canonical mapping for
   high-value names, while lightweight registrar zones handle
   community-specific or application-specific namespaces.

The key insight: Layer 1 resolution never touches a blockchain. Even when
Layer 2 uses a blockchain for name registration, the actual record
lookup (address records, content hashes, service endpoints) goes through
the DHT. The blockchain only answers "what zone key does this
human-readable name point to?"

### Why two layers?

**Zooko's triangle** states that a naming system can have at most two of
three properties: human-meaningful, secure, decentralised. Existing
systems navigate this differently:

| System | Human-meaningful | Secure | Decentralised |
|--------|-----------------|--------|---------------|
| DNS    | Yes | Yes (DNSSEC) | No (ICANN) |
| ENS    | Yes | Yes | Partially (Ethereum dependency) |
| GNS    | No (petnames) | Yes | Yes |
| IPNS   | No (key hashes) | Yes | Yes |

The two-layer design separates the concerns: Layer 1 is secure and
decentralised (giving up human-meaningful names); Layer 2 adds
human-meaningful names within a chosen trust context (a blockchain, a
registrar zone, or both). Users who need maximum decentralisation can
use Layer 1 directly with petnames. Users who want convenience use
Layer 2.

---

## Appendix B — DHT-Based Name Services: IPNS vs GNS

### Overview

IPNS and GNS are the only two production DHT-based naming systems.
Both are chain-agnostic and token-free. Both use Kademlia-derived DHTs.
They differ significantly in privacy model, record format, name model,
and maturity.

### Architectural Comparison

| Dimension | IPNS | GNS |
|-----------|------|-----|
| **DHT implementation** | Amino DHT (Kademlia, libp2p) | R5N DHT (randomised Kademlia, GNUnet) |
| **Network size** | ~25,000 server nodes, ~550,000 clients (Mar 2026) | Self-described "tiny"; no published metrics |
| **Standardisation** | IPFS specs (community standard) | RFC 9498 (IETF Informational, Nov 2023) |
| **Name model** | Self-certifying: name = hash of public key | Zone-based: zone key + label hierarchy |
| **Human-readable names** | No (requires ENS or DNSLink on top) | No (petnames; FCFS registrar for discovery) |
| **Record storage key** | Hash of public key (directly linkable to zone) | Blinded key: ZKDF(zone_key, label) (unlinkable without both inputs) |
| **Record encryption** | None (plaintext in DHT) | Yes (AES-256-CTR or ChaCha20-Poly1305) |
| **Query privacy** | Weak (DHT peers see the key being queried; key is directly the public key hash) | Strong (key blinding prevents correlation; peers cannot determine which zone or label is being queried) |
| **Zone enumeration** | N/A (no zone concept; each key is independent) | Requires brute force on (zone_key, label) pairs |
| **Record expiry** | 48 hours default; configurable | Publisher-defined; expiry baked into encryption IV |
| **Republish interval** | Every 4 hours (mandatory) | ZONEMASTER daemon (configurable) |
| **Record format** | IPNS Record (protobuf): value, sequence, validity, TTL, signatureV2 | GNS Resource Record (binary): type, flags, expiry, data, key, value |
| **Supported record types** | Single pointer (CID or path) | Multiple per zone: PKEY, EDKEY, GNS2DNS, BOX, SBOX, VPN, REDIRECT, NICK, PHONE, DID_DOCUMENT, and all DNS types |
| **Delegation** | None (flat namespace) | Hierarchical via PKEY/EDKEY records (arbitrary depth) |
| **DNS compatibility** | Via DNSLink (TXT record in DNS pointing to IPNS name) | Via dns2gns proxy, NSS plugin, GNS2DNS delegation record, SOCKS5 proxy |
| **Key rotation** | Not supported (name is the key hash; rotation = new name) | Supported (delegate to a new zone key via PKEY record) |
| **Revocation** | No standard mechanism | PoW-based revocation certificate (4-5 days to compute; must be pre-computed) |
| **Transport options** | DHT, PubSub (GossipSub), HTTP (delegated routing) | R5N DHT only (GNUnet transport layer) |
| **Resolution latency** | ~11s median (p50), 37-60s outliers (Aug 2025) | Not formally measured on current network |
| **Post-quantum** | None (Ed25519 only) | In progress (GNUnet 0.26.x PQ layer, Dec 2025) |
| **Active development** | Yes (Kubo v0.34+, Feb 2026) | Yes (GNUnet 0.27.0, Mar 2026) |
| **Software maturity** | Production (IPFS ecosystem) | Alpha ("early adopters only") |
| **Licence** | MIT/Apache 2.0 (libp2p) | AGPL 3.0 (GNUnet) |

### Key Differences Explained

#### Privacy model

This is the most significant architectural difference. IPNS stores
records under the hash of the publisher's public key. Any DHT peer
handling a lookup can trivially determine which public key is being
queried. Over time, peers can build a profile of which nodes query which
keys.

GNS uses a **key-blinding scheme** (specified in RFC 9498, Section 5):
the DHT storage key is derived as `ZKDF(zone_key, label)`, a
zone-key-derivation function that produces a key computationally
unlinkable to either the zone key or the label without knowledge of
both. Records are encrypted with a key derived from the same inputs,
with the record expiry timestamp mixed into the IV. This means:

- DHT peers cannot determine which zone a query belongs to
- DHT peers cannot read the record contents
- Expired records are cryptographically inert (decryption fails)
- Zone enumeration requires brute-forcing all possible label strings

For a Logos naming system where query privacy is a design goal, GNS's
key-blinding approach is strongly preferred over IPNS's transparent
model.

#### Name model

IPNS provides a **flat, self-certifying namespace**: each name is the
hash of a public key, with no structure or hierarchy. This is simple
and elegant but means:

- No sub-namespaces (cannot delegate `service.alice` under `alice`)
- No key rotation without changing the name
- Human-readable names require an external system

GNS provides a **hierarchical, zone-based namespace**: each zone is a
keypair, and zones can delegate to sub-zones via PKEY/EDKEY records.
This enables:

- Arbitrary-depth delegation (`service.department.org.zTLD`)
- Key rotation by updating the delegation record in the parent zone
- Multiple record types per zone (addresses, DNS records, VPN configs)
- Petname-based trust model where each user has their own root

For a naming system that needs to support complex organisational
hierarchies, service discovery, and key lifecycle management, GNS's
zone model is more capable.

#### Record persistence and republishing

IPNS records expire after 48 hours by default. The publisher must
republish every 4 hours to maintain availability. If the publisher goes
offline, the name becomes unresolvable within 48 hours.

GNS records have publisher-defined expiry. The ZONEMASTER daemon handles
republishing. The expiry is cryptographically enforced (baked into the
encryption IV), so expired records cannot be served even if they remain
in the DHT.

Both systems share the fundamental limitation of DHT-based naming:
records are ephemeral. Neither provides the "write once, available
forever" guarantee of blockchain-based naming. This is the primary
technical gap that the two-layer architecture addresses: Layer 2 (on
LEZ) can provide persistence guarantees for the name-to-zone-key
mapping, while Layer 1 (DHT) handles the actual record distribution.

#### Transport flexibility

IPNS has a significant advantage in transport flexibility. The same
IPNS record can be distributed via:

- Amino DHT (Kademlia)
- PubSub (GossipSub for near-instant propagation)
- HTTP delegated routing API
- Any custom transport (the record format is transport-agnostic)

GNS is currently bound to GNUnet's R5N DHT and transport layer. While
the record format is specified independently (RFC 9498), the
implementation is tightly coupled to GNUnet.

For a Logos naming system that must operate over Waku messaging, libp2p,
and potentially HTTP, IPNS's transport-agnostic record model is a
better fit. However, GNS's privacy model (key blinding, record
encryption) should be adopted regardless of the transport layer.

#### Licensing

IPNS's implementation (via libp2p and Kubo) is MIT/Apache 2.0 dual
licensed, which is compatible with any downstream project. GNUnet
(including GNS) is AGPL 3.0, which requires derivative works to be
released under the same licence. For a Logos project that may need
permissive licensing, the IPNS codebase is more directly reusable, but
GNS's cryptographic design (key blinding, record encryption) can be
reimplemented under any licence since the specification is published as
an RFC.

### Recommendation for the RFP

The optimal design combines:

- **GNS's privacy model** (key blinding for DHT storage, record
  encryption, zone enumeration resistance)
- **IPNS's transport flexibility** (record format decoupled from
  transport; support for DHT, PubSub, HTTP, and Waku)
- **GNS's zone hierarchy** (delegation via zone key records, enabling
  key rotation and sub-namespaces)
- **A custom record format** (protobuf or CBOR) that extends GNS's
  record types with multi-chain address records (ENS-style coin types)

This is not a trivial integration. GNS's key blinding is tightly
coupled to its zone model and R5N DHT. Porting it to a libp2p Kademlia
DHT or Waku transport requires adapting the key derivation and
encryption to new primitives. The RFP should treat this as a research
and engineering challenge, not a configuration exercise.

---

## Appendix C — Other DHT-Based and P2P Name Services

Beyond GNS and IPNS, the research identified several other systems that
use DHT or P2P techniques for naming, though none are production-ready
alternatives:

### Waku Naming Service (WNS)

A hackathon prototype (September 2025) built on the Waku messaging
protocol (part of the Logos stack). WNS maps names to public keys
(rather than wallet addresses) and uses Waku messaging for private
resolution negotiation. It is not a production system but demonstrates
that the Waku transport layer can support naming queries. Relevant as a
proof-of-concept for Layer 1 resolution over Waku.

### Tor Onion Services (.onion)

Tor's hidden service naming uses a self-certifying model similar to
IPNS: the `.onion` address is derived from the service's public key.
Resolution goes through the Tor network's DHT (hash ring of HSDir
nodes). Provides strong anonymity but no human-readable names.
The `.onion` model validates that DHT-based self-certifying naming can
work at scale (millions of active services) when the privacy properties
are strong enough to drive adoption in specific use cases.

### Mainline DHT (BitTorrent)

BitTorrent's Mainline DHT is the largest Kademlia deployment in
production (~15-25 million nodes). While designed for content lookup
(infohash to peer list), BEP 44 added mutable items: signed key-value
pairs stored in the DHT, indexed by public key. This is architecturally
similar to IPNS records. Mainline DHT has been used experimentally for
naming (e.g., the Pkarr project) but lacks encryption, expiry
enforcement, and any privacy mechanism. Its scale demonstrates that
Kademlia DHTs can support hundreds of millions of entries.

### Pkarr

An experimental project that uses BitTorrent's Mainline DHT (BEP 44)
for DNS-compatible name resolution. Pkarr stores signed DNS resource
records under Ed25519 public keys in the Mainline DHT. It provides
self-certifying names with DNS compatibility but no privacy (records
are plaintext, keys are unhashed). Notable for its simplicity and
leverage of existing DHT infrastructure. Not production-grade.

### Emercoin EmerDNS

A Bitcoin-fork blockchain (not strictly DHT-based) that uses a
Name-Value Storage (NVS) system for domain names under the `.coin`,
`.emc`, and `.lib` TLDs. Included here because its NVS is conceptually
similar to a key-value DHT but backed by PoW consensus. Low adoption
(~$13M market cap). Demonstrates the failure mode of blockchain-based
naming without ecosystem integration: technically functional but unused.

### Summary: DHT Naming Landscape

| System | DHT/Transport | Human Names | Privacy | Production | Scale |
|--------|--------------|-------------|---------|------------|-------|
| GNS | R5N (GNUnet) | No (petnames) | Strong (key blinding) | Alpha | Tiny |
| IPNS | Amino (libp2p) | No (key hash) | Weak | Production | ~575K nodes |
| WNS | Waku | No (key based) | Moderate (Waku privacy) | Prototype | N/A |
| Tor .onion | Tor HSDir | No (key hash) | Strong (Tor anonymity) | Production | Millions |
| Mainline DHT | Mainline (BitTorrent) | No (key hash) | None | Production | ~20M nodes |
| Pkarr | Mainline (BitTorrent) | No (key hash) | None | Experimental | Inherits Mainline |
| EmerDNS | Blockchain (NVS) | Yes (.coin) | None | Production | Near zero |

The landscape confirms that DHT-based naming is technically viable at
massive scale (Mainline DHT, Tor) but no existing system combines
human-readable names, privacy, and high availability. The two-layer
architecture proposed in Appendix A is a novel contribution that draws
on the best properties of each system.

---

## Appendix D — Anti-Squatting Mechanisms

See [appendix/anti-squatting-mechanisms.md](../appendix/anti-squatting-mechanisms.md):
what prevents name squatting (liveness, Harberger pricing, eligibility),
what merely slows it (fees, auctions, staking), and a potential
Harberger + staking direction for Logos.
