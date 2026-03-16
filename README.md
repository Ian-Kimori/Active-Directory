# Active-Directory

# Active Directory

**Complete Security Engineer Reference**

*Setup → Architecture → DNS → LDAP → Kerberos → NTDS.dit → OUs → GPO → Replication → SYSVOL*

---

## 0. What Active Directory Is

Active Directory Domain Services (AD DS) is Microsoft's on-prem identity platform. It is simultaneously many things:

- An LDAP-based directory
- A Kerberos authentication authority
- A DNS-integrated identity database
- A distributed multi-master replicated system
- A Group Policy engine
- A hierarchical object store built on ESENT (JET Blue) database
- A forest of security boundaries and trust relationships

The core of AD is the NTDS.dit database and the Kerberos Key Distribution Center (KDC).

---

## 1. AD Setup — Domain Controller Installation

When you install AD DS, you promote a Windows Server into a Domain Controller (DC).

### Step 1 — Install AD DS Role

Server Manager → Add Roles and Features → Active Directory Domain Services

### Step 2 — Promote to Domain Controller

During promotion you configure:

- Forest name (e.g. leokadia.net)
- Domain name (same as forest or subdomain)
- Forest and domain functional level
- DS Restore Mode (DSRM) password

### Step 3 — AD Automatically Installs DNS

> ⚠ AD cannot function without DNS.

### Step 4 — AD Database Is Created

| File/Path                      | Purpose                     |
| ------------------------------ | --------------------------- |
| `C:\Windows\NTDS\NTDS.dit`     | The actual AD database      |
| `C:\Windows\NTDS\edb.log`      | Transaction logs            |
| `C:\Windows\SYSVOL`            | GPOs + scripts              |
| `C:\Windows\SYSVOL\domain`     | Domain-level SYSVOL content |
| `C:\Windows\SYSVOL\sysvol`     | Replication root            |

### DC Naming Options

| Option                   | Example / Notes                                      |
| ------------------------ | ---------------------------------------------------- |
| Root domain as AD domain | `leokadia.net` — requires split-brain DNS discipline |
| Subdomain (recommended)  | `ad.leokadia.net` / `corp.leokadia.net`              |
| Internal-only domain     | `leokadia.local` — common in older networks          |

---

## 2. What Runs on a Domain Controller

A Domain Controller is not just a server — it runs a full stack of services:

| Service / Component                    | What It Does                                                               |
| -------------------------------------- | -------------------------------------------------------------------------- |
| KDC (Kerberos Key Distribution Center) | Handles AS (Authentication Service) and TGS (Ticket Granting Service)     |
| LDAP directory service                 | Read/write protocol to all AD objects (ports 389/636)                     |
| NTDS (AD Database engine)              | Backed by Microsoft JET Blue / ESENT — stores all objects                 |
| DNS server                             | Stores AD-integrated DNS zones including SRV records                      |
| Netlogon                               | Authenticates machine accounts, locates DCs, secure channel               |
| LSASS                                  | Stores Kerberos tickets, password hashes, auth data; creates access tokens |
| DFSR (DFS Replication)                 | Replicates SYSVOL, policies, and scripts across DCs                       |
| DRSUAPI                                | RPC protocol for NTDS.dit replication between DCs                         |
| Global Catalog (GC)                    | Stores partial attributes of all objects across the entire forest         |

> ⚠ Compromising LSASS = full domain compromise. Never expose DC services to the Internet.

---

## 3. DNS in Active Directory

AD requires DNS for everything: DC location, Kerberos, LDAP, GC, and replication. AD creates two AD-integrated DNS zones on promotion:

### Zone 1: `_msdcs.<forest>`

- `_ldap._tcp.dc._msdcs.leokadia.net` — forest-wide DC locator
- `_kerberos._tcp.dc._msdcs.leokadia.net` — KDC discovery
- This zone is forest-wide critical

### Zone 2: `<domain>`

- DC A records and CNAMEs
- `_gc._tcp.leokadia.net` — Global Catalog locator
- LDAP and Kerberos SRV records
- All internal host records

**These zones are AD-integrated: stored inside NTDS.dit and replicated via AD replication — no zone transfers needed.**

### Split-Brain DNS (when AD uses root name)

- Public authoritative zone → Internet-facing records (www, mail, MX, SPF, DMARC)
- Internal AD-integrated zone → DC/Kerberos/LDAP SRV, internal hosts
- Internal zone must mirror public A/CNAMEs so internal users can reach public services

---

## 4. NTDS.dit — The Heart of Active Directory

NTDS.dit is the core database stored at `C:\Windows\NTDS\NTDS.dit`. It contains everything:

| What's Stored                        | Notes                                      |
| ------------------------------------- | ------------------------------------------ |
| User objects                          | All domain user accounts                   |
| Computer objects                      | All domain-joined machines                 |
| OUs, Groups                           | Organizational structure                   |
| Password hashes                       | NTLM (MD4) + Kerberos keys (AES/RC4-HMAC) |
| Service Principal Names (SPNs)        | Required for Kerberos service auth         |
| Trust relationships                   | Inter-domain/forest trusts                 |
| Group Policy Container (GPC) objects  | GPO configuration stored in LDAP          |
| DNS zones                             | If AD-integrated (application partitions)  |
| Schema objects                        | Defines all object types and attributes    |
| Configuration partition               | Forest topology (sites, services)          |
| Domain partition                      | Users, computers, groups for each domain   |

### Internal Database Structure

- Engine: Microsoft ESENT (Extensible Storage Engine) / JET Blue
- Page size: 8 KB
- Data structure: B+ trees
- Transaction logs: `edb.log`
- Checkpoint files: `edb.chk`

### Password Storage

- NTLM hashes stored as MD4
- Kerberos keys stored as AES-256 and RC4-HMAC (depends on domain functional level)

---

## 5. LDAP — Directory Protocol

AD uses LDAPv3 as its directory access protocol. Everything in AD is an object with a Distinguished Name (DN), a GUID, a SID, and attributes.

### Example DN

```
CN=Ian Kimori,OU=IT,DC=leokadia,DC=net
```

### LDAP Partitions (Naming Contexts)

| Partition               | Contains                                              |
| ----------------------- | ----------------------------------------------------- |
| Schema Partition        | Defines all object types and attributes — forest-wide |
| Configuration Partition | Forest topology (sites, services, replication)        |
| Domain Partition        | Actual users, groups, computers for that domain       |
| Application Partitions  | DNS zones (forest-wide or domain-wide)                |

### LDAP Ports

| Port | Protocol                     |
| ---- | ---------------------------- |
| 389  | Standard LDAP                |
| 636  | LDAPS (SSL)                  |
| 3268 | Global Catalog (forest-wide) |
| 3269 | Global Catalog over SSL      |

---

## 6. Kerberos — The Authentication Engine

### Keys & Accounts

- `krbtgt` account: root key for TGT encryption — one per domain
- Service keys: derived from account passwords tied to SPNs

### Authentication Flow

| Step | Exchange            | What Happens                                                               |
| ---- | ------------------- | -------------------------------------------------------------------------- |
| 1    | AS-REQ / AS-REP     | Client asks KDC for TGT; KDC returns TGT encrypted with krbtgt key        |
| 2    | TGS-REQ / TGS-REP   | Client asks KDC for a service ticket (TGS) for a target SPN               |
| 3    | Ticket Presentation | Client presents TGS to resource; resource validates with its key           |
| 4    | Token Creation      | LSASS reads PAC from ticket, builds access token with SIDs and privileges  |

### PAC (Privilege Attribute Certificate)

Embedded inside Kerberos tickets. Contains:

- User SID
- Group memberships
- SIDHistory
- Authorization info

### Service Ticket Targets (examples)

- `CIFS/fileserver01` — SMB file share
- `HTTP/intranet.leokadia.net` — Web app
- `MSSQLSvc/sqlserver01:1433` — SQL Server
- `LDAP/dc01.leokadia.net` — LDAP query

### Kerberos Hardening Controls

- PKINIT — smart card authentication
- Constrained / Resource-based delegation (S4U2Self, S4U2Proxy) — restrict delegation
- FAST/Armoring — modern ticket protection
- AES-only realm — disable RC4 keys
- Enforce Kerberos; disable NTLM where feasible

### Critical Attack Vectors — Know These

- **Kerberoasting** — request TGS for any SPN, crack offline if service account has weak password
- **Golden Ticket** — steal krbtgt key → forge any TGT → total domain takeover
- **Silver Ticket** — steal service account key → forge TGS for that service only

> ⚠ Stealing the krbtgt key = Total Domain Takeover.

---

## 7. NTLM — Legacy Fallback Authentication

NTLM is used when Kerberos is not available: no SPN configured, cross-forest without trust, or legacy systems.

- Prefer NTLMv2 only — disable NTLMv1 via policy
- Enforce SMB signing to prevent relay attacks
- Enforce LDAP signing and channel binding
- Disable or restrict NTLM via Group Policy where possible

---

## 8. OUs, Containers, and AD Structure

### Containers (cannot be GPO-targeted)

- `CN=Users` — default user objects
- `CN=Computers` — default computer objects
- `CN=Builtin` — built-in groups (Administrators, etc.)

### Organizational Units (GPO-targetable, delegable)

- `OU=IT`, `OU=Finance`, `OU=HR`, `OU=Servers`, `OU=Workstations`, etc.

**OUs are used for:**

- Administrative delegation
- Policy (GPO) assignment and scoping
- Organizing computers and users
- Tiering (Tier-0 vs Servers vs Workstations)

Best practice: move objects out of default CNs into managed OUs immediately after creation.

### Example Full DN

```
CN=John Doe,OU=IT,OU=Users,DC=leokadia,DC=net
```

---

## 9. Group Policy — The Control Plane

Group Policy has two physical parts stored in two different locations:

| Component                    | Storage Location                              |
| ---------------------------- | --------------------------------------------- |
| GPC (Group Policy Container) | NTDS.dit — LDAP object in AD                  |
| GPT (Group Policy Template)  | SYSVOL (`C:\Windows\SYSVOL\domain\Policies`)  |

### Processing Order — LSDOU

- Local Policy
- Site Policy
- Domain Policy
- OU Policy (parent OU → child OU)

Later policies overwrite earlier ones unless Enforced.

### Inheritance Controls

- **Block Inheritance** — prevents parent GPOs from flowing down
- **Enforce / No Override** — forces a GPO regardless of Block Inheritance
- **Security Filtering** — apply GPO only to specific groups
- **WMI Filters** — conditional targeting by hardware/OS properties
- **Loopback (Replace/Merge)** — applies user GPOs based on computer OU, used for RDS/VDI/kiosk

### Common Policy Areas

- Security baselines (CIS / Microsoft)
- Credential Guard and LSA protection
- SMB signing, Firewall rules
- Software restriction / AppLocker / WDAC
- Disable legacy protocols (SMBv1, LM/NTLM)
- Password policies (or Fine-Grained Password Policies via PSOs)
- Login scripts, drive mappings, printer deployment
- Browser and Office hardening

---

## 10. AD Replication

AD uses multi-master replication — any DC can accept writes and replicate changes to all others.

### Protocols

- DRSUAPI (MS-DRSR) — replicates NTDS.dit content via RPC over port 135 + dynamic high ports
- DFSR — replicates SYSVOL (GPOs and scripts)

### Replication Tracking

- USN (Update Sequence Number) — per-DC change counter
- High-watermark vectors — track which changes have been received
- Up-to-dateness vectors — prevent replication loops

### Site Topology (Sites and Services)

- Sites map to IP subnets — define physical network boundaries
- Intra-site replication: frequent, uncompressed, notification-driven
- Inter-site replication: scheduled, compressed, via site links

### Tombstone Lifetime

- Default ~180 days — controls deletion retention
- If a DC is offline past tombstone lifetime, lingering objects result
- Use `repadmin /removelingeringobjects` to clean up

---

## 11. Trusts & Forests

| Trust Type     | Description                                                                    |
| -------------- | ------------------------------------------------------------------------------ |
| Intra-forest   | Full transitive trust across the forest root and all child domains — automatic |
| External trust | One-way or two-way trust between domains in different forests                  |
| Forest trust   | Full transitive trust between two forest roots                                 |
| Realm trust    | Trust with MIT Kerberos realms (non-Windows)                                   |

- Use selective authentication for inter-forest trusts to limit exposure
- Trust direction determines who can authenticate to whom
- Transitive trusts flow through intermediaries; non-transitive trusts do not

---

## 12. FSMO Roles & Time

### Five FSMO Roles

| Role                  | Scope / Function                                                  |
| --------------------- | ----------------------------------------------------------------- |
| Schema Master         | Forest-wide — controls schema changes                             |
| Domain Naming Master  | Forest-wide — controls adding/removing domains                    |
| RID Master            | Domain — allocates RID pools to DCs for SID creation              |
| PDC Emulator          | Domain — time source, GPO edits, password pre-auth, legacy compat |
| Infrastructure Master | Domain — resolves cross-domain object references                  |

### Time Synchronization

- All domain members sync time from their DC
- DCs sync from the PDC Emulator
- PDC Emulator syncs from a reliable external NTP source

> ⚠ Kerberos has a tight 5-minute clock skew tolerance. Time drift = auth failures.

---

## 13. Security Model — SIDs, Groups, Tokens

### Identifiers

- **SID** (Security Identifier) — unique security identity for every object
- **GUID** — globally unique, immutable, never changes even if object is moved/renamed
- Well-known SIDs: `S-1-5-32-544` = Administrators, `S-1-1-0` = Everyone, etc.

### SPNs (Service Principal Names)

SPNs uniquely identify Kerberos service instances. Examples:

```
MSSQLSvc/sqlserver01.leokadia.net:1433
HTTP/intranet.leokadia.net
CIFS/fileserver01.leokadia.net
```

SPNs are the reason for Kerberoasting attacks, delegation, and service authentication.

### Access Token

Created by LSASS at logon. Contains:

- User SID
- Group SIDs
- Privileges
- Used for all ACL/DACL authorization checks

### Group Strategy — AGDLP / AGUDLP

- A = Accounts → G = Global Groups → DL = Domain Local Groups → P = Permissions
- Nest user accounts into global groups, global groups into domain local groups, assign permissions to domain local groups

---

## 14. Full Authentication Flow — End to End

| Step               | What Happens                                                              |
| ------------------ | ------------------------------------------------------------------------- |
| 1 — DNS Lookup     | Client queries DNS: `_ldap._tcp.dc._msdcs.leokadia.net` to find nearest DC |
| 2 — DC Locator     | Netlogon finds nearest DC in the client's site                            |
| 3 — AS Exchange    | Client sends AS-REQ; KDC returns TGT (encrypted with krbtgt key)         |
| 4 — TGS Exchange   | Client requests service ticket (TGS) for target SPN                      |
| 5 — Token Creation | LSASS reads PAC → builds access token with SIDs + privileges              |
| 6 — Authorization  | Token evaluated against resource DACLs                                    |
| Fallback           | NTLMv2 if Kerberos unavailable — avoid where possible                    |

---

## 15. LDAP Security & AD CS

### LDAP Security

- LDAPS (port 636) — requires a DC certificate with Computer EKU, trusted by clients
- LDAP signing — prevents packet tampering
- LDAP channel binding — prevents relay/MITM attacks

### AD CS (On-Prem Certificate Authority)

- Enables smart card / PKINIT authentication
- Auto-enrollment via GPO for machine and user certificates
- Issues service certificates (LDAPS, IIS, RDS, etc.)

> ⚠ ESC1–ESC8 misconfigurations in AD CS templates are common lateral movement and privilege escalation paths — audit all templates.

---

## 16. Backup, Restore & Operations

| Operation                | Method / Notes                                                                          |
| ------------------------ | --------------------------------------------------------------------------------------- |
| System State Backup      | Includes NTDS.dit, SYSVOL, registry, boot files                                         |
| Authoritative Restore    | `ntdsutil` — rolls back specific objects; marks them authoritative so they replicate out |
| AD Recycle Bin           | Soft-restore deleted objects; enable at forest level once                               |
| IFM (Install From Media) | `ntdsutil ... ifm create full` — offline DC promotion with reduced WAN traffic          |

```
ntdsutil "activate instance ntds" "ifm" create full C:\IFM
ntdsutil "activate instance ntds" "ifm" create full C:\Dump
```

---

## 17. Required Ports — Intra-AD Communication

| Port / Protocol   | Service                                        |
| ----------------- | ---------------------------------------------- |
| 88 TCP+UDP        | Kerberos                                       |
| 389 / 636         | LDAP / LDAPS                                   |
| 3268 / 3269       | Global Catalog / GC-SSL                        |
| 53 TCP+UDP        | DNS                                            |
| 445               | SMB (SYSVOL, file shares)                      |
| 135 + 49152–65535 | RPC Endpoint Mapper + dynamic high ports       |
| 5722              | DFSR (SYSVOL replication)                      |

> ⚠ Segment DCs in a protected Tier-0 network. Only permit required ports from Privileged Access Workstations (PAWs).

---

## 18. Administration & Diagnostics Commands

### Built-in CLI

**Identity & Membership**

```
whoami /fqdn
whoami /groups
```

**Users & Groups**

```
net user /domain
net user <username> /domain
net group /domain
```

**DC Discovery**

```
nltest /dsgetdc:<domain>
nltest /dclist:<domain>
nltest /sc_verify:<domain>
```

**Kerberos Tickets**

```
klist
```

**Replication Health**

```
repadmin /replsummary
repadmin /showrepl
dcdiag /v
```

**Query Objects (dsquery / dsget)**

```
dsquery user -name John*
dsquery computer -o rdn
dsget user "CN=John Doe,OU=IT,DC=leokadia,DC=net" -memberof
```

**GPO Troubleshooting**

```
gpresult /r /scope:computer
gpresult /h report.html
gpupdate /force
```

**Secure Channel & IFM**

```
powershell -c "Test-ComputerSecureChannel -Verbose"
ntdsutil "activate instance ntds" "ifm" create full C:\IFM
```

### PowerShell (ActiveDirectory Module / RSAT)

**Domain & Forest**

```powershell
Get-ADDomain
Get-ADForest
Get-ADDomainController -Discover -NextClosestSite
```

**Users & Groups**

```powershell
Get-ADUser -Filter * -Properties memberOf | Select Name,SamAccountName
Get-ADGroup -Filter * | Select Name,GroupScope
Get-ADGroupMember "Domain Admins"
```

**Computers & OUs**

```powershell
Get-ADComputer -Filter * -Properties IPv4Address,OperatingSystem
Get-ADOrganizationalUnit -Filter * | Select Name,DistinguishedName
```

**GPO**

```powershell
Get-GPO -All
Get-GPResultantSetOfPolicy -ReportType Html -Path .\RSoP.html
```

**Replication**

```powershell
Get-ADReplicationPartnerMetadata -Target * -Scope Forest
Get-ADReplicationFailure -Scope Site
```

**Fine-Grained Password Policies**

```powershell
Get-ADFineGrainedPasswordPolicy -Filter *
```

---

## 19. Monitoring & Audit — Key Event IDs

| Event ID    | Category     | What It Means                                   |
| ----------- | ------------ | ----------------------------------------------- |
| 4624        | Logon        | Successful logon — check logon type             |
| 4625        | Logon        | Failed logon attempt                            |
| 4768        | Kerberos     | TGT requested (AS-REQ)                          |
| 4769        | Kerberos     | Service ticket requested (TGS-REQ)              |
| 4776        | NTLM         | NTLM authentication attempt                     |
| 4720 / 4726 | Account Mgmt | User account created / deleted                  |
| 4728 / 4729 | Group        | Member added/removed from global security group |
| 4672        | Privilege    | Admin logon with special privileges             |
| 4732 / 4733 | Group        | Local group membership change on DC             |
| 4740        | Account      | Account locked out                              |
| 5136 / 5137 | Directory    | AD object modified / created                    |
| 1102        | Audit        | Security audit log cleared — high priority      |

Forward all events to a SIEM. Enable Advanced Audit Policy and Kerberos/NTLM auditing. Prioritize DC and Tier-0 monitoring.

---

## 20. Security Hardening Checklist

| Area             | Action                                                                                                                |
| ---------------- | --------------------------------------------------------------------------------------------------------------------- |
| Tiering          | Isolate DCs and Tier-0 admins. Use PAWs for admin access only.                                                       |
| GPO Hardening    | Enable LSA protection, Credential Guard, SMB signing, LDAP signing & channel binding, disable LM/NTLM where possible |
| Kerberos         | Prefer AES-256 only. Rotate krbtgt after incidents. No unconstrained delegation.                                      |
| Service Accounts | Use gMSA where possible. Randomized long passwords. Audit and clean SPNs.                                            |
| Protocols        | Disable SMBv1. Restrict legacy protocols. Enforce HTTPS/LDAPS. HSTS on web services.                                 |
| Patching         | DCs on first-class patch cycle — patch before workstations.                                                          |
| Backups          | Regular System State backups. Test authoritative restore. Enable AD Recycle Bin.                                      |
| DNS              | Keep AD SRV records internal. Implement split-brain properly. Do not expose DC DNS.                                   |
| Time             | Authoritative NTP on PDC Emulator. Tight clock skew for Kerberos.                                                    |
| Admin Scope      | Just-in-time (JIT) / Just-enough admin (JEA). Eliminate standing domain admin rights.                                |
| AD CS            | Audit all certificate templates for ESC1–ESC8 misconfigurations.                                                     |

---

## 21. Client Joining & App Integration

### Joining Clients

- Ensure client DNS points to AD DNS server (set via DHCP Option 006)
- Join via Settings, Control Panel, or PowerShell:

```powershell
Add-Computer -DomainName leokadia.net
```

### Application Integration

- Kerberos via SPNs (`HTTP/...`, `MSSQLSvc/...`, `CIFS/...`) — register with `setspn`
- LDAP/LDAPS for directory lookups and application authentication binds
- AD CS for certificate auto-enrollment via GPO

### Hybrid (On-Prem + Azure AD)

- Azure AD Connect — Password Hash Sync (PHS) or Pass-Through Authentication (PTA)
- Conditional Access policies in Azure AD
- Azure AD SSO while keeping on-prem AD authoritative for Kerberos/NTLM

---

## 22. Architecture — How It All Connects

The full picture, from client authentication to resource access:

```
Clients (Win/macOS/Linux)
    │
    ▼
DNS (AD-integrated) → _ldap._tcp / _kerberos._tcp SRV records
    │
    ▼
Domain Controller
    ├── KDC → issues TGT (AS) and Service Tickets (TGS)
    ├── LDAP (389/636/3268) → directory queries
    ├── LSASS → access token creation from PAC
    ├── NTDS.dit → stores everything
    ├── DNS → SRV record lookup for all AD services
    └── SYSVOL (via DFSR) → delivers GPOs to clients
    │
    ▼
Resources (CIFS, MSSQL, HTTP, LDAP) ← validated via Kerberos TGS
```

AD is one unified system: NTDS.dit stores all objects → LDAP provides access → Kerberos handles authentication → DNS enables discovery → Domain Controllers are the authority → OUs organize objects → GPOs control systems → SYSVOL replicates policies → LSASS manages tokens → Replication keeps all DCs in sync.
