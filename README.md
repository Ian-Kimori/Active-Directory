# Active-Directory

Perfect — here is the **deep, internal, security‑level explanation** of EVERYTHING in Active Directory, from **setup → DC architecture → DNS → LDAP → Kerberos → NTDS.dit → OUs → Containers → GPO → Authentication flow → Commands → Internal objects → Replication → SYSVOL**.

This is the **full big picture**, written for **security engineers and domain experts**.

***

# 🟣 **0. WHAT ACTIVE DIRECTORY *IS***

Active Directory Domain Services (AD DS) is:

### ✔ An LDAP‑based directory

### ✔ A Kerberos authentication authority

### ✔ A DNS-integrated identity database

### ✔ A distributed multi‑master replicated system

### ✔ A Group Policy engine

### ✔ A hierarchical object store built on ESENT (JET Blue) database

### ✔ A forest of security boundaries and trust relationships

And the core of AD is the **NTDS.dit database** and the **Kerberos Key Distribution Center (KDC)**.

***

# 🟦 **1. AD SETUP (Domain Controller Installation, Roles, and Files)**

When you install Active Directory Domain Services (AD DS):

## ✔ You promote a Windows Server into a **Domain Controller (DC)**

This involves:

### 🟩 Step 1 — Install AD DS role

### 🟩 Step 2 — Promote to Domain Controller

You choose:

*   Forest name (e.g., `leokadia.net`)
*   Domain name (same as above or subdomain)
*   Forest functional level
*   Domain functional level
*   DS Restore Mode password (Directory Services Recovery Mode)

### 🟩 Step 3 — AD automatically installs DNS

Because AD **cannot function** without DNS.

### 🟩 Step 4 — Creates internal AD database

The database is created here:

    C:\Windows\NTDS\NTDS.dit   ← the actual AD database
    C:\Windows\NTDS\edb.log    ← transaction logs
    C:\Windows\SYSVOL          ← GPOs + scripts

### 🟩 Step 5 — Creates important folders

    C:\Windows\SYSVOL\domain
    C:\Windows\SYSVOL\sysvol
    C:\Windows\NTDS

***

# 🟦 **2. WHAT IS A DOMAIN CONTROLLER (DEEP INTERNAL)**

A Domain Controller runs:

### ✔ **KDC (Kerberos Key Distribution Center)**

Handles:

*   AS (Authentication Service)
*   TGS (Ticket Granting Service)

### ✔ **LDAP directory service**

LDAP = the read/write protocol to AD objects.

### ✔ **NTDS (AD Database engine)**

Backed by Microsoft JET Blue / ESENT database engine.

### ✔ **DNS server**

Stores AD‑integrated DNS zones.

### ✔ **Netlogon service**

Authenticates machine accounts, locates DCs.

### ✔ **LSASS (Local Security Authority Subsystem Service)**

Stores:

*   Kerberos tickets
*   Password hashes
*   Authentication data
*   Token generation

Compromise LSASS → full domain compromise.

### ✔ **DFS Replication (DFSR)**

Replicates:

*   SYSVOL
*   Policies
*   Scripts

### ✔ **DRSUAPI**

RPC protocol for replication of NTDS.dit content.

### ✔ **Global Catalog**

Stores partial attributes of all objects across the forest.

***

# 🟦 **3. DC NAMING OPTIONS (ROOT, SUBDOMAIN, LOCAL)**

### 🔵 Option A: Root domain as AD domain

Example:

    leokadia.net

Your current setup.

### 🟢 Option B: Subdomain as AD domain

Example:

    ad.leokadia.net
    corp.leokadia.net

Best practice.

### 🟠 Option C: Internal-only domain

Example:

    leokadia.local

Common in older networks.

***

# 🟦 **4. AD DNS INTERNALS (SRV RECORDS, ZONES, RESOLUTION)**

Active Directory requires DNS for **DC location, Kerberos, LDAP, GC, replication**.

When a DC is promoted, AD creates two DNS zones:

### ✔ **Zone 1: \_msdcs.leokadia.net**

Contains:

    _ldap._tcp.dc._msdcs.leokadia.net
    _kerberos._tcp.dc._msdcs.leokadia.net

This is forest-wide.

### ✔ **Zone 2: leokadia.net**

Contains:

*   Domain controllers
*   A records
*   CNAMEs
*   LDAP service records
*   Kerberos service records

### ✔ These zones live in NTDS.dit (AD-integrated).

No zone transfer needed → replication happens through AD replication.

***

# 🟦 **5. AD DATABASE — NTDS.dit (THE HEART OF AD)**

`NTDS.dit` contains EVERYTHING:

### ✔ User objects

### ✔ Computer objects

### ✔ OUs

### ✔ Groups

### ✔ Password hashes (NTLM + Kerberos keys)

### ✔ Service Principal Names (SPNs)

### ✔ Trust relationships

### ✔ Group Policy Container objects

### ✔ DNS zones (if AD-integrated)

### ✔ Schema objects

### ✔ Configuration partition

### ✔ Domain partition

### Internal Structure (technical):

*   Backed by Microsoft **ESENT** (Extensible Storage Engine), also called **JET Blue**
*   Page size: 8 KB
*   Uses B+ trees
*   Uses transaction logs (`edb.log`)
*   Uses checkpoint files (`edb.chk`)

### Password storage inside NTDS.dit:

*   NTLM hashes stored as MD4
*   Kerberos keys stored as AES, RC4-HMAC (depends on domain functional level)

***

# 🟦 **6. LDAP — HOW EVERYTHING IS STORED & ACCESSED**

AD uses **LDAPv3** as its directory protocol.

Everything in AD is an **object**.

### Object structure:

    distinguishedName (DN)
    objectGUID
    objectSID
    attributes

Example user DN:

    CN=Ian Kimori,OU=IT,DC=leokadia,DC=net

### LDAP partitions:

1.  **Schema Partition**
    *   Defines all object types and attributes.

2.  **Configuration Partition**
    *   Forest-wide configuration (sites, services).

3.  **Domain Partition**
    *   Actual users, groups, computers.

4.  **Application partitions**
    *   DNS zones (forest-wide or domain-wide).

### LDAP ports:

*   **389** → Normal LDAP
*   **636** → LDAPS
*   **3268** → Global Catalog
*   **3269** → Global Catalog over SSL

***

# 🟦 **7. KERBEROS — THE INTERNAL AUTHENTICATION ENGINE**

When a user logs in:

### Step 1: AS-REQ → AS-REP

The KDC issues a **Ticket Granting Ticket (TGT)** encrypted with the **krbtgt** account key.

### Step 2: TGS-REQ → TGS-REP

The client asks for service tickets (TGS) for:

*   File servers
*   Exchange
*   Web apps
*   LDAP
*   CIFS
*   MSSQL
*   Etc.

### Step 3: Client presents service ticket to resource.

### Keys inside Kerberos tickets:

*   PAC (Privilege Attribute Certificate) contains:
    *   Groups
    *   SIDHistory
    *   User SID
    *   Authorization info

If attacker steals TGT → **Golden Ticket** attack.  
If attacker steals krbtgt key → **Total Domain Takeover**.

***

# 🟦 **8. OUs, CONTAINERS, AND AD STRUCTURE**

Active Directory organizes objects in:

### ✔ Containers (non-GPO targetable)

*   CN=Users
*   CN=Computers
*   CN=Builtin

### ✔ Organizational Units (GPO targetable)

*   OU=IT
*   OU=Finance
*   OU=HR

OUs are used for:

*   Delegation
*   Policy assignment
*   Organizing computers and users

### Example full DN:

    CN=John Doe,OU=IT,OU=Users,DC=leokadia,DC=net

***

# 🟦 **9. GROUP POLICY — HOW AD CONTROLS MACHINES**

Group Policy has two parts:

### ✔ GPC (Group Policy Container) — stored in NTDS.dit

### ✔ GPT (Group Policy Template) — stored in SYSVOL

GPOs flow:

    DC --> SYSVOL --> Clients

Examples of controls:

*   Password policy
*   Firewall rules
*   Software installation
*   Registry modifications
*   Login scripts
*   Security restrictions

***

# 🟦 **10. AD REPLICATION (DEEP)**

AD uses **multi‑master replication**.

Protocols:

*   **DRSUAPI**
*   **MS-DRSR**
*   RPC over 135
*   DFSR for SYSVOL

Replication is divided by:

### ✔ Sites

### ✔ Subnets

### ✔ Site Links

### ✔ Intra‑site and inter‑site replication

***

# 🟦 **11. AUTHENTICATION FLOW (FULL)**

### When you log into a domain:

#### 1️⃣ DNS query:

    _ldap._tcp.dc._msdcs.leokadia.net

#### 2️⃣ DC Locator finds nearest Domain Controller

#### 3️⃣ Kerberos AS-REQ sent to DC

DC returns TGT.

#### 4️⃣ Client requests Ticket Granting Service (TGS) ticket

For whatever service it wants (CIFS, HTTP, MSSQL)

#### 5️⃣ Token created in LSASS

Contains:

*   SID
*   Group memberships
*   Primary group
*   Security privileges

#### 6️⃣ You access resources.

***

# 🟦 **12. IMPORTANT COMMANDS (ADMIN + SECURITY)**

### ✔ List domain users

    net user /domain

### ✔ List domain groups

    net group /domain

### ✔ Find all computers

    dsquery computer

### ✔ Find a user

    dsquery user -name john*

### ✔ Describe user

    net user username /domain

### ✔ See your domain

    whoami /fqdn

### ✔ Find Domain Controller

    nltest /dsgetdc:leokadia.net

### ✔ Replication status

    repadmin /replsummary

### ✔ Show all Kerberos tickets

    klist

### ✔ Export NTDS.dit (for forensics)

    ntdsutil "activate instance ntds" "ifm" create full C:\Dump

***

# 🟦 **13. INTERNAL OBJECTS (SIDs, GUIDs, SPNs)**

### ✔ Security Identifiers (SIDs)

Unique to every object.

### ✔ GUIDs

Immutable, never change.

### ✔ SPNs (Service Principal Names)

Unique identifiers for services using Kerberos.

Example:

    MSSQLSvc/sqlserver01.leokadia.net:1433
    HTTP/intranet.leokadia.net

SPNs are the reason for:

*   Kerberoasting attacks
*   Delegation
*   Service authentication

***

# 🟦 **14. PUTTING IT ALL TOGETHER (THE WHOLE SYSTEM)**

Active Directory is a system where:

*   **NTDS.dit stores everything**
*   **LDAP is the access protocol**
*   **Kerberos is the authentication mechanism**
*   **DNS is the location service**
*   **Domain Controllers are the authority**
*   **OUs organize objects**
*   **GPO controls systems**
*   **SYSVOL replicates policies**
*   **LSASS manages tickets and tokens**
*   **Replication keeps all DCs in sync**
*   **Clients use DNS to find DCs**
*   **DCs use Kerberos to issue tickets**

Everything is connected.

***

# 🟩 **If you want, I can now generate for you:**

### ✔ A full diagram (architecture, flows, internal structures)

### ✔ A PDF of this whole explanation

### ✔ A cheat sheet for all AD internals

### ✔ A pentester’s cheat sheet (commands + attacks + enumeration)

### ✔ A network diagram including AD, DNS, Kerberos, LDAP

Tell me what format you want next.


Awesome—here’s a **deep, clean, enterprise‑grade** guide to Active Directory (AD), from initial setup through LDAP/Kerberos internals, directory structure (OUs/containers), the DC stack (NTDS.dit, DNS, LSASS, SYSVOL), replication, Group Policy processing, trusts, auth flows, commands, and security hardening. It’s written for **professional implementation and security operations**.

***

## 1) What AD DS Is (Crisp Definition)

**Active Directory Domain Services (AD DS)** is Microsoft’s on‑prem identity platform that provides:

*   **Authentication**: Kerberos (primary), NTLM (legacy fallback)
*   **Authorization**: Group membership → access tokens → ACL evaluation
*   **Directory**: LDAP‑addressable object store (users, computers, groups, policies, services)
*   **Policy**: Group Policy (GPO) delivery and enforcement
*   **Naming/Discovery**: AD‑integrated DNS (SRV records for DC/KDC/GC location)
*   **Replication**: Multi‑master, site‑aware, transactional replication across DCs

At the core: **NTDS.dit** (ESENT database), **KDC**, **LDAP**, **DNS**, **LSASS**, **SYSVOL**.

***

## 2) Deployment: From Zero to First Domain Controller

### 2.1 Prereqs

*   Windows Server 2019/2022 (hardened baseline)
*   Static IP, correct time sync (NTP), hostname (e.g., `DC01`)
*   Pick domain naming model:
    *   **Root**: `leokadia.net` (requires split‑brain DNS discipline)
    *   **Subdomain (recommended)**: `ad.leokadia.net` / `corp.leokadia.net`
    *   **Internal**: `leokadia.local` (simple, less cloud‑friendly)

### 2.2 Install AD DS + Promote

*   **Server Manager** → *Add Roles and Features* → **Active Directory Domain Services**
*   Promote:
    *   New forest/domain or join existing
    *   Set **DSRM** password
    *   Enable **DNS Server** during promotion
*   Reboot; verify services:
    *   **AD DS**, **KDC**, **DNS**, **Netlogon**, **DFSR**, **LSASS** running

### 2.3 Files & Locations

*   `C:\Windows\NTDS\NTDS.dit` → AD database
*   `C:\Windows\NTDS\edb.log` → transaction log
*   `C:\Windows\SYSVOL\domain\Policies` → GPO templates (GPT)
*   `_msdcs.<forest-root>` and `<domain>` → AD‑integrated DNS zones

***

## 3) Core DC Stack (What Actually Runs on a DC)

*   **NTDS (AD DS / ESENT)** → Object store (users, groups, computers, OUs, trusts, DNS if integrated)
*   **KDC** → Kerberos Authentication Service (AS) + Ticket‑Granting Service (TGS)
*   **LDAP** → Directory protocol (389/636), GC (3268/3269)
*   **DNS (AD‑integrated)** → SRV records for DC/KDC/GC/DC‑locator
*   **LSASS** → Security authority; holds Kerberos tickets, cached creds, creates access tokens
*   **Netlogon** → Secure channel, DC locator registrations, machine account auth
*   **DFSR** → Replicates **SYSVOL** (GPOs/scripts)
*   **Global Catalog (GC)** → Partial attribute set for forest‑wide searches

**Never** expose DC services (LDAP/DNS/Kerberos/SMB/RPC) directly to the Internet.

***

## 4) DNS in AD (Service Discovery Backbone)

AD requires DNS for **everything**: clients locate DCs via SRV records.

### 4.1 AD SRV Records (examples)

*   `_ldap._tcp.dc._msdcs.leokadia.net` → DC locator (forest-wide)
*   `_kerberos._tcp.leokadia.net` → KDC discovery
*   `_gc._tcp.leokadia.net` → Global Catalog
*   Host A records for DCs: `DC01.leokadia.net`

### 4.2 Zones

*   **`_msdcs.<forest-root>`**: forest‑critical locator info
*   **`<domain>`**: domain service/host records

These zones are **AD‑integrated**: stored in NTDS.dit and replicated by AD, not via zone transfers.

### 4.3 Split‑Brain DNS (if AD uses the root name)

*   **Public authoritative zone** → Internet‑facing records (www, mail, MX, SPF, DMARC)
*   **Internal AD‑integrated zone** → DC/Kerberos/LDAP SRV, internal hosts
*   Internals must **mirror** needed public A/CNAMEs so users inside can still reach public services.

***

## 5) LDAP (Directory Model)

*   **Protocol**: LDAPv3 (389), LDAPS (636)
*   **Partitions (Naming Contexts)**:
    *   **Schema**: object/attribute definitions
    *   **Configuration**: forest topology (sites, services)
    *   **Domain**: users, groups, computers, OUs for that domain
    *   **Application**: e.g., DNS application partitions

**Objects** have:

*   **DN**: `CN=Jane Doe,OU=IT,DC=leokadia,DC=net`
*   **GUID** (immutable), **SID** (security identity)
*   Attributes (e.g., `sAMAccountName`, `userPrincipalName`, `memberOf`, `servicePrincipalName`)

**GC (3268/3269)** → forest‑wide searchable subset of attributes.

***

## 6) Kerberos (Primary Authentication)

### 6.1 Keys & Accounts

*   **krbtgt** account: root key for TGT encryption (per domain)
*   Service keys derived from account passwords (for SPNs)

### 6.2 Flow

1.  **AS‑REQ/AS‑REP**: Client asks KDC for **TGT**; KDC returns TGT (encrypted with `krbtgt` key)
2.  **TGS‑REQ/TGS‑REP**: Client asks KDC for **service ticket** (TGS) for a target SPN (e.g., `CIFS/fileserver1`)
3.  Client presents TGS to the service; service validates ticket using its key

**PAC** inside tickets conveys group memberships/SID—consumed by LSASS to produce the access token.

### 6.3 Variants & Controls

*   **PKINIT** (smart cards)
*   **Constrained/Resource‑based delegation** (S4U2Self, S4U2Proxy)
*   **FAST/Armoring** (modern hardening)
*   **AES‑only** realm (disable RC4)
*   **Kerberos enforcement** on services (disable NTLM where feasible)

**Operational risks** (know them): Kerberoasting (weak service account passwords), Golden/Silver tickets (krbtgt/service key theft).

***

## 7) NTLM (Legacy Fallback)

*   Used when Kerberos isn’t available (e.g., no SPN, cross‑forest w/o trust, legacy systems)
*   Prefer **NTLMv2** only, enforce SMB signing, **LDAP signing/channel binding**, disable NTLM where possible or restrict via policies.

***

## 8) Directory Structure: OUs vs Containers

*   **Containers** (non‑GPO targetable): `CN=Users`, `CN=Computers`, `CN=Builtin`
*   **OUs** (GPO targetable, delegable): `OU=Endpoints`, `OU=Servers`, `OU=Finance`, `OU=IT`
    *   Use OUs for **administrative delegation**, **scope boundaries**, **policy application**, **tiering** (Workstations vs Servers vs Tier‑0 admin assets)

**Naming best practices**: Predictable OU tree; keep default CNs empty and relocate objects to managed OUs.

***

## 9) Group Policy (GPO): Control Plane

*   **GPC** (container) in **NTDS.dit** + **GPT** (template files) in **SYSVOL**
*   Processing order: **L**ocal → **S**ite → **D**omain → **OU** (**LSDOU**)
*   Inheritance controls: **Block Inheritance**, **Enforce/No Override**, **Security Filtering**, **WMI filters**, **Loopback** (Replace/Merge) for RDS/kiosk/VDI

Common policy areas:

*   Security baselines (CIS/Microsoft)
*   Credential Guard, LSA protection, SMB signing, Firewall
*   Software Restriction / AppLocker / WDAC
*   Disable legacy protocols (SMBv1, LM/NTLM as appropriate)
*   Browser/Office hardening
*   Scripts/Drives/Printers deployment

***

## 10) Replication & Sites

*   **Multi‑master** replication using **DRSUAPI (MS‑DRSR)** over RPC
*   **USN** (Update Sequence Number) tracking per DC
*   **High‑watermark vectors**, **up‑to‑dateness vectors** prevent loops
*   **Site topology** (Active Directory Sites and Services):
    *   Map **IP subnets** to **Sites**
    *   Intra‑site: frequent, uncompressed
    *   Inter‑site: scheduled, compressed via **site links**

**SYSVOL** uses **DFSR** (modern) to replicate GPOs/scripts (retire FRS if still present).

**Tombstone lifetime** (default \~180 days) controls object deletion retention; **lingering objects** occur if a DC is offline past tombstone—use `repadmin` cleanup.

***

## 11) Trusts & Forests

*   **Intra‑forest**: full transitive trust across the forest root and child domains
*   **Inter‑forest trusts**: external, forest, realm trusts (MIT Kerberos realms)
*   **Direction**: one‑way or two‑way; **Transitivity**: transitive or non‑transitive
*   Use selective authentication for security boundaries.

***

## 12) Time, PDC Emulator & FSMO Roles

**FSMO (5 roles):**

1.  **Schema Master** (forest)
2.  **Domain Naming Master** (forest)
3.  **RID Master** (domain)
4.  **PDC Emulator** (domain) — time source, password pre‑auth, GPO edits, legacy compat
5.  **Infrastructure Master** (domain)

**Time**: All domain members must sync (Kerberos tolerance is tight). Default hierarchy: DCs sync from **PDC Emulator** → PDC syncs from reliable NTP.

***

## 13) Security Model (SIDs, Groups, Tokens)

*   **SID**: unique security identity per object
*   **Well‑known SIDs**: `S-1-5-32-544` (Administrators), etc.
*   **Groups**: AGDLP/AGUDLP strategies for nesting/pruning
*   **Access token** created at logon: contains user SID, group SIDs, privileges → used for ACL checks.

***

## 14) Authentication: End‑to‑End Flow

1.  **DC Locator**: Client queries DNS for `_ldap._tcp.dc._msdcs.<forest>`
2.  **AS Exchange**: TGT issued by KDC (krbtgt key)
3.  **TGS Exchange**: Service ticket for target SPN
4.  **Service validation**: Server validates ticket; LSASS creates access token
5.  **Authorization**: Token used against resource DACLs

Fallback: **NTLMv2** if Kerberos not possible (avoid where feasible).

***

## 15) Certificates, LDAP Security, and AD CS

*   **LDAPS (636)** requires a DC certificate (computer EKU) trusted by clients (enterprise CA or public CA)
*   **LDAP signing / channel binding**: mitigate relays/MITM
*   **AD CS** (on‑prem CA) enables:
    *   Smart‑card/PKINIT
    *   Auto‑enrollment via GPO
    *   Service certificates (LDAPS, IIS, RDS)
    *   Be mindful of **ESC‑x** misconfigurations in AD CS templates (common lateral movement risks)

***

## 16) Operations: Backup/Restore, Recycle Bin, IFM

*   **System State Backup** includes AD DS (NTDS.dit), SYSVOL, registry, boot files
*   **Authoritative Restore** (ntdsutil) for object rollback
*   **AD Recycle Bin**: soft‑restore deleted objects (enable at forest level)
*   **IFM (Install From Media)**: `ntdsutil ... ifm create full` for branch DC promotion with reduced WAN usage

***

## 17) Ports to Permit (Typical Intra‑AD)

*   **Kerberos**: 88/TCP+UDP
*   **LDAP/LDAPS**: 389/636
*   **GC/GC‑SSL**: 3268/3269
*   **DNS**: 53/TCP+UDP
*   **SMB**: 445
*   **RPC Endpoint Mapper**: 135, plus dynamic RPC high ports (49152–65535 by default)
*   **DFSR**: 5722

Segment DCs in a protected **Tier‑0** network; allow only required ports from managed admin workstations (PAWs).

***

## 18) Administration & Diagnostics (Commands)

### 18.1 Built‑in CLI

```bat
:: Identity & membership
whoami /fqdn
whoami /groups

:: Users/Groups
net user /domain
net user <username> /domain
net group /domain

:: DC discovery
nltest /dsgetdc:<domain>
nltest /dclist:<domain>

:: Kerberos tickets
klist

:: Replication health
repadmin /replsummary
repadmin /showrepl
dcdiag /v

:: Query objects (dsquery/dsget)
dsquery user -name John*
dsquery computer -o rdn
dsget user "CN=John Doe,OU=IT,DC=leokadia,DC=net" -memberof

:: GPO troubleshooting
gpresult /r /scope:computer
gpresult /h report.html
gpupdate /force

:: Secure channel
nltest /sc_verify:<domain>
powershell -c "Test-ComputerSecureChannel -Verbose"

:: IFM / Database ops
ntdsutil "activate instance ntds" "ifm" create full C:\IFM
```

### 18.2 PowerShell (RSAT / ActiveDirectory module)

```powershell
# Domain/forest
Get-ADDomain
Get-ADForest
Get-ADDomainController -Discover -NextClosestSite

# Users/Groups
Get-ADUser -Filter * -Properties memberOf | Select Name,SamAccountName
Get-ADGroup -Filter * | Select Name,GroupScope
Get-ADGroupMember "Domain Admins"

# Computers
Get-ADComputer -Filter * -Properties IPv4Address,OperatingSystem

# OUs
Get-ADOrganizationalUnit -Filter * | Select Name,DistinguishedName

# GPO
Get-GPO -All
Get-GPResultantSetOfPolicy -ReportType Html -Path .\RSoP.html

# Replication
Get-ADReplicationPartnerMetadata -Target * -Scope Forest
Get-ADReplicationFailure -Scope Site

# Fine-Grained Password Policies (PSOs)
Get-ADFineGrainedPasswordPolicy -Filter *
```

***

## 19) Monitoring & Audit (Security Operations)

**Key Event IDs (Windows Security Log):**

*   **4624**: Successful logon (type important)
*   **4625**: Failed logon
*   **4768/4769**: Kerberos TGT/TGS issuance
*   **4776**: NTLM authentication
*   **4720/4726**: User account created/deleted
*   **4728/4729**: Member added/removed from global security group
*   **4672**: Admin logon with special privileges
*   **4732/4733**: Local group membership change (DCs)
*   **4740**: Account locked out
*   **5136/5137**: Directory object modified/created
*   **1102**: Audit log cleared

Forward to SIEM; enable **Advanced Auditing** and **Kerberos/NTLM** auditing; monitor **DC** and **Tier‑0** with priority.

***

## 20) Security Hardening Checklist (High‑Value)

*   **Tiering**: Isolate DCs/Tier‑0 admins; use **PAWs** for admin
*   **GPO Hardening**: LSA protection, Credential Guard, SMB signing, disable LM/NTLM where possible, LDAP signing & channel binding, PowerShell Constrained Language Mode as appropriate
*   **Kerberos**: Prefer AES; rotate **krbtgt** after incidents; constrain delegation (no unconstrained)
*   **Accounts**: Randomized long passwords for service accounts; use **gMSA** where possible; clean SPNs
*   **Protocols**: Disable SMBv1; restrict legacy protocols; enforce HTTPS/LDAPS; HSTS on web
*   **Patching**: DCs first-class patch cycle
*   **Backups**: Regular **System State**; test authoritative restore; enable **AD Recycle Bin**
*   **DNS**: Keep AD SRV internal; do **not** expose DC DNS to Internet; implement split‑brain properly
*   **Time**: Authoritative NTP on PDC Emulator; tight skew for Kerberos
*   **Admin Scope**: Just‑in‑time, just‑enough admin (JIT/JEA); eliminate standing domain admin rights

***

## 21) Joining Clients & App Integration

*   **Client join**: Ensure client DNS points to **AD DNS** (DHCP Option 006). Join via Settings/Control Panel or `Add-Computer -DomainName leokadia.net`.
*   **App integration**:
    *   **Kerberos** via SPNs (HTTP/…, MSSQLSvc/…, CIFS/…)
    *   **LDAP/LDAPS** for directory lookups/binds
    *   **AD CS** for cert auto‑enrollment
*   **Hybrid**: Azure AD Connect (password hash sync / PTA), conditional access, AAD SSO—keep on‑prem AD authoritative for Kerberos/NTLM.

***

## 22) Clean Visual: How It All Fits

    Clients (Win/macOS/Linux) ──> DNS (AD-integrated) ──> _ldap/_kerberos SRV ──> Domain Controller
          |                          |                                  |
          |                          └──> Finds nearest DC via Sites    └──> KDC issues TGT/TGS
          |                                                                LDAP queries (389/636/3268)
          └──> Kerberos TGS for services (CIFS/MSSQL/HTTP with SPN)       LSASS creates access tokens
                                                                           GPO via SYSVOL (DFSR)

***

### Want me to package this as:

*   A **print‑ready PDF** (with diagrams),
*   A **Tier‑0 hardening checklist** you can run through in audits, or
*   A **Kerberos & LDAP troubleshooting flowchart**?

Tell me your preference and I’ll generate it.
