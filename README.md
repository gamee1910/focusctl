# Focus Control (`focusctl`)

> **Personal Internet Firewall & Anti-Distraction Daemon for Linux**  
> System-level network filtering, scheduled focus enforcement, and anti-bypass distraction blocking.

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [System Architecture](#system-architecture)
3. [Core Technical Libraries](#core-technical-libraries)
4. [Blocking & Anti-Bypass Mechanisms](#blocking--anti-bypass-mechanisms)
5. [Blocklist Sources & Category Management](#blocklist-sources--category-management)
6. [Development Roadmap](#development-roadmap)
7. [Installation & Development Guide](#installation--development-guide)
8. [References & Useful Links](#references--useful-links)

---

## Quick Links & References

- Core Go Libraries:
  - DNS Server / Sinkhole: [miekg/dns](https://github.com/miekg/dns)
  - Linux nftables Controller: [google/nftables](https://github.com/google/nftables)
  - Pure-Go SQLite Driver: [modernc.org/sqlite](https://gitlab.com/cznic/sqlite)
  - CLI Framework: [spf13/cobra](https://github.com/spf13/cobra)
  - Systemd Integration: [coreos/go-systemd](https://github.com/coreos/go-systemd)
- Blocklists & Categorization:
  - StevenBlack Unified Hosts: [github.com/StevenBlack/hosts](https://github.com/StevenBlack/hosts)
  - OISD Blocklists: [oisd.nl](https://oisd.nl)
- Specifications & Linux Documentation:
  - Firefox DoH Canary Domain: [Mozilla Support: Canary Domain](https://support.mozilla.org/en-US/kb/canary-domain-use-application-dnsnet)
  - nftables Documentation: [nftables Wiki](https://wiki.nftables.org/)
  - Linux Capabilities (`CAP_NET_ADMIN`): [man7.org/linux/man-pages/man7/capabilities.7.html](https://man7.org/linux/man-pages/man7/capabilities.7.html)
  - RFC 8484 (DNS over HTTPS): [datatracker.ietf.org/doc/html/rfc8484](https://datatracker.ietf.org/doc/html/rfc8484)
  - RFC 7858 (DNS over TLS): [datatracker.ietf.org/doc/html/rfc7858](https://datatracker.ietf.org/doc/html/rfc7858)


---

## Project Overview

### The Problem
Traditional browser extensions and notification-based applications are trivially bypassed during moments of distraction (taking only a couple of clicks to disable the extension, launch another browser, or open an incognito window).

### The Focus Guard Solution
- **System/Network-Level Blocking**: Intervenes directly at the domain name resolution (DNS) and Linux kernel packet filtering (`nftables`) layers.
- **Strict Commitment Device (Anti-Bypass)**: Focus sessions cannot be easily cancelled with a simple `stop` command, and alternative DNS bypass vectors (DoH/DoT) are actively neutralized.
- **Flexible Management**: Supports timer sessions (`focusctl start 90m`), recurring schedule windows (`focusctl schedule`), and category-based filtering (Social, Adult, Gaming, Gambling, Custom).

---

## System Architecture

```text
                     ┌──────────────────┐
                     │   focusctl CLI   │
                     └─────────┬────────┘
                               │ 
                               ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                   focus-guardd (Daemon)                     │
  │                                                             │
  │  ┌──────────────┐   ┌──────────────┐   ┌─────────────────┐  │
  │  │  Scheduler   │   │ Policy/Lists │   │ SQLite Storage  │  │
  │  │ (Timer/Cron) │   │ (Categories) │   │ (Logs/Sessions) │  │
  │  └──────┬───────┘   └──────┬───────┘   └─────────────────┘  │
  │         └───────────┬──────┘                                │
  │                     ▼                                       │
  │         ┌───────────────────────┐                           │
  │         │ Network Enforcement   │                           │
  │         │ - Local DNS Proxy     │                           │
  │         │ - nftables Controller │                           │
  │         └───────────┬───────────┘                           │
  └─────────────────────┼───────────────────────────────────────┘
                        ▼
             Linux Kernel (Netfilter / DNS / IP Stack)
```

---

## Core Technical Libraries

### 1. Language & Platform
- **Go (Golang)**: v1.22+
- **Operating System**: Linux (Kernel v5.x+ with `nftables` and `systemd` support).

### 2. Selected Go Libraries

| Component | Recommended Library | Purpose & Rationale |
| :--- | :--- | :--- |
| **DNS Server / Sinkhole** | [`github.com/miekg/dns`](https://github.com/miekg/dns) | De-facto standard library in Go for high-performance DNS server/proxy implementations, supporting UDP/TCP and custom response generation (`0.0.0.0` or `NXDOMAIN`). |
| **nftables Controller** | [`github.com/google/nftables`](https://github.com/google/nftables) | Direct Netlink API communication with the Linux kernel in pure Go without CGO (`libnftnl`). Used to configure port 53 redirection and drop DoH/DoT traffic. |
| **Database** | [`modernc.org/sqlite`](https://gitlab.com/cznic/sqlite) | Pure Go SQLite driver without CGO dependencies, facilitating effortless cross-compilation and static binary distribution. |
| **CLI Framework** | [`github.com/spf13/cobra`](https://github.com/spf13/cobra) | Standard Go CLI framework for `focusctl`, offering subcommands, shell autocompletion, and flag parsing. |
| **IPC (Inter-Process Communication)** | Standard `net` package (Unix Domain Socket) | Secure communication between CLI and Daemon via `/var/run/focus-guard.sock` using JSON-RPC or REST-style JSON messages. |
| **Systemd Service** | [`github.com/coreos/go-systemd/v22`](https://github.com/coreos/go-systemd) | State notification integration (`sd_notify`) for systemd service watchdog monitoring. |

---

## Blocking & Anti-Bypass Mechanisms

### 1. DNS Sinkhole & Forwarder
- The daemon launches a DNS Proxy listening on `127.0.0.1:5353` (or `127.0.0.53:53`).
- **Query Resolution Workflow**:
  - Upon receiving a domain resolution query, it is checked against an in-memory Trie cache (reversed domain suffix tree).
  - If the domain matches a blocked category: Immediately return `0.0.0.0` (A record) or `::` (AAAA record).
  - If the domain is allowed: Forward the query to the upstream resolver (e.g., Cloudflare `1.1.1.1` or the system resolver) and relay the answer back to the client.

### 2. DNS Anti-Bypass (Transparent Redirection via nftables)
To prevent client software from bypassing the local sinkhole by querying external DNS servers (such as `8.8.8.8` or `1.1.1.1` directly):
- **NAT Redirection Rules**:
  ```bash
  # Redirect all outbound DNS traffic (port 53 UDP/TCP) to Focus Guard's DNS Proxy
  nft add table inet focus_guard
  nft add chain inet focus_guard prerouting '{ type nat hook prerouting priority dstnat; }'
  nft add rule inet focus_guard prerouting udp dport 53 redirect to :5353
  nft add rule inet focus_guard prerouting tcp dport 53 redirect to :5353
  ```

### 3. Neutralizing DoH (DNS over HTTPS) & DoT (DNS over TLS)
Modern web browsers (Firefox, Chrome) often enable DoH by default, encrypting DNS queries to circumvent local port 53 DNS filters:
1. **Canary Domain (Firefox/Chrome Standard)**:
   - Responds to queries for `use-application-dns.net` with `NXDOMAIN`. When receiving this signal, Firefox automatically disables DoH and reverts to system DNS resolution. See [Mozilla Canary Domain Documentation](https://support.mozilla.org/en-US/kb/canary-domain-use-application-dnsnet).
2. **Block DoT Port**: Reject or drop outgoing traffic on port `853` (TCP/UDP) via nftables rules.
3. **Block Known DoH Resolver IP Ranges**: Drop direct outbound connections to IP addresses of popular public DoH endpoints (Cloudflare, Google, Quad9, NextDNS) while a focus session is active.

### 4. Session Commitment (Anti-Cheat Mechanism)
- Intercepts and rejects attempts to prematurely stop sessions (`focusctl stop` returns an error during an active session).
- Configurable enforcement modes:
  - **Strict Mode**: Unconditional lock; the session cannot be terminated before the timer elapses.
  - **Penalty Mode**: Requires typing a randomized 100–200 character terminal captcha string if an emergency abort is requested.
- Process resilience backed by `systemd` (`Restart=always`) with graceful cleanup handlers to restore standard network configuration upon system shutdown.

---

## Blocklist Sources & Category Management

Focus Guard supports ingesting and parsing community-standard blocklists:

- **StevenBlack Hosts**:
  - Unified base list with composable extensions: `porn`, `gambling`, `social`, `fakenews`.
  - Repository: [github.com/StevenBlack/hosts](https://github.com/StevenBlack/hosts)
- **OISD Blocklists**:
  - High-quality, low-false-positive domain categorization lists.
  - Homepage: [oisd.nl](https://oisd.nl)
- **In-Memory Storage & Fast Lookups**:
  - Utilizes a **Domain Suffix Trie** in RAM: Delivers $O(L)$ lookups where $L$ is the number of domain segments (e.g., `m.facebook.com` -> `com` -> `facebook` -> `m`).
  - Custom rules, exceptions, and category settings persist across sessions in SQLite.

---

## Development Roadmap

### Phase 1: MVP (Minimum Viable Product)
- [ ] Initialize Go module and repository structure (`cmd/` and `internal/`).
- [ ] Implement local DNS proxy using [`miekg/dns`](https://github.com/miekg/dns) returning `0.0.0.0` for a simple blacklist.
- [ ] Build Unix Domain Socket server (`focus-guardd`) and client (`focusctl`).
- [ ] Implement session command: `focusctl start <duration>` with automated session unlock when time expires.

### Phase 2: Category & Blocklist Engine
- [ ] Parser for `hosts` file formats and raw domain lists.
- [ ] In-memory Suffix Trie matcher for low-latency queries across hundreds of thousands of domains.
- [ ] SQLite integration ([`modernc.org/sqlite`](https://gitlab.com/cznic/sqlite)) for persistent session history and custom domain rules.
- [ ] Built-in categories: `social`, `porn`, `gambling`, `gaming`.

### Phase 3: nftables & Anti-Bypass Hardening
- [ ] Integrate [`google/nftables`](https://github.com/google/nftables) for port 53 redirection and DoH/DoT packet filtering.
- [ ] Implement DoH Canary Domain (`use-application-dns.net`) response handling.
- [ ] Recurring schedule engine based on weekday time windows (e.g., `08:00 - 12:00 BLOCK`).
- [ ] Additional CLI commands: `focusctl status`, `focusctl schedule`, `focusctl history`.

### Phase 4: Hardening & Analytics
- [ ] Strict Mode and Terminal Captcha penalty workflows.
- [ ] Blocked query analytics (per-domain and per-category counters).
- [ ] Production packaging with `systemd` service units and an automated installation script.

---

## Installation & Development Guide

### Environment Prerequisites
- Go 1.22 or newer: [go.dev/dl](https://go.dev/dl/)
- Linux Kernel with `nftables` support
- `systemd` (recommended for background service management)

### Basic Commands (Planned)
```bash
# 1. Build binaries
go build -o bin/focusctl ./cmd/focusctl
go build -o bin/focus-guardd ./cmd/focus-guardd

# 2. Run Daemon with network administration capabilities
sudo ./bin/focus-guardd

# 3. Control via CLI
./bin/focusctl start 45m
./bin/focusctl status
```

---

## References & Useful Links

- **Go Libraries**:
  - [miekg/dns](https://github.com/miekg/dns) - Go DNS library
  - [google/nftables](https://github.com/google/nftables) - Pure Go Netlink client for nftables
  - [modernc.org/sqlite](https://gitlab.com/cznic/sqlite) - Pure Go SQLite driver
  - [spf13/cobra](https://github.com/spf13/cobra) - Modern Go CLI application framework
  - [coreos/go-systemd](https://github.com/coreos/go-systemd) - Go bindings to systemd socket activation and notification
- **Blocklists & Feeds**:
  - [StevenBlack/hosts](https://github.com/StevenBlack/hosts) - Curated hosts lists with categorized extensions
  - [oisd.nl](https://oisd.nl) - Curated domain blocklists
- **Network Standards & Linux Docs**:
  - [Mozilla Support: Canary Domain](https://support.mozilla.org/en-US/kb/canary-domain-use-application-dnsnet) - Disabling DoH via canary domain
  - [nftables Wiki](https://wiki.nftables.org/) - Official documentation for Linux nftables
  - [RFC 8484: DNS Queries over HTTPS (DoH)](https://datatracker.ietf.org/doc/html/rfc8484)
  - [RFC 7858: DNS over TLS (DoT)](https://datatracker.ietf.org/doc/html/rfc7858)
  - [Linux Capabilities (`CAP_NET_ADMIN`)](https://man7.org/linux/man-pages/man7/capabilities.7.html)
