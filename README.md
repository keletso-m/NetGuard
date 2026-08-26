# MyNetScanner

**Network visibility, change detection, and attack-surface monitoring for your local network.**

MyNetWatcher is a cross-platform network monitoring tool built with **C# and .NET**. It discovers devices on a network, identifies exposed services, records network snapshots, detects changes between scans, and provides a simple view of the network's current attack surface.

The project is designed to be lightweight and understandable rather than a replacement for large tools such as Nmap, Nessus, or a full SIEM.

> **Discover your network. Track what changes. Understand what is exposed.**

---

## Features

### Network Discovery

MyNetWatcher can inspect a local network and identify devices that are currently reachable.

* Detect local network interfaces
* Automatically determine network ranges
* Discover active hosts
* Resolve hostnames where available
* Identify MAC addresses where the platform permits it
* Identify device vendors using MAC/OUI information

### Port & Service Discovery

For discovered hosts, MyNetWatcher can inspect TCP ports and record detected services.

Example:

```text
192.168.1.1
├── 53/tcp    DNS
├── 80/tcp    HTTP
└── 443/tcp   HTTPS

192.168.1.20
├── 22/tcp    SSH
├── 80/tcp    HTTP
└── 445/tcp   SMB
```

The initial version focuses on practical TCP discovery rather than attempting to reproduce the functionality of a full vulnerability scanner.

### Network Snapshots

Each scan produces a snapshot of the observed network.

```text
Snapshot #12
────────────────────────────
Devices:       17
Open ports:    43
Scan duration: 12.4 seconds
```

Snapshots allow MyNetWatcher to understand how the network changes over time.

### Change Detection

MyNetWatcher compares the current snapshot against previous observations.

It can identify events such as:

```text
NEW DEVICE
192.168.1.47

NEW PORT
192.168.1.20:8080

CLOSED PORT
192.168.1.32:22

DEVICE OFFLINE
192.168.1.10
```

This turns a simple network scanner into a monitoring tool.

### Attack Surface

MyNetWatcher provides a lightweight risk view based on the services discovered during scanning.

Example:

```text
ATTACK SURFACE
────────────────────────

HIGH       1
MEDIUM     4
LOW       12
```

Initial rules are intentionally simple and explainable.

For example:

```text
23/tcp     Telnet       HIGH
445/tcp    SMB          MEDIUM
3389/tcp   RDP          MEDIUM
22/tcp     SSH          LOW
80/tcp     HTTP         LOW
```

The project does **not** attempt to perform exploitation or comprehensive vulnerability assessment.

---

# Dashboard

The planned dashboard provides a high-level view of the network.

```text
┌───────────────────────────────────────────────────────┐
│ MyNetWatcher                                           │
├───────────────────────────────────────────────────────┤
│                                                       │
│  NETWORK OVERVIEW                                     │
│                                                       │
│  Devices        17       Open Ports       43          │
│  Online         15       Changes           3          │
│                                                       │
├───────────────────────────────────────────────────────┤
│                                                       │
│  RECENT CHANGES                                       │
│                                                       │
│  + New device       192.168.1.47                      │
│  + New port         192.168.1.20:8080                 │
│  - Closed port      192.168.1.32:22                   │
│                                                       │
├───────────────────────────────────────────────────────┤
│                                                       │
│  ATTACK SURFACE                                       │
│                                                       │
│  HIGH        █                                         │
│  MEDIUM      ████                                      │
│  LOW         ████████████                              │
│                                                       │
└───────────────────────────────────────────────────────┘
```

The UI will be developed after the core scanning and snapshot functionality is working.

---

# Architecture

MyNetWatcher is divided into separate components so that the network engine is independent from the web interface.

```text
                         MyNetWatcher
                              │
                ┌─────────────┴─────────────┐
                │                           │
          Network Engine              Web Application
                │                           │
        ┌───────┼────────┐            ┌─────┴─────┐
        │       │        │            │           │
     Discovery  TCP    Snapshot     ASP.NET     Blazor
                         Engine       Core
                           │            │
                           └──────┬─────┘
                                  │
                              Database
```

The intended project structure is:

```text
MyNetWatcher/
│
├── src/
│   ├── MyNetWatcher.Core/
│   ├── MyNetWatcher.Scanner/
│   ├── MyNetWatcher.Risk/
│   ├── MyNetWatcher.Infrastructure/
│   ├── MyNetWatcher.Api/
│   └── MyNetWatcher.Web/
│
├── tests/
│   ├── MyNetWatcher.Core.Tests/
│   ├── MyNetWatcher.Scanner.Tests/
│   └── MyNetWatcher.Risk.Tests/
│
├── docs/
│
└── README.md
```

### Core

Contains the domain models and application logic.

Examples:

```text
Network
Device
Port
Service
Scan
Snapshot
Change
Finding
```

### Scanner

Responsible for interacting with the local network.

```text
Network interfaces
CIDR ranges
Host discovery
TCP connections
DNS resolution
MAC addresses
```

### Risk

Contains the rules used to evaluate the observed attack surface.

Rules should remain deterministic and explainable.

### Infrastructure

Handles persistence and external infrastructure.

The initial development environment will use a local database, with Azure services introduced later.

### API

ASP.NET Core API responsible for exposing scans, devices, snapshots, changes, and findings.

### Web

The browser-based dashboard.

---

# Technology Stack

## Core

* **C#**
* **.NET**
* .NET Worker Services
* `System.Net`
* TCP/UDP sockets
* Async/await
* Cancellation tokens

## Backend

* **ASP.NET Core**
* Entity Framework Core
* REST API
* Dependency Injection
* Background services

## Frontend

* **Blazor**
* HTML/CSS
* JavaScript where necessary

## Database

Development:

* SQLite

Production/cloud:

* Azure SQL or PostgreSQL

## Azure

Azure will be introduced as the project develops.

Planned services include:

* Azure App Service or Azure Container Apps
* Azure SQL
* Azure Blob Storage
* Azure Application Insights
* Azure Key Vault
* Azure Service Bus

Not every service is required for the initial release.

## Development

* Linux
* Git
* GitHub
* Docker
* GitHub Actions

---

# How It Works

A typical scan follows this process:

```text
1. Detect local interfaces
          │
          ▼
2. Determine target network
          │
          ▼
3. Discover active hosts
          │
          ▼
4. Identify host information
          │
          ▼
5. Scan selected TCP ports
          │
          ▼
6. Identify known services
          │
          ▼
7. Create network snapshot
          │
          ▼
8. Compare with previous snapshot
          │
          ▼
9. Generate changes
          │
          ▼
10. Evaluate attack surface
          │
          ▼
11. Store results
```

The important distinction is that the scanner does not simply return a list of open ports.

It produces **state**.

That state can then be compared over time.

---

# Example

Suppose the first scan discovers:

```text
192.168.1.20

22/tcp
80/tcp
443/tcp
```

The next scan discovers:

```text
192.168.1.20

22/tcp
80/tcp
443/tcp
8080/tcp
```

MyNetWatcher calculates:

```text
CHANGE DETECTED

Host:
192.168.1.20

Change:
Port opened

Port:
8080/tcp

Previous:
Closed

Current:
Open
```

The risk engine can then evaluate the newly exposed service.

---

# Azure Architecture

Once the local version is stable, MyNetWatcher can be extended into a cloud-backed architecture.

```text
                       Azure
                         │
                 ┌───────▼────────┐
                 │  ASP.NET Core  │
                 │      API       │
                 └───────┬────────┘
                         │
             ┌───────────┼───────────┐
             │           │           │
             ▼           ▼           ▼
        Azure SQL     Blob       App Insights
                      Storage
                         │
                         │
                    Scan Reports
```

A future version can support remote agents:

```text
                         Azure
                           │
                    ┌──────▼──────┐
                    │ Control API │
                    └──────┬──────┘
                           │
                      Service Bus
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
           Agent A      Agent B      Agent C
            Linux        Linux       Windows
              │            │            │
             LAN A        LAN B        LAN C
```

This allows MyNetWatcher to eventually monitor multiple networks without requiring the Azure service itself to have direct access to those networks.

---

# Roadmap

## Phase 1 — Network Engine

* [ ] Detect local network interfaces
* [ ] Parse CIDR ranges
* [ ] Discover active hosts
* [ ] Resolve hostnames
* [ ] Discover MAC addresses where available
* [ ] Scan TCP ports
* [ ] Identify common services
* [ ] Build command-line interface

## Phase 2 — Snapshots

* [ ] Create scan snapshots
* [ ] Persist scan results
* [ ] Introduce SQLite
* [ ] Compare snapshots
* [ ] Detect new devices
* [ ] Detect removed devices
* [ ] Detect opened ports
* [ ] Detect closed ports

## Phase 3 — Risk Analysis

* [ ] Create rule-based risk engine
* [ ] Categorize common services
* [ ] Add severity levels
* [ ] Explain why a finding was generated
* [ ] Add attack-surface summary

## Phase 4 — Web Dashboard

* [ ] ASP.NET Core API
* [ ] Blazor dashboard
* [ ] Device list
* [ ] Device details
* [ ] Scan history
* [ ] Change history
* [ ] Attack-surface dashboard
* [ ] Filtering and sorting

## Phase 5 — Azure

* [ ] Containerize the application
* [ ] Deploy API to Azure
* [ ] Move persistent data to Azure SQL
* [ ] Add Blob Storage for reports
* [ ] Add Application Insights
* [ ] Add secure configuration with Key Vault
* [ ] Set up GitHub Actions deployment

## Future

* [ ] Scheduled scans
* [ ] Notifications
* [ ] Real-time updates with SignalR
* [ ] Remote monitoring agents
* [ ] Multi-network support
* [ ] CSV/JSON/PDF reports
* [ ] Authentication and role-based access

---

# Security & Responsible Use

MyNetWatcher is intended for monitoring networks that you own or have explicit authorization to assess.

Network scanning can generate traffic and may trigger security controls or alerts.

Use the software responsibly and only against authorized systems.

MyNetWatcher is designed as a **visibility and monitoring tool**, not an exploitation framework.

It does not intentionally attempt to:

* exploit discovered services
* bypass authentication
* obtain credentials
* compromise systems
* evade security controls

---

# Development

Clone the repository:

```bash
git clone https://github.com/<your-username>/MyNetWatcher.git
cd MyNetWatcher
```

Restore dependencies:

```bash
dotnet restore
```

Build:

```bash
dotnet build
```

Run tests:

```bash
dotnet test
```

Run the application:

```bash
dotnet run
```

> The exact commands and project startup instructions will be updated as the implementation develops.

---

# Project Goals

MyNetScanner is primarily a learning and portfolio project focused on understanding how networking, backend development, security concepts, and cloud infrastructure fit together.

The main goals are to gain practical experience with:

* Network programming in C#
* Asynchronous .NET applications
* ASP.NET Core
* Entity Framework Core
* Background workers
* State comparison and event detection
* Security-oriented rule engines
* REST APIs
* Blazor
* Azure
* Docker
* CI/CD
* Automated testing
* Application observability

The project intentionally favors a **small, understandable architecture** over unnecessary complexity.

---

# License

This project will be released under the **MIT License**.
