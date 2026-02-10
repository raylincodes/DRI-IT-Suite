# Enterprise IT Management Platform — Feature Specification

**Working Document — v2**
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
- Dark theme UI
- SkiaSharp integration for custom rendering (graphs, heatmaps, floor plans)

### Shared Infrastructure

- SQL Server database backend
- Raw ADO.NET data layer (SqlConnection/SqlCommand, manual SQL, no ORMs)
- Unified credential vault with DPAPI encryption at rest
- Background service engine (scheduled discovery, scanning, monitoring)
- Real-time UI updates via AppState store pattern with ObservableProperty
- First-run setup wizard (auto-discover AD, configure OUs, set credentials)

### Alerting & Notification Engine

Core platform infrastructure — not a module feature. Every module routes alerts through this shared engine.

- Email notification (SMTP, configurable per-alert severity)
- Webhook support (Teams, Slack, generic HTTP POST)
- In-app dashboard alert feed with acknowledgement tracking
- Per-module alert thresholds and suppression rules
- Scheduled report delivery (compliance reports, digest summaries, audit packages)
- Alert history log with full context per event

### Connectivity Engine

- Unified remote execution layer abstracting all protocols
- WinRM remoting
- WMI/CIM queries (Microsoft.Management.Infrastructure)
- SMB file access
- SSH (for Linux targets)
- SNMP (UDP + ASN.1/BER parsing)
- Connection pooling and credential management
- Structured result returns per host

---

## Module 1 — Agentless RMM (ConnectWise Automate Alternative)

> **Note:** WinRM/WMI bootstrapping is a first-class concern. The connectivity engine must handle the case of using legacy WMI to enable modern WinRM on endpoints that haven't been configured yet. Decision pending on whether PS remoting is required or whether WMI-only execution is sufficient for all operations.

### Remote Execution

- PowerShell remoting across fleet (System.Management.Automation)
- Remote command execution via WMI Win32_Process.Create
- Script library with version history
- Scheduled automation with configurable intervals
- Approval workflows for sensitive operations (reboot, service restart)
- Job queue with status tracking and result logging

### Endpoint Management

- Remote reboot (force and graceful)
- Remote user logoff
- Remote service start/stop/restart (System.ServiceProcess.ServiceController)
- Remote process kill
- Remote registry read/write (Microsoft.Win32.RegistryKey.OpenRemoteBaseKey)
- Remote software install/uninstall via WMI or PowerShell

### Printer Management

Read-only printer monitoring via WMI (Win32_Printer, Win32_PrintJob). Remote printer driver installation is not supported on modern Windows following PrintNightmare mitigations.

- Printer status monitoring per endpoint
- Print queue depth and stalled job detection
- Print server health monitoring (Windows Print Spooler service status)

> **v2 Consideration:** Ephemeral one-shot deployment stub for printer driver installation — a self-deleting bootstrapper that installs a driver and removes itself, without leaving a persistent agent. This is architecturally compatible with the agentless model and worth revisiting after v1 ships.

### Live Endpoint Health Dashboard

- Online/offline status via ICMP ping sweep
- CPU utilization (real-time via WMI perf counters)
- RAM usage
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
- Compliance heatmap visualization
- WSUS integration for patch source data
- Exportable compliance reports
- Patch rollout capability is a future consideration pending feasibility assessment

---

## Module 2 — Asset Intelligence (Lansweeper Alternative)

### Agentless Discovery

- WMI/CIM-based hardware and software enumeration
- SNMP device discovery (switches, routers, printers, UPS)
- SSH-based discovery for Linux/network devices
- Continuous scheduled scanning with configurable intervals
- Auto-population of new devices across LAN subnets

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

- Installed applications enumeration via registry (HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall and Wow6432Node equivalent) — Win32_Product is explicitly excluded due to Windows Installer consistency check side effects
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
- Fleet-wide asset statistics and visualizations

### Audit Evidence Collection

- Point-in-time inventory snapshots
- Change-over-time reports
- Exportable inventory data for auditors

---

## Module 3 — Network Recon & Visualization

### Real-Time Interactive Topology Mapper

- Force-directed graph or geographic layout
- Auto-discovery of network topology via SNMP, ARP, CDP/LLDP
- Device type classification (switch, router, server, workstation, printer)
- Connection mapping (which port connects to what)
- VLAN visualization with color coding
- Click-through to device details

### Switch Port Mapper

- Walk managed switches via SNMP
- Map every port to connected device (MAC table + ARP + reverse DNS + AD lookup)
- Visual representation of each switch with color-coded ports
- Port states: active, inactive, PoE-enabled, trunked, VLAN-tagged, err-disabled
- Cable trace from endpoint to switch to patch panel

### WiFi Network Monitor

- Connected device enumeration from wireless access points (SNMP where supported)
- Per-device bandwidth utilization
- Rogue access point detection
- SSID inventory across the facility
- Client association history

### DHCP Lease Visualizer

- Pull lease data from Windows DHCP servers
- Scope utilization heatmaps
- Rogue device detection (got lease but not in AD)
- Device subnet movement tracking over time
- Lease expiration monitoring

### Cable Plant / Rack Visualizer

- Define server rooms, racks, patch panels, switch ports
- Visual cable path: endpoint → wall jack → patch panel → switch port
- Documentation for cable runs
- Audit-ready infrastructure diagrams

---

## Module 4 — Compliance & Configuration

### CIS/NIST/PCI Benchmark Scanning

- Automated checks against CIS Benchmarks for Windows
- NIST 800-53 control mapping
- PCI-DSS requirement checks
- Custom benchmark creation
- Per-machine compliance scoring

### Configuration Drift Detection

- Baseline configuration capture
- Periodic comparison against baseline
- Alert on unauthorized configuration changes
- Drift visualization over time

### GPO State Management

- GPO inventory and documentation
- Before/after snapshot comparison
- Effective policy calculation per OU
- GPO change tracking with history

### GPO Conflict Detector

- Scan all Group Policy Objects
- Identify conflicting settings (two GPOs setting the same registry value differently)
- Show effective policy per OU with visual inheritance tree
- Highlight security-relevant settings deviating from CIS benchmarks
- Precedence and filtering analysis (WMI filters, security filtering, enforcement)

### Compliance Heatmaps

- Organization-wide compliance score visualization
- Drill-down from org level → OU → individual machine
- Trend tracking over time
- Color-coded risk grading

### Automated Audit Evidence

- Scheduled compliance report generation
- Evidence package creation (screenshots, configs, scan results)
- Timestamped proof of compliance state
- Export in auditor-friendly formats

---

## Module 5 — Threat Dashboard (SIEM for Small IT Teams)

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

- Geolocated failed login attempts on world map
- Brute force detection with automatic pattern recognition
- Lateral movement visualization (unusual auth patterns between machines)
- Timeline-based event viewer with filtering

### Detection Rules

- Pre-built detection rules for common attack patterns
- Custom rule creation engine
- Alert severity classification
- Notification via the shared alerting engine (email, webhook, dashboard)

### Event Log Forensic Timeline

- Pull Security, System, PowerShell, Sysmon logs from any machine (agentless)
- Correlate into unified chronological timeline
- Visual storyboard rendering
- "At 2:14 AM user X authenticated → 2:15 AM PowerShell encoded command → 2:16 AM new service installed"
- Exportable incident timeline for reports

### PowerShell Script Block Collector

- Fleet-wide collection of Event ID 4104 (script block logging)
- Pattern scanning for suspicious content:
  - Encoded commands (base64)
  - Invoke-Mimikatz / Invoke-WebRequest to external URLs
  - [System.Reflection.Assembly]::Load calls
  - AMSI bypass patterns
  - Credential harvesting functions
- Alert on detection with full script content

---

## Module 6 — Credential Hygiene & AD Security

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
- Password age distribution across all accounts
- Compliance comparison against NIST/CIS recommendations

### Kerberos Ticket Inspector

- Real-time monitoring of Kerberos auth on DCs (via event logs)
- TGT request tracking
- Service ticket request monitoring
- Failed authentication analysis
- Kerberoasting indicators (anomalous TGS-REQ patterns)
- AS-REP roasting detection
- Visual dashboard of authentication activity

### AD Attack Path Visualizer (Defensive BloodHound)

- LDAP enumeration of AD relationships (same data SharpHound collects)
- Graph rendering of privilege escalation paths (SkiaSharp)
- Identify shortest path: standard user → Domain Admin
- Highlight top 5 worst attack paths with remediation steps
- Neo4j backend for graph traversal — embedded as a local service alongside SQL Server
- Framed entirely from defensive perspective
- Risk-scored findings with prioritized recommendations

### Group Policy Security Analysis

- Security-relevant GPO settings inventory
- Comparison against CIS/NIST benchmarks
- Weak settings identification (LM hash storage, NTLMv1 allowed, etc.)
- GPO inheritance visualization with effective policy calculation

### User Access Control Automation

- Scheduled enable/disable of AD accounts
- Temporary access provisioning with automatic revocation
- Contractor/vendor access lifecycle management
- Audit trail of all access changes
- SQL Server-backed scheduling engine

---

## Module 7 — Deception Engine

### Honey Token Factory

- Auto-generate and deploy deception assets across AD
- Fake service accounts with Kerberoastable SPNs
- Fake admin accounts with enticing names (svc_backup_admin, etc.)
- Dummy credential files (.rdp, passwords.xlsx, creds.txt) on file shares
- All assets are monitored tripwires
- MITRE ATT&CK technique mapping on trigger
- Zero false positive detection — if it fires, you have an attacker

### Canary File Shares

- Create SMB shares with enticing names (\\server\IT-Passwords)
- Place believable decoy files
- Monitor access via agentless event log collection
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

- Centralized view of all deployed deception assets
- Status monitoring (active, triggered, expired)
- Trigger history with full forensic details
- Health checks to ensure deceptions remain believable
- Rotation scheduling to keep deceptions fresh

---

## Module 8 — MITRE ATT&CK Coverage & Validation

### ATT&CK Coverage Mapper

- Scan environment for current detection capabilities:
  - Which event logs are being collected
  - Which audit policies are enabled
  - PowerShell logging configuration
  - Sysmon rules deployment status
- Map against full MITRE ATT&CK Windows matrix
- Visual heatmap: green (detectable), yellow (partial), red (blind)
- Per-technique remediation: exact GPO settings, audit policies, and log subscriptions to enable
- Track coverage improvement over time

### Purple Team Automator

- Execute safe versions of common attack techniques against own environment
- Verify whether defenses detect each technique
- Test categories:
  - Did PowerShell script block logging catch the encoded command?
  - Did event log capture the lateral movement?
  - Did Kerberoasting attempt generate an alert?
  - Did the honey token get tripped?
- Push-button validation with pass/fail report per technique
- Atomic Red Team-style testing integrated into platform
- Scheduled recurring validation

---

## Module 9 — Pentest & Security Assessment

### Internal Vulnerability Scanner

- Port scan own fleet
- Enumerate exposed services per machine
- Identify machines running SMBv1
- Check for missing patches on exposed services
- Flag RDP exposed without Network Level Authentication
- Find open admin shares (C$, ADMIN$)
- Detect default/weak service configurations

### SMB Security Assessment

- Check SMB signing status across fleet (disabled = relay attack surface)
- Identify machines vulnerable to SMB relay
- NTLMv1 vs NTLMv2 enforcement check
- SMB encryption status
- Risk scoring per machine

### LLMNR/NBT-NS Poisoning Detection

- Check if LLMNR and NBT-NS are disabled fleet-wide
- Monitor for unexpected LLMNR responses (Responder detection)
- Alert on machines with poisoning protocols still enabled
- Remediation guidance (GPO settings to disable)

### ARP Watch / Rogue Device Detector

- Passive ARP traffic monitoring
- New device appearance detection
- ARP spoofing / MITM attack detection
- MAC address conflict identification
- Cross-reference with AD and DHCP data
- Flag devices with static IPs not in any inventory

---

## Module 10 — Endpoint Forensics

### USB Device Audit Trail

- Query registry and event logs across fleet (agentless)
- Complete history of every USB device ever connected to every machine
- Timestamps, device serial numbers, user context
- Device type classification (storage, HID, printer, etc.)
- DLP compliance reporting
- Exportable audit evidence

### Shadow IT Detector

- Scan installed software across all endpoints via registry enumeration
- Compare against approved software whitelist
- Flag unapproved installations with install dates and user context
- Department-level trend tracking
- Periodic automated scanning with alerting

### WMI Persistence Scanner

- Agentless scan for rogue WMI event subscriptions (attacker persistence)
- Suspicious scheduled tasks detection
- Services with unusual binary paths
- Run/RunOnce registry keys pointing to non-standard locations
- Startup folder anomalies
- Agentless autoruns equivalent for entire fleet

### Process Anomaly Baseline

- Periodic process snapshots across fleet via WMI
- Build baseline of "normal" processes per machine role
- Alert on anomalies:
  - Known offensive tools (mimikatz, psexec, rubeus, etc.)
  - Unsigned processes running from temp directories
  - Processes with suspicious parent-child relationships
  - New services that weren't in previous baseline

### File Integrity Monitor

- Watch critical system files, config files, registry keys across fleet
- Compute and store cryptographic hashes
- Alert on unauthorized modification with before/after diff
- PCI-DSS FIM requirement compliance
- Configurable watch lists per machine role (DC, file server, workstation)

---

## Module 11 — Certificate & Crypto Management

### Certificate Expiration Dashboard

- Scan environment for SSL/TLS certificates:
  - Web servers (IIS bindings)
  - LDAPS on domain controllers
  - RDP certificates
  - SQL Server certificates
  - SMTP/Exchange certificates
- Countdown timers per certificate
- Alerting at configurable thresholds (90, 60, 30, 7 days) via shared alerting engine
- Certificate chain validation
- Single dashboard view of all internal certificate health

---

## Technology Stack

### Primary

- **Language:** C# / .NET 8+
- **UI Framework:** WPF with Lepo WPF UI (Windows-only)
- **Database:** SQL Server (on-premise) + Neo4j (embedded, for AD graph traversal in Module 6)
- **Data Access:** Raw ADO.NET (SqlConnection / SqlCommand)

### Protocols & APIs

- **WMI/CIM:** Microsoft.Management.Infrastructure
- **PowerShell:** System.Management.Automation
- **Active Directory:** System.DirectoryServices.Protocols (LDAP)
- **Event Logs:** System.Diagnostics.Eventing.Reader
- **Remote Registry:** Microsoft.Win32.RegistryKey.OpenRemoteBaseKey
- **Remote Services:** System.ServiceProcess.ServiceController
- **Raw Sockets:** System.Net.Sockets
- **Packet Capture:** P/Invoke to Npcap or SharpPcap
- **SNMP:** Custom UDP + ASN.1/BER parsing (or lightweight library)
- **Syslog:** UDP port 514, RFC 5424 parsing
- **Crypto:** System.Security.Cryptography

### Rendering

- **Custom Visualizations:** SkiaSharp (2D canvas, hardware acceleration)
- **Network Graphs:** Force-directed layout engine
- **AD Attack Graph:** Neo4j + SkiaSharp rendering
- **Charts & Heatmaps:** Custom SkiaSharp rendering

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

---

*This is a living document. Features will be refined and prioritized as development progresses.*
