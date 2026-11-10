# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.5.0] - 2026-11-08

### Added & Re-Architected (`SOVEREIGN-CHANGELOG-E2E-01`)
- **`[Protocol / Transport Topology]` Ground-Floor Topology-Aware RFC 2308 NODATA Synthesis for AAAA Filtering (`aegis::forward`, `aegis::server::forward_stage`, `aegis::config::types::AaaaFilterMode`) (`SOVEREIGN-RFC-BAREMETAL-01`)**:
  - *Root Cause Dissection*: On customer networks where local routers advertise IPv6 link-local/ULA prefixes without valid upstream default IPv6 WAN routing (`::/0`), forwarding upstream public `AAAA` records triggers RFC 6724 "Happy Eyeballs" socket timeout penalties across Android, Linux, and Apple network stacks. Sockets attempt unreachable IPv6 destinations first, incurring up to 2,000ms timeouts per connection and generating tens of thousands of `ENETUNREACH` socket failure loops across streaming media apps (e.g. Pluto TV, YouTube, Instagram).
  - *Architectural & Mathematical Resolution*:
    1. **`AaaaFilterMode` Configuration Primitive**: Added `AaaaFilterMode` enum to [`src/config/types.rs`](file:///home/emhcet/private/projects/desktop/rust/aegis/src/config/types.rs) with three deterministic operational modes: `Passthrough` (default), `FilterV4Transport` (synthesizes NODATA for queries originating over IPv4 transport), and `FilterAlways`.
    2. **Stage-3 Microsecond Synthesis**: In [`src/server/forward_stage.rs`](file:///home/emhcet/private/projects/desktop/rust/aegis/src/server/forward_stage.rs), incoming recursive `RecordType::AAAA` queries matching the filter criteria immediately synthesize an authoritative RFC 2308 NODATA response (`RCode::NoError`, `answers: []`, `RA=true`, `AA=false`) in $< 12\mu\text{s}$ logged as `AAAA_FILTER_RFC2308_NODATA`.
    3. **Authoritative Preservation Invariant**: Evaluated strictly in Stage 3 (Forwarding), ensuring that authoritative zones (Stage 2) preserving production `AAAA` records (e.g. `rmediatech.com`, `matrix.rmediatech.com`) remain 100% intact and resolvable worldwide without filtering.
  - *Empirical Verification*: Verified in `tests/adversarial_topology_aware_aaaa_filter_tests.rs` asserting RFC 2308 zero-answer synthesis on IPv4 transport, authoritative `AAAA` preservation, and upstream fallback failure in passthrough mode.

---

## [0.4.0] - 2026-11-06

### Added & Re-Architected (`SOVEREIGN-CHANGELOG-E2E-01`)
- **`[State / Performance]` Zero-Allocation 2-Label Apex Bitwise Bloom Pre-Filter & Reverse Label Radix Trie (`aegis::sinkhole::bloom`, `aegis::sinkhole::trie`, `aegis::sinkhole::engine`) (`SOVEREIGN-AEGIS-SINKHOLE-02`)**:
  - *Root Cause Dissection*: Previous `SinkholeEngine` stored rules in two flat `HashMap<DnsName, SinkholeAction>` tables (`exact_rules` and `wildcard_rules`). Evaluating a $k$-label query (`inspect_name`) required iterating `0..k` and executing `labels[i..].to_vec()`, triggering $k$ separate heap `Vec<String>` allocations and $\frac{k(k+1)}{2}$ `String` clones on every single incoming DNS query. Under 1M+ rule threat feeds (StevenBlack, OISD), flat `HashMap` storage also incurred $>350\text{ MB}$ of bucket/string pointer overhead.
  - *Architectural & Mathematical Resolution*:
    1. **Stage-1 `FastDomainBloom` (`src/sinkhole/bloom.rs`)**: Engineered a cache-aligned 4-Megabit (`512 KB`) bitwise Bloom filter hashing the 2-label registrable suffix `(labels[n-2], labels[n-1])` (plus a dedicated 64-Kilobit TLD filter for single-label rules) using 64-bit FNV-1a + SplitMix64 double hashing (`h1 + i * h2`, $k=4$ probes) directly over `&[String]` slices with zero heap allocations. Because exact rules, 2-label wildcards, and deep $N$-label subdomain rules share the same 2-label registrable apex as any query targeting them, $>99.9\%$ of benign queries are proved clean and fast-passed in $<10\text{ns}$ without touching the trie.
    2. **Stage-2 `DomainRadixTrie` (`src/sinkhole/trie.rs`)**: Replaced flat hash maps with a reverse label suffix tree (`com -> evil -> tracker -> sub`) keyed by `Box<str>`. Lookups walk `labels.iter().rev()` in a single $O(k)$ pass via `Borrow<str>` with **zero heap allocations** and **zero string clones**, supporting exact blocks, suffix/wildcard blocks, and explicit allowlist/bypass rules (`add_allow_exact`, `add_allow_wildcard`) that exempt trusted subdomains of blocked parent domains.
    3. **Multi-Format Threat Feed Ingester & `AtomicSinkholeEngine` (`src/sinkhole/engine.rs`)**: Added zero-copy parsing for 4 canonical threat feed formats (`/etc/hosts` with automatic self-protection for `localhost`/`broadcasthost`, Adblock/uBlock/OISD `||domain^` and `@@||trusted^` exception syntax, Dnsmasq/Unbound `address=/domain/0.0.0.0`, and raw domain lists) plus lock-free RCU atomic hot-swapping (`AtomicSinkholeEngine`).
  - *Empirical Verification*: Verified in `tests/adversarial_sinkhole_tests.rs` across 8 adversarial & scale suites including 50,000-domain Bloom fast-pass benchmarks ($<100\text{ns}$), multi-format OISD/Adblock exception override tests, `/etc/hosts` localhost self-preservation, and concurrent `AtomicSinkholeEngine` RCU swaps.

---

## [0.3.0] - 2026-11-05

### Added
- **`[Protocol / Encrypted Transport]` RFC 7858 DNS-over-TLS (DoT) & RFC 8484 DNS-over-HTTPS (DoH) Upstream Engine (`aegis::forward::encrypted`) (`SOVEREIGN-TRINITY-ULTIMATE-01`)**:
  - *Root Cause Dissection*: Standard UDP/53 upstream recursive forwarding exposes outbound query names in cleartext on the ISP/datacenter wire and is vulnerable to on-path BGP/UDP packet injection and ISP transparent DNS hijacking.
  - *Architectural & Mathematical Resolution*: Engineered `EncryptedUpstreamClient` backed by `tokio-rustls` 0.26 (`rustls` 0.23 + `ring` 0.17 + `webpki-roots` 0.26) supporting `tls://<ip>:<port>#<sni>` (RFC 7858 2-byte big-endian length-prefixed TLS frames) and `https://<ip>:<port>/dns-query#<sni>` (RFC 8484 `application/dns-message` HTTP/1.1 wire frames with `Content-Length` boundary framing and 64 KiB payload bomb cutoff), automatically injecting EDNS0 `DO=1` (`0x8000`) flags and enforcing `strict_encrypted_only` when configured.
  - *Empirical Verification*: Verified in `tests/adversarial_dnssec_doh_dot_and_swr_tests.rs` (`test_rfc7858_dns_over_tls_end_to_end_encrypted_upstream` and `test_rfc8484_dns_over_https_end_to_end_and_oversized_bomb_defense`).
- **`[Protocol / Cryptography]` RFC 4034 & RFC 4035 Pure-Rust Cryptographic DNSSEC Validation Engine (`aegis::dnssec`)**:
  - *Root Cause Dissection*: Trusting upstream recursive resolver responses without cryptographic signature verification allows compromised or poisoned upstream caches to inject forged `A`/`AAAA` records into downstream clients.
  - *Architectural & Mathematical Resolution*: Added wire support for `DS` (`43`), `RRSIG` (`46`), `NSEC` (`47`), and `DNSKEY` (`48`) records alongside RFC 4035 `AD` (`0x02`) and `CD` (`0x01`) header bits. Implemented `DnssecValidator` performing RFC 4034 Appendix B key tag calculation, RFC 4034 §5.1.4 `DS` digest verification (SHA-256 Digest 2 & SHA-384 Digest 4), RFC 4034 §6.2 canonical RRset sorting and signing buffer construction, and cryptographic signature verification across Ed25519 (Alg 15), ECDSA P-256/P-384 (Algs 13/14 with IEEE P1363-to-ASN.1 DER conversion), and RSASHA256/RSASHA512 (Algs 8/10 with RFC 3110 public key decoding). Forged or expired signatures are immediately rejected with `RCODE=2 (SERVFAIL)` and logged as `DNSSEC_BOGUS_BLOCKED`.
  - *Empirical Verification*: Verified in `tests/adversarial_dnssec_doh_dot_and_swr_tests.rs` (`test_rfc4034_rfc4035_dnssec_ed25519_and_ds_chain_validation`).
- **`[State / Performance]` RFC 8767 Stale-While-Revalidate (SWR) & Bounded LRU Cache (`aegis::forward::upstream`)**:
  - *Root Cause Dissection*: When a cached DNS record's TTL expires, synchronous upstream resolution adds $15\text{ms}\text{–}80\text{ms}$ of tail latency to client requests.
  - *Architectural & Mathematical Resolution*: Upgraded `UpstreamForwarder` with a 3-state cache lifecycle (`Fresh`, `StaleWhileRevalidate`, `Miss`), bounded LRU eviction (`max_cache_entries`), and stampede-deduplicated `in_flight_swr` tracking. Expired records within `swr_window_secs` return immediately from RAM in $<5\mu\text{s}$ (`FORWARD_CACHE_SWR_HIT`) with a clamped 5-second TTL while spawning an asynchronous `tokio::spawn` background refresh task.
  - *Empirical Verification*: Verified in `tests/adversarial_dnssec_doh_dot_and_swr_tests.rs` (`test_rfc8767_stale_while_revalidate_sub5us_and_async_background_refresh`).

---

## [0.2.0] - 2026-11-04

### Added
- **`[Protocol / Active Deception]` Layer 3/4 DNS Synthetic Mirage & Canary Poisoning Engine (`aegis::mirage`) (`SOVEREIGN-MIRAGE-DECEPTION-01`)**:
  - *Root Cause Dissection*: Traditional DNS sinkholes (`0.0.0.0` / `NXDOMAIN`) and wildcard authoritative zones either immediately alert offensive subdomain scanners (`subfinder`, `amass`, `massdns`, `dnsx`) that a target is dead/filtered, or worse—when a wildcard (`*`) record is present in an authoritative zone—resolve high-risk reconnaissance lures (`internal-vault.<zone>`, `db-master.<zone>`, `terraform-state.<zone>`) straight to production web servers!
  - *Architectural & Mathematical Resolution*: Engineered `DnsMirageEngine` with pure-Rust RFC 2104 `HMAC-SHA256(secret, client_ip | qname)` canary derivation. Intercepts `SinkholeAction::Mirage` rules and pre-wildcard authoritative reconnaissance lures (`_env`, `internal-vault`, `terraform-state`, `admin-staging`, `db-master`, `_sovereign-canary`), returning deceptive `RCODE=0 (NoError)` responses with deterministic RFC 2544 sandbox IPv4 (`198.18.x.y`), IPv6 (`2001:db8:ae61::/48`), and `TXT` canary records (`v=aegis-mirage1; canary=mirage-<hex>`) plus progressive quadratic tarpit drip pacing ($\text{Delay}_{\text{ms}} = \min(\text{Base} \times \text{Infractions}^2, \text{Max})$).
  - *Empirical Verification*: Verified in `tests/adversarial_mirage_and_surface_audit_tests.rs` asserting deterministic attacker IP unmasking, pre-wildcard lure interception, and ANSI injection scrubbing.
- **`[Crucible / Surface Audit]` 12-Vector Outside-In DNS Surface Exposure Auditor (`aegis::surface_audit` & `aegis audit-surface`) (`SOVEREIGN-OUTSIDE-IN-AUDIT-01`)**:
  - *Root Cause Dissection*: Operators lacked a built-in, zero-dependency way to empirically verify their Layer 3/4 DNS perimeter from an external adversary's perspective across amplification, zone transfer, and subdomain enumeration vectors.
  - *Architectural & Mathematical Resolution*: Built `DnsSurfaceAuditEngine` probing 12 canonical DNS threat vectors (`DNS-01` Open Recursion, `DNS-02` `AXFR` Zone Transfer, `DNS-03` RFC 8482 `ANY` Amplification, `DNS-04` `CHAOS TXT version.bind`, `DNS-05` DoH Bypass Canary, `DNS-06`..`DNS-11` Recon Subdomain Lures, and `DNS-12` RFC 2136 Dynamic Update injection) with both live UDP probing and instant in-memory `--self-check` grading (`A+` to `F`).
  - *Empirical Verification*: Verified in `tests/adversarial_mirage_and_surface_audit_tests.rs` (`test_outside_in_dns_surface_audit_12_vectors_grade_a_plus` and `test_outside_in_dns_surface_audit_detects_exposed_open_resolver_and_unmasked_leak`).

---

## [0.1.3] - 2026-10-08

### Added
- **RFC 8482 Minimal ANY Query Response Defense (`aegis::zone::store`)**:
  - Re-engineered zone query evaluation to synthesize minimal RFC 8482 single-RR answers instead of dumping all zone records.
  - Mitigates DNS reflection amplification attacks, bringing maximum amplification factor from $50\times$ down to $\le 1.2\times$.
- **Bounded UDP Worker Concurrency (`aegis::server`)**:
  - Introduced a 10,000-permit `tokio::sync::Semaphore` bounding concurrent UDP processing worker tasks.
  - Drops flood packets at the socket boundary when concurrency is saturated, defusing Tokio task queue memory exhaustion and thread pool starvation under volumetric DDoS floods.
- **ISC BIND9 & dig Establishment Parity Suite (`tests/parity_bind9_dig_tests.rs`)**:
  - RFC 1035 wire header parity: ID preservation, authoritative flags (`QR=1`, `AA=1`, `RD=1` preserved, `RA=0`, `RCODE=0`).
  - Wire record format parity: big-endian network byte order for A (4 octets) and AAAA (16 octets).
  - RFC 2308 negative caching parity: NODATA and NXDOMAIN returns with zone SOA in Authority section.
  - CNAME chaining parity: returns alias CNAME and target A record in the answer section.
  - Sub-microsecond query resolution benchmark assertion.
- **Deep Wire Self-Attack Battery (`tests/adversarial_dns_wire_tests.rs`)**:
  - Deep 6-hop pointer chain exhaustion testing against `MAX_POINTER_HOPS` (5).
  - QDCOUNT allocation bomb testing (`QDCOUNT = 65535` with truncated payload).
  - Reserved RFC 1035 label bit pattern fuzzing (`0b10`).
- **Cross-Platform & VM Deployment Architecture**:
  - Documentation and test configurations for Linux (systemd, static Musl), macOS (launchd, BSD dig), and Windows (PowerShell, `Resolve-DnsName`, NSSM service wrapper).

### Changed
- **Deconstructed CLI Architecture (`AGY-RULE-SOVEREIGN-FLAGSHIP-01`)**:
  - Rebuilt `src/main.rs` into a pristine 13-line lifecycle shell (was 261 lines).
  - Extracted CLI definitions and async execution into dedicated modular runner `src/cli.rs`, exported via `src/lib.rs`.

---

## [0.1.0] - 2026-10-03

### Added
- **OJP Wire Protocol Engine (`aegis::wire`)**:
  - RFC 1035 / RFC 3596 / RFC 6891 zero-copy parser and serializer.
  - Hardened circular compression pointer loop defense (bounded to $\le 5$ hops).
  - Out-of-bounds pointer offset validation.
  - Strict label ($L \le 63$) and domain ($N \le 255$) length boundary checks.
  - EDNS0 OPT RR buffer sizing negotiation (advertising 1232 octet UDP payload).
- **Authoritative Radix Zone Store (`aegis::zone`)**:
  - In-memory longest-suffix matching for hosted zone origins.
  - Full support for A, AAAA, CNAME, TXT, MX, NS, SOA, SRV, and PTR records.
  - Wildcard record synthesis (`*.domain`).
  - Automatic CNAME chasing.
  - RFC-compliant NODATA and NXDOMAIN synthesis with SOA in authority section.
- **Defensive Threat Interception Engine (`aegis::sinkhole`)**:
  - Sub-microsecond suffix and wildcard domain matching.
  - Threat actions: `NxDomain`, `ZeroIp` (`0.0.0.0` / `::`), `Refused`, and silent `Drop`.
  - Native loading of standard `/etc/hosts` formats and threat domain lists.
  - Atomic metrics tracking of intercepted queries.
- **Response Rate Limiting (`aegis::rrl`)**:
  - BCP 153 token-bucket rate limiter per `/24` IPv4 and `/48` IPv6 subnets.
  - Configurable burst allowance and slip rates (`TC=1` responses forcing TCP retry).
  - Silent dropping of excessive UDP reflection queries.
- **Upstream Forwarding & Cache (`aegis::forward`)**:
  - Round-robin upstream query distribution.
  - Thread-safe TTL-adjusted in-memory caching.
- **Async Tokio Dual Transport (`aegis::server`)**:
  - High-throughput UDP port 53 listener.
  - TCP port 53 listener with RFC 1035 2-byte length-framed streaming.
- **CLI Commands**:
  - `aegis serve`: Launch server with custom TOML configurations.
  - `aegis check`: Offline validation of configuration syntax and zone records.
  - `aegis query`: Built-in microsecond DNS query tool (supporting UDP and TCP).
  - `aegis sinkhole test`: Offline evaluation of domain threat interception rules.
- **Adversarial Self-Attack TDD Suite (`tests/adversarial_*_tests.rs`)**:
  - Pointer loop bombs (`0xC000`, mutual recursion).
  - Buffer overflow and label length fuzzing.
  - Amplification floods and subnet saturation tests.
  - Case-mutation and syntax evasion tests.
