# NetGuard

**Network visibility, change detection, attack-surface monitoring, and policy enforcement for authorized networks.**

NetGuard is a cross-platform network security application built with **C# and .NET**. It discovers devices and exposed services, maintains network snapshots, detects changes over time, evaluates the network's attack surface, and applies configurable security policies through the host firewall.

The project is designed to be **lightweight, understandable, and explainable** rather than a replacement for tools such as Nmap, Nessus, SIEM platforms, or enterprise firewalls.

> **Discover your network. Track what changes. Understand what is exposed. Control what is allowed.**

---

## Project Goals

NetGuard is built to explore practical network-security engineering using the .NET ecosystem.

The project focuses on:

* Network discovery and service enumeration
* Network inventory and historical snapshots
* Change detection
* Attack-surface monitoring
* Device trust and security policies
* Firewall rule evaluation and enforcement
* Security event collection
* Explainable risk assessment
* REST APIs and web-based monitoring
* Cloud deployment with Azure
* Automated testing and CI/CD

The goal is not to build another Nmap or enterprise firewall.

The goal is to understand how **network visibility, security policy, enforcement, and monitoring fit together into one system.**

---

## Architecture

```text
                         ┌─────────────────────┐
                         │      NetGuard       │
                         └──────────┬──────────┘
                                    │
                ┌───────────────────┼───────────────────┐
                │                   │                   │
                ▼                   ▼                   ▼
        ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
        │    Network    │   │   Security    │   │   Firewall    │
        │    Engine     │   │    Engine     │   │    Engine     │
        └───────┬───────┘   └───────┬───────┘   └───────┬───────┘
                │                   │                   │
        ┌───────┼───────┐     ┌─────┼──────┐      ┌─────┼──────┐
        │       │       │     │     │      │      │     │      │
        ▼       ▼       ▼     ▼     ▼      ▼      ▼     ▼      ▼
    Discovery  Ports  DNS   Trust  Risk  Events  Rules Policy  Status
        │       │       │     │     │      │      │     │      │
        └───────┴───────┴─────┴─────┴──────┴──────┴─────┴──────┘
                                    │
                                    ▼
                           ┌─────────────────┐
                           │   Persistence   │
                           │  EF Core / DB   │
                           └────────┬────────┘
                                    │
                                    ▼
                           ┌─────────────────┐
                           │  ASP.NET Core   │
                           │       API       │
                           └────────┬────────┘
                                    │
                                    ▼
                           ┌─────────────────┐
                           │    Dashboard    │
                           │     Blazor      │
                           └─────────────────┘
```

NetGuard separates **observation** from **enforcement**.

The network engine observes the environment, while the policy and firewall components determine what should be allowed or blocked.

---

# Features

## Network Visibility

NetGuard maintains an inventory of devices and services discovered on the local network.

* Automatic interface discovery
* Network range detection
* Host discovery
* IP address tracking
* MAC address tracking where available
* Hostname resolution
* Vendor identification where available
* TCP service discovery
* Configurable scan ranges
* Periodic scanning
* Device inventory

Example:

```text
NETWORK INVENTORY

192.168.1.1
  Router
  80/tcp    HTTP
  443/tcp   HTTPS

192.168.1.20
  Laptop
  22/tcp    SSH
  631/tcp   IPP

192.168.1.42
  Unknown Device
  80/tcp    HTTP
  445/tcp   SMB
```

---

# Network Snapshots

Every scan can produce a network snapshot.

A snapshot records the observed state of the network at a specific point in time.

```text
Snapshot
────────────────────────────

Time:
2026-08-29 10:42

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
2026-08-29 10:42
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

Changes are stored as events so that network history can be investigated over time.

---

# Attack-Surface Monitoring

NetGuard evaluates discovered services to provide a lightweight view of the network's exposed attack surface.

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

22/tcp      SSH
80/tcp      HTTP
445/tcp     SMB
3389/tcp    RDP

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

For example:

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

### Firewall Integration

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
Host Firewall
    │
    ▼
Network Traffic
```

This keeps the project focused on security policy, orchestration, monitoring, and engineering rather than implementing packet filtering inside the kernel.

---

# Security Events

Firewall activity and network changes can produce security events.

Example:

```text
SECURITY EVENT

Time:
11:32:18

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

Additional policy tests verify rule priority, disabled rules, trust-state changes, repeated violations, and firewall failures.

---

# Technology Stack

## Backend

* **C#**
* **.NET**
* **ASP.NET Core**
* **Entity Framework Core**
* **.NET Worker Services**
* `System.Net`
* TCP/UDP networking
* `async/await`
* `CancellationToken`

## Frontend

* **Blazor**
* HTML
* CSS
* JavaScript where required

## Storage

Development:

* SQLite

Production:

* Azure SQL

## Cloud

* **Microsoft Azure**
* Azure App Service or Container Apps
* Azure SQL
* Azure Blob Storage
* Application Insights
* Azure Key Vault where appropriate

Azure services are introduced progressively rather than being required for local development.

---

# Project Structure

```text
netguard/
├── src/
│   ├── NetGuard.Api/
│   │   ├── Controllers/
│   │   ├── Services/
│   │   └── Program.cs
│   │
│   ├── NetGuard.Core/
│   │   ├── Models/
│   │   ├── Policies/
│   │   ├── Risk/
│   │   └── Interfaces/
│   │
│   ├── NetGuard.Network/
│   │   ├── Discovery/
│   │   ├── Scanning/
│   │   ├── Interfaces/
│   │   └── Snapshots/
│   │
│   ├── NetGuard.Firewall/
│   │   ├── Rules/
│   │   ├── Adapters/
│   │   └── Enforcement/
│   │
│   ├── NetGuard.Persistence/
│   │   ├── DbContext/
│   │   ├── Configurations/
│   │   └── Migrations/
│   │
│   └── NetGuard.Web/
│       ├── Components/
│       ├── Pages/
│       └── Services/
│
├── tests/
│   ├── NetGuard.Core.Tests/
│   ├── NetGuard.Network.Tests/
│   ├── NetGuard.Firewall.Tests/
│   └── NetGuard.IntegrationTests/
│
├── docs/
├── scripts/
├── docker/
├── .github/
│   └── workflows/
├── NetGuard.sln
├── Dockerfile
├── README.md
└── LICENSE
```

---

# Quick Start

## Requirements

* .NET SDK
* Linux, Windows, or macOS
* Git
* A local network for authorized testing

For firewall enforcement, some features may require appropriate operating-system permissions.

> NetGuard should only be used on networks and systems you own or are explicitly authorized to assess.

---

## Clone

```bash
git clone https://github.com/yourusername/netguard.git
cd netguard
```

## Restore dependencies

```bash
dotnet restore
```

## Build

```bash
dotnet build
```

## Run

```bash
dotnet run --project src/NetGuard.Api
```

Start the web application:

```bash
dotnet run --project src/NetGuard.Web
```

---

# Configuration

Example configuration:

```json
{
  "NetGuard": {
    "ScanIntervalMinutes": 5,
    "ConnectionTimeoutMilliseconds": 1000,
    "DefaultPolicy": "Allow",
    "MonitorChanges": true
  }
}
```

Sensitive configuration such as credentials and cloud secrets should not be committed to source control.

Local development secrets should use the appropriate .NET development secret mechanisms or environment variables.

---

# API

The ASP.NET Core API exposes network and security information to the dashboard.

Example endpoints:

| Method | Endpoint               | Description                      |
| ------ | ---------------------- | -------------------------------- |
| GET    | `/api/devices`         | List discovered devices          |
| GET    | `/api/devices/{id}`    | Get device details               |
| GET    | `/api/scans`           | List network snapshots           |
| GET    | `/api/changes`         | List detected changes            |
| GET    | `/api/events`          | List security events             |
| GET    | `/api/rules`           | List firewall rules              |
| POST   | `/api/rules`           | Create a firewall rule           |
| PUT    | `/api/rules/{id}`      | Update a firewall rule           |
| DELETE | `/api/rules/{id}`      | Delete a firewall rule           |
| GET    | `/api/firewall/status` | Get firewall status              |
| POST   | `/api/scans`           | Start an authorized network scan |

The API documentation is available through ASP.NET Core's OpenAPI/Swagger tooling during development.

---

# Testing

NetGuard uses automated tests for the policy engine, network state handling, change detection, and API behavior.

Run the test suite:

```bash
dotnet test
```

Testing focuses particularly on deterministic security behavior.

Example:

```text
Unknown device + TCP/445
        ↓
Policy evaluation
        ↓
DENY
```

and:

```text
Trusted device + HTTPS
        ↓
Policy evaluation
        ↓
ALLOW
```

Firewall integration tests should use a controlled test environment and should never modify firewall policy on an unintended system.

---

# Azure Deployment

NetGuard is designed to support deployment to Microsoft Azure.

A production deployment can use:

```text
                    Azure
                      │
          ┌───────────┼───────────┐
          │           │           │
          ▼           ▼           ▼
     App Service   Azure SQL   Blob Storage
          │           │           │
          └───────────┼───────────┘
                      │
                      ▼
              Application Insights
```

Potential Azure components include:

### Azure App Service / Container Apps

Hosts the ASP.NET Core application.

### Azure SQL

Stores:

* Devices
* Network snapshots
* Services
* Security events
* Firewall policies
* Risk findings

### Azure Blob Storage

Can store generated reports and exported historical data.

### Application Insights

Provides:

* Application telemetry
* Request monitoring
* Exceptions
* Performance data

### Key Vault

Used for production secrets and credentials where required.

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

NetGuard does not intentionally collect passwords, private keys, or other authentication secrets.

---

# Roadmap

## Phase 1 — Network Engine

* [ ] Network interface discovery
* [ ] CIDR parsing
* [ ] Host discovery
* [ ] TCP service scanning
* [ ] Hostname resolution
* [ ] Device model

## Phase 2 — Network Inventory

* [ ] Network snapshots
* [ ] Persistent device inventory
* [ ] Service history
* [ ] New device detection
* [ ] Removed device detection
* [ ] New service detection
* [ ] Removed service detection
* [ ] Snapshot comparison

## Phase 3 — Firewall & Policy Engine

* [ ] Rule model
* [ ] Allow/deny evaluation
* [ ] Source/destination matching
* [ ] TCP/UDP matching
* [ ] Port matching
* [ ] Rule priorities
* [ ] Enable/disable rules
* [ ] Default policy
* [ ] Device trust states
* [ ] Firewall abstraction
* [ ] Linux firewall adapter
* [ ] Firewall status
* [ ] Rule hit counters

## Phase 4 — Security Intelligence

* [ ] Security event model
* [ ] Blocked connection logging
* [ ] Policy violation detection
* [ ] Unusual connection detection
* [ ] Previously unseen destination detection
* [ ] Attack-surface scoring
* [ ] Explainable risk findings
* [ ] Evidence records
* [ ] Alert system

## Phase 5 — Web Application

* [ ] ASP.NET Core API
* [ ] OpenAPI documentation
* [ ] Blazor dashboard
* [ ] Device inventory
* [ ] Network topology/inventory view
* [ ] Scan history
* [ ] Change timeline
* [ ] Attack-surface view
* [ ] Firewall rules
* [ ] Security events
* [ ] Risk findings

## Phase 6 — Azure

* [ ] Azure deployment
* [ ] Azure SQL
* [ ] Application Insights
* [ ] Blob Storage
* [ ] Secure configuration
* [ ] CI/CD pipeline

## Phase 7 — Validation

* [ ] Unit test suite
* [ ] Integration tests
* [ ] Policy evaluation tests
* [ ] Firewall integration tests
* [ ] Security scenario tests
* [ ] Regression tests
* [ ] Isolated network test environment

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

### Least Privilege

Unknown devices should not automatically receive the same access as trusted devices.

### Defense in Depth

Network discovery, change detection, policy enforcement, and monitoring operate together rather than relying on a single security mechanism.

### Historical Evidence

Security decisions should be backed by observable events and historical network state.

### Safe by Default

Scanning and firewall operations should require explicit configuration and authorization.

### Cross-Platform

The core application should remain portable across supported operating systems, while OS-specific firewall functionality is isolated behind platform-specific adapters.

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

# License

MIT License — see [`LICENSE`](LICENSE).

---

# Author

**Keletso Monyamane**

GitHub: `@keletso-m`

---

## Project Status

**Active Development**

NetGuard is a learning and portfolio project focused on developing practical experience with:

* C#
* .NET
* ASP.NET Core
* Network programming
* Security engineering
* Policy engines
* Firewall integration
* Distributed/cloud application architecture
* Azure
* Automated testing
* Production-oriented software engineering

The project is developed incrementally, with functionality added only after the underlying component is understood and tested.
