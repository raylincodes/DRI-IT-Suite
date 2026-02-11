# Enterprise IT Management Platform — Feature Specification

**Working Document — v2.1 (Lean & Shippable)**
**Date: February 2026**

---

## Target Market

This platform is designed exclusively for **on-premises IT departments** managing their own infrastructure and endpoints. It is not an MSP tool, does not support multi-tenancy, and makes no assumptions about cloud-hosted infrastructure (Azure, AWS, etc.). All data stays on the customer's hardware. The platform is designed to be fully functional in air-gapped environments with zero external dependencies at runtime.

---

## Platform Foundation

### Core Shell Application

- WPF with Lepo WPF UI desktop application (Windows-only)
- Module-based architecture — each feature plugs into shared shell
- Main navigation with module switching
- **Dark theme UI only** (no light mode)
- LiveCharts2 for dashboard visualizations (pie, bar, donut charts)
- ScottPlot for real-time monitoring graphs (CPU, RAM, network over time)

### Shared Infrastructure

- SQL Server database backend (standard relational tables, no graph extensions)
- Raw ADO.NET data layer (SqlConnection/SqlCommand, manual SQL, no ORMs)
- Unified credential vault with DPAPI encryption at rest
- Background service engine (scheduled discovery, scanning, monitoring)
- Real-time UI updates via AppState store pattern with ObservableProperty
- First-run setup wizard (auto-discover AD, configure OUs, set credentials)
- Async/await with `SemaphoreSlim` for concurrent scanning (50-100 machines simultaneously)

### Alerting & Notification Engine

Core platform infrastructure — not a module feature. Every module routes alerts through this shared engine.

- Email notification (SMTP, configurable per-alert severity)
- Webhook support (Teams, Slack, generic HTTP POST)
- In-app dashboard alert feed with acknowledgement tracking
- Per-module alert thresholds and suppression rules
- Scheduled report delivery (compliance reports, digest summaries, audit packages)
- Alert history log with full context per event

### Connectivity Engine

Unified remote execution layer abstracting all protocols with intelligent fallback strategy.

- **Primary transport:** WinRM/PowerShell remoting (System.Management.Automation)
- **Legacy fallback:** WMI/CIM queries (Microsoft.Management.Infrastructure) for bootstrapping WinRM
- **Bootstrapping workflow:**
  1. Attempt WinRM connection first (port 5985/5986)
  2. If WinRM unavailable, fall back to legacy WMI (DCOM/RPC)
  3. Via WMI, execute: `Enable-PSRemoting -Force -SkipNetworkProfileCheck`
  4. Wait 10 seconds for service startup
  5. Retry WinRM connection
  6. Mark machine as "WinRM-enabled" in database for future connections
- SMB file access for file operations
- SSH (for Linux targets)
- SNMP (UDP + ASN.1/BER parsing) for network devices
- Connection pooling and credential management
- Structured result returns per host

---

## V1 Core Modules

The following modules constitute the shippable V1 product. All other modules are deferred to V2.

---

## Module 1 — Agentless RMM

### Remote Execution

- PowerShell remoting across fleet via WinRM (System.Management.Automation)
- Remote command execution via WMI Win32_Process.Create (legacy fallback)
- Script library with version history
- Scheduled automation with configurable intervals
- Approval workflows for sensitive operations (reboot, service restart)
- Job queue with status tracking and result logging
- Real-time PowerShell output displayed in monospaced terminal window

### Endpoint Management

- Remote reboot (force and graceful)
- Remote user logoff
- Remote service start/stop/restart (System.ServiceProcess.ServiceController)
- Remote process kill
- Remote registry read/write (Microsoft.Win32.RegistryKey.OpenRemoteBaseKey)
- Remote software install/uninstall via WMI or PowerShell

### Printer Management

Read-only printer monitoring via WMI (Win32_Printer, Win32_PrintJob).

- Printer status monitoring per endpoint
- Print queue depth and stalled job detection
- Print server health monitoring (Windows Print Spooler service status)

> **V2 Consideration:** Ephemeral one-shot deployment stub for printer driver installation — a self-deleting bootstrapper that installs a driver and removes itself, without leaving a persistent agent.

### Live Endpoint Health Dashboard

Real-time monitoring with ScottPlot line charts showing historical trends.

- Online/offline status via async ICMP ping sweep
- CPU utilization (real-time via WMI perf counters) with 1-hour history graph
- RAM usage with trend visualization
- Disk space and disk health (Win32_DiskDrive status)
- Network adapter throughput (bytes sent/received per sec)
- Battery status for laptops (chassis type detection + Win32_Battery)
- Logged-in user identification (Win32_ComputerSystem.UserName)
- Last boot time / uptime tracking
- Power state monitoring (suspend/resume events)

### Patch Compliance Tracker

- Query Windows Update status across all endpoints via WMI/CIM
- Missing KB identification per machine
- Days-behind-current calculation
- Risk scoring based on CVE severity of missing patches
- Compliance visualization via LiveCharts2 (bar/pie charts, not heatmaps)
- WSUS integration for patch source data
- Exportable compliance reports
- Patch rollout capability deferred to V2

---

## Module 2 — Asset Intelligence

### Agentless Discovery

- WMI/CIM-based hardware and software enumeration
- SNMP device discovery (switches, routers, printers, UPS)
- SSH-based discovery for Linux/network devices
- Continuous scheduled scanning with configurable intervals
- Auto-population of new devices across LAN subnets
- Async parallel scanning (50-100 machines concurrently via SemaphoreSlim)

### Hardware Inventory

- CPU (model, cores, logical processors, clock speed)
- RAM (capacity per DIMM, speed, manufacturer, slot location)
- Disk drives (model, size, media type, health status, SMART data)
- GPU (model, VRAM, driver version)
- Motherboard (manufacturer, product, serial number)
- BIOS/UEFI (manufacturer, version, release date, serial number)
- Network adapters (description, MAC, IP, bandwidth)
- Chassis type (desktop/laptop/server detection)
- Monitor information (model, serial, resolution)
- Peripheral tracking (USB devices connected)

### Software Inventory

- Installed applications enumeration via registry (`HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall` and Wow6432Node equivalent)
- **Win32_Product is explicitly excluded** due to Windows Installer consistency check side effects
- Software version tracking across fleet
- License tracking framework
- Software change detection (new installs, removals, updates between scans)

### Operating System Details

- OS name, version, build number, architecture
- Windows serial number / product key
- Last boot time
- Installed Windows features and roles (on servers)

### Network Configuration

- IP addresses (IPv4/IPv6)
- Subnet masks and gateways
- DNS server configuration
- DHCP vs static assignment detection
- VLAN membership (where detectable via SNMP)

### Live Compliance Posture Dashboard

- Asset age tracking (warranty/lifecycle management)
- Hardware spec compliance (minimum RAM, disk space thresholds)
- OS version compliance (end-of-life detection)
- Fleet-wide asset statistics and visualizations via LiveCharts2

### Audit Evidence Collection

- Point-in-time inventory snapshots
- Change-over-time reports
- Exportable inventory data for auditors

---

## Module 5 — Threat Dashboard

### Windows Event Log Aggregation

- Agentless collection via WinRM (EventLogReader, real-time subscriptions)
- Security, System, Application, PowerShell, Sysmon log collection
- Centralized log storage in SQL Server
- Event correlation across multiple machines

### Syslog Ingestion

- UDP port 514 syslog receiver (RFC 5424 parsing)
- Firewall log ingestion
- Normalized event storage

### Real-Time Threat Visualization

- Failed login attempt tracking with source IP geolocation
- Brute force detection with automatic pattern recognition
- Lateral movement detection (unusual auth patterns between machines)
- Timeline-based event viewer with filtering
- LiveCharts2 visualizations for threat trends (not map-based)

### Detection Rules

- Pre-built detection rules for common attack patterns
- Custom rule creation engine
- Alert severity classification
- Notification via the shared alerting engine (email, webhook, dashboard)

### Event Log Forensic Timeline

- Pull Security, System, PowerShell, Sysmon logs from any machine (agentless)
- Correlate into unified chronological timeline
- Sortable/filterable data grid presentation
- Example: "At 2:14 AM user X authenticated → 2:15 AM PowerShell encoded command → 2:16 AM new service installed"
- Exportable incident timeline for reports

### PowerShell Script Block Collector

- Fleet-wide collection of Event ID 4104 (script block logging)
- Pattern scanning for suspicious content:
  - Encoded commands (base64)
  - Invoke-Mimikatz / Invoke-WebRequest to external URLs
  - `[System.Reflection.Assembly]::Load` calls
  - AMSI bypass patterns
  - Credential harvesting functions
- Alert on detection with full script content
- Results displayed in sortable grid with syntax highlighting

---

## Module 6 — Credential Hygiene & AD Security

**Note:** All visualizations in this module use **sortable data grids and LiveCharts2**, not force-directed graphs. Attack path visualization is deferred to V2.

### Active Directory Security Posture Assessment

- Kerberoastable accounts (SPNs on user accounts)
- AS-REP roastable accounts (no pre-auth required)
- Unconstrained delegation hosts
- Constrained delegation misconfigurations
- Resource-Based Constrained Delegation (RBCD) misconfigurations
- Stale admin accounts (inactive Domain Admins)
- Nested group membership analysis
- AdminSDHolder permission audit
- Service account risk assessment
- Results displayed as sortable/filterable tables with risk scoring

### Stale Object Graveyard

- Computer accounts not authenticated in 90+ days
- User accounts with passwords older than configurable threshold
- Empty security groups
- Orphaned GPO links
- Unused service accounts
- Duplicate SPNs
- Prioritized cleanup dashboard with one-click remediation guidance

### Password Policy Analysis

- Domain password policy assessment
- Fine-grained password policy enumeration
- Password age distribution across all accounts (histogram via LiveCharts2)
- Compliance comparison against NIST/CIS recommendations

### Kerberos Ticket Inspector

- Real-time monitoring of Kerberos auth on DCs (via event logs)
- TGT request tracking
- Service ticket request monitoring
- Failed authentication analysis
- Kerberoasting indicators (anomalous TGS-REQ patterns)
- AS-REP roasting detection
- Dashboard showing authentication activity trends (LiveCharts2)

### Group Policy Security Analysis

- Security-relevant GPO settings inventory
- Comparison against CIS/NIST benchmarks
- Weak settings identification (LM hash storage, NTLMv1 allowed, etc.)
- GPO inheritance analysis with sortable grid showing effective policy per OU

### User Access Control Automation

- Scheduled enable/disable of AD accounts
- Temporary access provisioning with automatic revocation
- Contractor/vendor access lifecycle management
- Audit trail of all access changes
- SQL Server-backed scheduling engine

---

## Module 7 — Deception Engine

**This is the killer differentiator.** Zero false positive detection — if it fires, you have confirmed attacker activity.

### Honey Token Factory

- Auto-generate and deploy deception assets across AD
- Fake service accounts with Kerberoastable SPNs
- Fake admin accounts with enticing names (svc_backup_admin, etc.)
- Dummy credential files (.rdp, passwords.xlsx, creds.txt) on file shares
- All assets are monitored tripwires
- MITRE ATT&CK technique mapping on trigger
- Instant high-confidence alerts via alerting engine

### Canary File Shares

- Create SMB shares with enticing names (`\\server\IT-Passwords`)
- Place believable decoy files
- Monitor access via agentless event log collection (Event ID 5140, 4663)
- Any access = confirmed attacker or insider threat
- Single wizard to deploy shares, create bait, wire monitoring

### DNS Canary

- Plant fake DNS records for non-existent servers
- Enticing names: sql-backup-prod, legacy-vpn-gateway, dev-admin-portal
- Monitor DNS resolution attempts via DNS debug logging
- Any resolution = internal reconnaissance activity
- Agentless detection

### Fake Service Listeners

- Lightweight TCP listeners on unused IPs or unused ports on real servers
- SSH (22), RDP (3389), HTTP (80/443), SQL (1433), SMB (445)
- No real service behind them
- Any connection attempt = attacker scanning or misconfiguration
- Instant high-confidence alert

### Honey Token Management Dashboard

- Centralized view of all deployed deception assets (sortable grid)
- Status monitoring (active, triggered, expired)
- Trigger history with full forensic details
- Health checks to ensure deceptions remain believable
- Rotation scheduling to keep deceptions fresh

---

## Technology Stack

### Primary

- **Language:** C# / .NET 8+
- **UI Framework:** WPF with Lepo WPF UI (Windows-only, dark theme only)
- **Database:** SQL Server (on-premise, standard relational tables)
- **Data Access:** Raw ADO.NET (SqlConnection / SqlCommand, manual SQL)
- **Charts:** LiveCharts2 (dashboards, summary visualizations)
- **Time-Series Graphs:** ScottPlot (real-time monitoring, historical trends)

### Protocols & APIs

- **WinRM/PowerShell:** System.Management.Automation (primary transport)
- **WMI/CIM:** Microsoft.Management.Infrastructure (legacy fallback for bootstrapping)
- **Active Directory:** System.DirectoryServices.Protocols (LDAP queries)
- **Event Logs:** System.Diagnostics.Eventing.Reader
- **Remote Registry:** Microsoft.Win32.RegistryKey.OpenRemoteBaseKey
- **Remote Services:** System.ServiceProcess.ServiceController
- **Raw Sockets:** System.Net.Sockets
- **SNMP:** Custom UDP + ASN.1/BER parsing (or lightweight library)
- **Syslog:** UDP port 514, RFC 5424 parsing
- **Crypto:** System.Security.Cryptography (DPAPI for credential vault)

---

## Architecture Principles

- **Agentless:** Zero persistent software deployment on managed endpoints
- **On-Premise:** All data stays on customer's infrastructure
- **Perpetual License:** No SaaS subscriptions, no recurring costs
- **Air-Gap Ready:** Designed for environments with no internet access
- **Minimal Dependencies:** Avoid third-party libraries where possible
- **Procedural Style:** Explicit, verbose code — no DI containers, no interfaces for abstraction's sake
- **Single Application:** One cohesive WPF shell with pluggable modules
- **Windows-First:** Primary target is Windows enterprise environments
- **Security-Informed:** Defensive tools built with understanding of offensive techniques
- **No Multi-Tenancy:** Single-organization deployment — this is not an MSP tool
- **Performance-First:** Async/await with concurrency controls, efficient SQL queries, minimal UI overhead

---

## Deferred to V2

The following modules are intentionally cut from V1 to ensure a shippable product:

### Module 3 — Network Recon & Visualization
- Force-directed topology graphs
- Visual switch port mappers
- Geographic network layouts
- Cable plant/rack visualizers

**V1 Alternative:** Basic network device inventory via SNMP with sortable grids showing IP, MAC, switch port, VLAN.

### Module 4 — Compliance & Configuration
- CIS/NIST/PCI benchmark scanning
- Configuration drift detection
- GPO conflict detector
- Compliance heatmaps

**V1 Alternative:** Basic GPO inventory and AD security checks (Module 6) provide foundational security posture.

### Module 6 Extensions — AD Attack Path Visualizer
- Force-directed graph rendering of attack paths
- Neo4j graph database
- BloodHound-style path enumeration

**V1 Alternative:** Provide lists of security issues (Kerberoastable accounts, delegation misconfigs) without graphical path visualization.

### Module 8 — MITRE ATT&CK Coverage & Validation
- ATT&CK coverage mapper
- Purple team automator
- Technique validation framework

**V1 Alternative:** Deception Engine (Module 7) provides detection validation for specific techniques.

### Module 9 — Pentest & Security Assessment
- Internal vulnerability scanner
- SMB security assessment
- LLMNR/NBT-NS poisoning detection
- ARP watch

**V1 Alternative:** Basic port scanning and service enumeration via network discovery tools.

### Module 10 — Endpoint Forensics
- USB device audit trail
- Shadow IT detector
- WMI persistence scanner
- Process anomaly baseline
- File integrity monitoring

**V1 Alternative:** Event log analysis (Module 5) provides basic forensic capabilities.

### Module 11 — Certificate & Crypto Management
- Certificate expiration dashboard
- Automated cert scanning across environment

**V1 Alternative:** Manual certificate tracking or third-party cert monitoring tools.

---

## V1 Success Criteria

The V1 release is considered successful if it delivers:

1. ✅ Agentless fleet management (inventory, remote execution, patching)
2. ✅ Centralized security event aggregation (mini-SIEM)
3. ✅ Active Directory security posture assessment
4. ✅ **Deception engine with zero false positive attacker detection**
5. ✅ Professional dark-themed UI with sortable grids and clean visualizations
6. ✅ Fast, responsive async operations (250 machines scanned in 2-3 minutes)
7. ✅ On-premise deployment with zero cloud dependencies
8. ✅ Exportable compliance and audit reports

---

## Development Timeline Estimate

**Total: 20-26 weeks (4-6 months at sustainable pace)**

- WPF shell + navigation: 1-2 weeks
- SQL Server + data layer: 1-2 weeks
- Connectivity engine (WinRM + WMI bootstrapping): 2-3 weeks
- Module 2 (Asset Intelligence): 3-4 weeks
- Module 1 (RMM): 4-6 weeks
- Module 6 (AD Security): 3-4 weeks
- Module 7 (Deception Engine): 2-3 weeks
- Module 5 (Threat Dashboard): 3-4 weeks
- Polish, testing, documentation: 2-3 weeks

---

*This is a living document. Features will be refined and prioritized as development progresses.*
