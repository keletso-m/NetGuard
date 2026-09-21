# NetGuard

**Network visibility, change detection, attack-surface monitoring, and policy enforcement for authorized networks.**

NetGuard is a network security application built from scratch in **Rust**. It discovers devices and exposed services, maintains network snapshots, detects changes over time, evaluates network attack surfaces, records security events, and applies configurable firewall policies.

The project focuses on building a lightweight and explainable security system rather than replacing tools such as Nmap, Nessus, SIEM platforms, or enterprise firewalls.

> **Discover your network. Track what changes. Understand what is exposed. Control what is allowed.**

---

# Project Goals

NetGuard is designed to explore practical network-security engineering using the Rust ecosystem.

The project focuses on:

* Network interface and host discovery
* Raw packet-based network discovery
* TCP service enumeration
* Network inventory and historical snapshots
* Change detection
* Attack-surface monitoring
* Device trust and security policies
* Firewall rule evaluation and enforcement
* Security event collection
* Explainable risk assessment
* REST API development
* Web-based network monitoring
* Structured logging and observability
* Automated testing and benchmarking
* Containerized deployment
* CI/CD with GitHub Actions

The goal is not to build another Nmap or enterprise firewall.

The goal is to understand how **network visibility, security policy, enforcement, and monitoring fit together into one system**.

---

# Architecture

```text
                         ┌──────────────────────┐
                         │       NetGuard       │
                         └──────────┬───────────┘
                                    │
             ┌──────────────────────┼──────────────────────┐
             │                      │                      │
             ▼                      ▼                      ▼
      ┌─────────────┐       ┌──────────────┐       ┌──────────────┐
      │   Network   │       │   Security   │       │   Firewall   │
      │    Engine   │       │    Engine    │       │    Engine    │
      └──────┬──────┘       └──────┬───────┘       └──────┬───────┘
             │                      │                      │
       ┌─────┼─────┐          ┌─────┼─────┐          ┌─────┼─────┐
       │     │     │          │     │     │          │     │     │
       ▼     ▼     ▼          ▼     ▼     ▼          ▼     ▼     ▼
     ARP   ICMP  TCP        Trust   Risk  Events    Rules Policy Status
     Scan  Scan  Scan
       │     │     │
       └─────┴─────┘
             │
             ▼
      ┌─────────────────┐
      │   Persistence   │
      │ SQLite / Postgres│
      └────────┬────────┘
               │
               ▼
      ┌─────────────────┐
      │    Axum API     │
      └────────┬────────┘
               │
               ▼
      ┌─────────────────┐
      │    Dashboard    │
      │   HTMX + HTML   │
      └─────────────────┘
```

NetGuard separates **observation** from **enforcement**.

The network engine observes the environment, while the security and policy engines determine how that information should be interpreted and what actions should be taken.

OS-specific firewall functionality is isolated behind a firewall abstraction so the core application remains portable.

---

# Features

## Network Visibility

NetGuard maintains an inventory of devices and services discovered on the local network.

Features include:

* Automatic network-interface discovery
* CIDR/network range detection
* ARP-based host discovery
* ICMP-based host discovery
* TCP service discovery
* IP address tracking
* MAC address tracking where available
* Hostname resolution
* Vendor identification where available
* Configurable scan ranges
* Periodic scanning
* Device inventory
* Optional packet capture using `pcap`

Example:

```text
NETWORK INVENTORY

192.168.1.1
  Router
  80/tcp     HTTP
  443/tcp    HTTPS

192.168.1.20
  Laptop
  22/tcp     SSH
  631/tcp    IPP

192.168.1.42
  Unknown Device
  80/tcp     HTTP
  445/tcp    SMB
```

---

# Network Snapshots

Every scan can produce a network snapshot.

A snapshot records the observed state of the network at a specific point in time.

```text
Snapshot
────────────────────────────

Time:
2026-09-21 20:42

Devices:
14

Services:
31

Changes:
3
```

Snapshots allow NetGuard to answer questions such as:

* What devices were present?
* What services were exposed?
* What changed since the previous scan?
* When was a service first observed?
* When was a device last seen?
* Which services disappeared?
* Which devices are new?

Snapshots can be serialized using `bincode` for internal storage or transport where appropriate.

---

# Change Detection

NetGuard compares network snapshots to identify changes.

### New Device

```text
NEW DEVICE

192.168.1.72
Unknown hostname
Unknown vendor

First observed:
2026-09-21 20:42
```

### New Service

```text
SERVICE CHANGE

Device:
192.168.1.20

Added:
TCP/8080

Previously:
Not observed
```

### Removed Service

```text
SERVICE CHANGE

Device:
192.168.1.20

Removed:
TCP/445
```

Changes are persisted as security and network events so that historical network activity can be investigated over time.

---

# Attack-Surface Monitoring

NetGuard provides a lightweight view of the network's exposed attack surface.

It does **not** attempt to perform full vulnerability assessment.

Instead, it focuses on questions such as:

* Which devices expose services?
* Which services appeared recently?
* Which ports are considered sensitive?
* Which devices have unusually large exposed surfaces?
* Which services have changed since the previous snapshot?

Example:

```text
DEVICE
192.168.1.20

Attack Surface
────────────────────────────

22/tcp       SSH
80/tcp       HTTP
445/tcp      SMB
3389/tcp     RDP

Risk: HIGH

Reasons:
• Multiple exposed services
• Remote-access service detected
• SMB exposed
```

Risk decisions are designed to remain **explainable** rather than relying on an opaque machine-learning model.

---

# Device Trust

Devices can be assigned a security state:

```text
TRUSTED
UNKNOWN
RESTRICTED
BLOCKED
```

Trust state can influence the policies applied to a device.

Example:

```text
UNKNOWN DEVICE POLICY

HTTPS       ALLOW
DNS         ALLOW
SSH         BLOCK
SMB         BLOCK
RDP         BLOCK
```

This allows NetGuard to treat newly discovered or untrusted devices differently from known devices.

---

# Firewall & Policy Enforcement

NetGuard includes a configurable policy engine for controlling network communication.

Policies can define:

* Source IP / CIDR
* Destination IP / CIDR
* TCP / UDP
* Source ports
* Destination ports
* Allow / deny actions
* Rule priority
* Rule descriptions
* Enable / disable state
* Device trust requirements

Example:

```text
RULE #20

Name:
Block SMB from unknown devices

Source:
UNKNOWN

Destination:
192.168.1.0/24

Protocol:
TCP

Port:
445

Action:
DENY

Priority:
20
```

Rules are evaluated according to their priority and matching conditions.

---

# Firewall Integration

NetGuard is **not intended to replace the operating system's firewall subsystem**.

Instead, NetGuard acts as a policy and management layer and integrates with the host firewall for enforcement.

```text
NetGuard
    │
    ▼
Policy Engine
    │
    ▼
Firewall Adapter
    │
    ▼
Linux nftables
    │
    ▼
Network Traffic
```

Linux enforcement can use:

* `nft`
* `nftnl`
* `rtnetlink`

The firewall layer is isolated behind an internal abstraction so that platform-specific implementation does not leak into the core security engine.

---

# Security Events

Firewall activity and network changes can produce security events.

Example:

```text
SECURITY EVENT

Time:
20:32:18

Device:
192.168.1.72

Destination:
192.168.1.20

Protocol:
TCP

Port:
445

Action:
BLOCKED

Rule:
Block SMB from unknown devices

Reason:
Policy violation
```

Events provide evidence for later investigation.

Structured logging is implemented using the Rust `tracing` ecosystem.

---

# Security Intelligence

NetGuard correlates network observations, policy decisions, and security events.

Example:

```text
New device detected
        │
        ▼
Device is UNKNOWN
        │
        ▼
Restricted policy applied
        │
        ▼
SMB connection attempted
        │
        ▼
Firewall blocks connection
        │
        ▼
Security event recorded
        │
        ▼
Policy violation detected
        │
        ▼
Risk updated
        │
        ▼
Alert + evidence
```

A device's security state can therefore be based on more than simply the ports it exposes.

Example:

```text
DEVICE
192.168.1.42

Risk: MEDIUM
Trust: UNKNOWN

Reasons:

• New device
• 3 previously unseen services
• 27 blocked connection attempts
• Attempted access to restricted subnet
```

---

# Detection Scenarios

NetGuard is designed around explicit, testable security scenarios.

| Scenario                      | Expected Result  |
| ----------------------------- | ---------------- |
| Trusted device → HTTPS        | Allow            |
| Unknown device → HTTPS        | Allow            |
| Unknown device → SMB          | Block            |
| Device → restricted subnet    | Block            |
| New service appears           | Alert            |
| Unusual connection burst      | Alert            |
| Previously unseen destination | Record           |
| Policy violation              | Alert + evidence |

Additional policy tests verify:

* Rule priority
* Disabled rules
* Trust-state changes
* Repeated violations
* Firewall failures
* Policy conflicts
* Invalid configuration

---

# Technology Stack

## Language & Runtime

* **Rust**
* **Tokio**
* Rust async/await
* `CancellationToken`
* `Arc`
* `Mutex` / `RwLock` where appropriate
* Rust channels and task coordination

Tokio provides the asynchronous runtime for network operations, background scanning, API handling, and internal task scheduling.

---

## Network Discovery

* `pnet`
* `socket2`
* Standard Rust networking primitives

`pnet` provides access to lower-level networking functionality for tasks such as:

* ARP discovery
* ICMP probing
* Raw packet construction
* Packet inspection

`socket2` is used where lower-level socket configuration is required.

---

## Packet Capture

Optional deeper network visibility can be provided through:

* `pcap`
* libpcap

Packet capture is intended to support additional visibility and security-event collection without making deep packet inspection a core requirement of the project.

---

## Firewall Enforcement

Linux firewall integration can use:

* `nft`
* `nftnl`
* `rtnetlink`

The implementation is isolated behind a firewall adapter.

```text
FirewallEngine
      │
      ▼
FirewallAdapter
      │
      ├── Linux / nftables
      │
      └── Future platform adapters
```

---

## CLI

NetGuard's command-line interface uses:

* `clap`

Example commands:

```bash
netguard scan
netguard devices
netguard services
netguard changes
netguard events
netguard policy list
netguard policy apply
netguard firewall status
```

---

## Serialization

NetGuard uses:

* `serde`
* `serde_json`
* `bincode`

`serde_json` is used for configuration and API-facing serialization.

`bincode` can be used for compact internal snapshots and binary persistence where appropriate.

---

## Persistence

Database access uses:

* `sqlx`
* SQLite for development
* PostgreSQL for production

SQLite keeps local development simple while PostgreSQL provides a production-oriented relational backend without tying the application to a specific cloud provider.

The persistence layer stores information such as:

* Devices
* Network snapshots
* Services
* Network changes
* Security events
* Firewall policies
* Risk findings
* Device trust state

---

## Web API

The HTTP API is built with:

* **Axum**
* Tokio
* `serde`
* `serde_json`

Example endpoints:

| Method | Endpoint               | Description                      |
| ------ | ---------------------- | -------------------------------- |
| GET    | `/api/devices`         | List discovered devices          |
| GET    | `/api/devices/:id`     | Get device details               |
| GET    | `/api/scans`           | List network snapshots           |
| GET    | `/api/changes`         | List detected changes            |
| GET    | `/api/events`          | List security events             |
| GET    | `/api/rules`           | List firewall rules              |
| POST   | `/api/rules`           | Create a firewall rule           |
| PUT    | `/api/rules/:id`       | Update a firewall rule           |
| DELETE | `/api/rules/:id`       | Delete a firewall rule           |
| GET    | `/api/firewall/status` | Get firewall status              |
| POST   | `/api/scans`           | Start an authorized network scan |

---

# Dashboard

The dashboard is designed to remain lightweight.

The primary approach is:

```text
Axum
  │
  ├── HTML
  ├── HTMX
  └── JSON API
```

HTMX allows interactive dashboard functionality without requiring a large JavaScript frontend.

Possible dashboard views include:

* Device inventory
* Network topology
* Service inventory
* Scan history
* Change timeline
* Attack surface
* Security events
* Firewall rules
* Risk findings

A Rust-native WebAssembly frontend using **Leptos** can be introduced later if a richer client-side application becomes useful.

---

# Logging & Observability

NetGuard uses:

* `tracing`
* `tracing-subscriber`

Structured logs provide visibility into:

* Network scans
* Discovery results
* Policy evaluation
* Firewall operations
* Security events
* API requests
* Background tasks
* Errors and failures

Example:

```text
INFO network_scan_started
  interface=eth0
  network=192.168.1.0/24

INFO device_discovered
  ip=192.168.1.42
  mac=AA:BB:CC:DD:EE:FF

WARN policy_violation
  device=192.168.1.42
  destination=192.168.1.20
  port=445
```

---

# Project Structure

```text
netguard/
├── src/
│   ├── api/
│   │   ├── handlers/
│   │   ├── routes/
│   │   └── middleware/
│   │
│   ├── cli/
│   │   └── commands/
│   │
│   ├── core/
│   │   ├── models/
│   │   ├── policies/
│   │   ├── risk/
│   │   ├── events/
│   │   └── traits/
│   │
│   ├── network/
│   │   ├── discovery/
│   │   ├── arp/
│   │   ├── icmp/
│   │   ├── scanning/
│   │   ├── packets/
│   │   └── snapshots/
│   │
│   ├── firewall/
│   │   ├── rules/
│   │   ├── adapters/
│   │   ├── nftables/
│   │   └── enforcement/
│   │
│   ├── persistence/
│   │   ├── models/
│   │   ├── repositories/
│   │   └── migrations/
│   │
│   ├── security/
│   │   ├── detection/
│   │   ├── correlation/
│   │   └── intelligence/
│   │
│   ├── config/
│   ├── telemetry/
│   └── main.rs
│
├── tests/
│   ├── core/
│   ├── network/
│   ├── firewall/
│   ├── persistence/
│   └── integration/
│
├── benches/
│   └── network.rs
│
├── migrations/
├── docs/
├── scripts/
├── docker/
├── .github/
│   └── workflows/
│
├── Cargo.toml
├── Cargo.lock
├── Dockerfile
├── README.md
└── LICENSE
```

---

# Quick Start

## Requirements

* Rust toolchain
* Cargo
* Linux recommended for firewall enforcement
* Git
* A local network for authorized testing

For development:

```bash
rustup update
```

Some network discovery operations require elevated privileges depending on the operating system and interface being used.

Firewall enforcement requires appropriate Linux permissions.

> NetGuard should only be used on networks and systems you own or are explicitly authorized to assess.

---

## Clone

```bash
git clone https://github.com/yourusername/netguard.git
cd netguard
```

---

## Build

```bash
cargo build
```

For an optimized build:

```bash
cargo build --release
```

---

## Run

```bash
cargo run
```

Or run the compiled binary:

```bash
./target/release/netguard
```

---

## CLI

Example:

```bash
cargo run -- scan
```

```bash
cargo run -- devices
```

```bash
cargo run -- services
```

```bash
cargo run -- changes
```

```bash
cargo run -- firewall status
```

---

# Configuration

NetGuard uses structured configuration loaded through the application configuration layer.

Example:

```toml
[network]
scan_interval_seconds = 300
connection_timeout_ms = 1000
monitor_changes = true

[network.discovery]
arp = true
icmp = true
tcp = true

[policy]
default_action = "allow"

[persistence]
database_url = "sqlite://netguard.db"

[logging]
level = "info"
```

Production PostgreSQL configuration can use an environment variable:

```bash
export DATABASE_URL="postgres://user:password@localhost/netguard"
```

Sensitive configuration and database credentials should never be committed to source control.

---

# API

The Axum API exposes network and security information to the dashboard and external clients.

Example:

```text
GET /api/devices
GET /api/devices/:id
GET /api/scans
GET /api/changes
GET /api/events
GET /api/rules
POST /api/rules
PUT /api/rules/:id
DELETE /api/rules/:id
GET /api/firewall/status
POST /api/scans
```

API payloads use `serde` and `serde_json`.

---

# Testing

NetGuard uses Rust's built-in testing infrastructure.

Run the test suite:

```bash
cargo test
```

For coverage:

```bash
cargo tarpaulin
```

Testing focuses particularly on deterministic security behavior.

Example:

```text
Unknown device + TCP/445
        │
        ▼
Policy evaluation
        │
        ▼
DENY
```

And:

```text
Trusted device + HTTPS
        │
        ▼
Policy evaluation
        │
        ▼
ALLOW
```

Firewall integration tests should run inside a controlled environment and should never modify firewall policy on an unintended system.

---

# Benchmarking

Performance-sensitive network operations are benchmarked using:

* `criterion`

Example benchmark areas:

```text
Host discovery
TCP connection scanning
Packet parsing
CIDR processing
Snapshot comparison
Policy evaluation
Rule matching
Serialization
```

The goal is to measure the performance of the underlying components rather than simply reporting application-level throughput.

Example:

```text
Benchmark: snapshot_comparison

1,000 devices
10,000 services

Mean:
...

P95:
...

P99:
...
```

---

# Docker

NetGuard can be containerized using Docker.

Build:

```bash
docker build -t netguard .
```

Run:

```bash
docker run --rm -p 8080:8080 netguard
```

Network discovery and firewall enforcement may require additional container capabilities depending on the deployment environment.

For security-sensitive functionality, host-level or dedicated lab deployment may be preferable to a restricted container.

---

# CI/CD

GitHub Actions is used for automated validation.

The CI pipeline can perform:

```text
cargo fmt --check
        │
        ▼
cargo clippy
        │
        ▼
cargo test
        │
        ▼
cargo build --release
        │
        ▼
Docker build
```

Additional jobs can run coverage and benchmarks where appropriate.

---

# Security Considerations

NetGuard is an **authorization-first defensive tool**.

Use it only against:

* Networks you own
* Systems you administer
* Lab environments
* Environments where you have explicit authorization

NetGuard should not be used to scan or interfere with networks belonging to other people or organizations.

Firewall changes can affect network connectivity. Development and integration testing should therefore be performed inside an isolated or disposable environment whenever possible.

NetGuard does not intentionally collect:

* Passwords
* Private keys
* Authentication secrets

Packet capture functionality, when enabled, should be treated as sensitive because captured network traffic may contain information that users did not intend to expose.

---

# Roadmap

## Phase 1 — Network Engine

* Network interface discovery
* CIDR parsing
* ARP discovery
* ICMP discovery
* TCP service scanning
* Hostname resolution
* Device model
* Async scanning with Tokio
* Raw socket handling
* Network snapshot model

---

## Phase 2 — Network Inventory

* Network snapshots
* Persistent device inventory
* Service history
* New device detection
* Removed device detection
* New service detection
* Removed service detection
* Snapshot comparison
* Historical queries

---

## Phase 3 — Firewall & Policy Engine

* Rule model
* Allow/deny evaluation
* Source/destination matching
* TCP/UDP matching
* Port matching
* Rule priorities
* Enable/disable rules
* Default policy
* Device trust states
* Firewall abstraction
* Linux nftables adapter
* Firewall status
* Rule hit counters

---

## Phase 4 — Security Intelligence

* Security event model
* Blocked connection logging
* Policy violation detection
* Unusual connection detection
* Previously unseen destination detection
* Attack-surface scoring
* Explainable risk findings
* Evidence records
* Alert system
* Optional packet-capture integration

---

## Phase 5 — Web Application

* Axum API
* API documentation
* HTMX dashboard
* Device inventory
* Network topology/inventory view
* Scan history
* Change timeline
* Attack-surface view
* Firewall rules
* Security events
* Risk findings

---

## Phase 6 — Persistence & Deployment

* SQLite development backend
* PostgreSQL production backend
* SQLx migrations
* Docker deployment
* Configuration management
* Production logging
* CI/CD pipeline

---

## Phase 7 — Validation

* Unit test suite
* Integration tests
* Policy evaluation tests
* Firewall integration tests
* Security scenario tests
* Regression tests
* Benchmark suite
* Isolated network test environment
* Failure and recovery testing

---

# Design Principles

### Explainability

Security decisions should have a reason.

```text
BLOCKED

Reason:
Unknown device attempted SMB access
```

rather than simply:

```text
BLOCKED
```

---

### Least Privilege

Unknown devices should not automatically receive the same access as trusted devices.

---

### Defense in Depth

Network discovery, change detection, policy enforcement, and monitoring operate together rather than relying on a single security mechanism.

---

### Historical Evidence

Security decisions should be backed by observable events and historical network state.

---

### Safe by Default

Scanning and firewall operations should require explicit configuration and authorization.

---

### Separation of Concerns

Network observation, security analysis, persistence, API handling, and firewall enforcement should remain independently testable.

---

### Minimal Dependencies

NetGuard should use specialized dependencies where they provide meaningful capabilities while keeping the core architecture understandable.

---

### Performance Awareness

Network operations should be asynchronous and measurable.

Performance-sensitive code should be benchmarked rather than optimized based on assumptions.

---

# What NetGuard Is Not

NetGuard is intentionally **not**:

* A replacement for Nmap
* A vulnerability scanner
* A full IDS/IPS
* An antivirus product
* A SIEM
* A deep-packet-inspection engine
* A commercial enterprise firewall
* A kernel-level packet-filtering implementation

The project deliberately focuses on the intersection of:

```text
Network Visibility
       +
Change Detection
       +
Attack-Surface Monitoring
       +
Policy Enforcement
       +
Security Evidence
```

---

# Technology Summary

| Component         | Technology                        |
| ----------------- | --------------------------------- |
| Language          | Rust                              |
| Async Runtime     | Tokio                             |
| Network Discovery | `pnet`, `socket2`                 |
| Raw Networking    | Raw sockets / packet construction |
| Packet Capture    | `pcap`                            |
| Firewall          | nftables / `nftnl` / `rtnetlink`  |
| CLI               | `clap`                            |
| Serialization     | `serde`, `serde_json`, `bincode`  |
| Persistence       | `sqlx`                            |
| Development DB    | SQLite                            |
| Production DB     | PostgreSQL                        |
| Web API           | Axum                              |
| Dashboard         | Axum + HTMX                       |
| Optional Frontend | Leptos                            |
| Logging           | `tracing`, `tracing-subscriber`   |
| Testing           | `cargo test`                      |
| Coverage          | `cargo tarpaulin`                 |
| Benchmarking      | Criterion                         |
| Containerization  | Docker                            |
| CI/CD             | GitHub Actions                    |

---

# License

MIT License — see [`LICENSE`](LICENSE).

---

# Author

**Keletso Monyamane**

GitHub: `@keletso-m`

---

# Project Status

**Active Development**

NetGuard is a learning and portfolio project focused on developing practical experience with:

* Rust
* Tokio
* Network programming
* Raw packet handling
* Security engineering
* Network discovery
* Policy engines
* Firewall integration
* Persistence
* Async systems
* Observability
* Automated testing
* Performance benchmarking
* Production-oriented software engineering

The project is developed incrementally, with functionality added only after the underlying component is understood and tested.
