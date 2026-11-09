<p align="center">
  <img src="./assets/brandons-logo.png" alt="Brandon Stevenson | IT Professional" width="185">
</p>

# Palo Alto NGFW | Security Policy & Network Implementation

**Technical Implementation Record**  
Physical firewall restoration · PAN-OS CLI administration · Layer 3 segmentation · Security policy · Source NAT

**Platform:** Palo Alto Networks PA-820  
**Software:** PAN-OS 11.1.4-h7  
**Access:** Serial console using Tera Term  
**Completed:** November 9, 2026

---

## Overview

This record documents the restoration and configuration of a physical Palo Alto Networks next-generation firewall. Work was performed directly through the appliance's maintenance environment and PAN-OS command-line interface.

Following factory restoration and baseline inspection, I configured three Layer 3 interfaces, security zones, a virtual router, address and service objects, application-aware access policies, and outbound source NAT.

The candidate configuration was reviewed, validated, and committed. **PAN-OS reported `FIN / OK` for both the full validation and manual commit.** Final CLI captures confirm the presence of the security and NAT policies in the running configuration.

The appliance remained physically isolated during configuration. The recorded interface and routing outputs are included as observed; **the successful commit does not constitute a live connectivity or packet-forwarding test.**

### Contents

1. [Architecture](#architecture)
2. [Appliance restoration](#restoration)
3. [Console preparation and baseline inspection](#baseline)
4. [Layer 3 interfaces, security zones and routing](#network)
5. [Address and service objects](#objects)
6. [Security policy implementation](#policies)
7. [Source NAT and candidate review](#nat)
8. [Validation and commit](#commit)
9. [Post-change verification](#verification)
10. [Implementation results](#results)
11. [Repository structure](#repository)

<br>

<a id="architecture"></a>
## 01 — Architecture

![Configured NGFW network topology, security zones and policy flows](./assets/topology.png)

*Figure 1. Logical network topology showing interface assignments, zone boundaries, virtual routing, interzone security policy, and outbound source NAT. The flow arrows represent configured policy intent, not captured data-plane traffic.*

<br>

| Interface | Security zone | Configured IP | Function |
|---|---|---|---|
| `ethernet1/3` | `LAB-TRUST` | `10.10.10.1/24` | Client network gateway |
| `ethernet1/4` | `LAB-SERVERS` | `10.20.20.1/24` | Server network gateway |
| `ethernet1/5` | `LAB-UNTRUST` | `203.0.113.2/30` | External-facing interface and source-NAT address |

All three interfaces belong to virtual router `LAB-VR`. A default route was configured through `ethernet1/5` toward next hop `203.0.113.1`.

The private client and server networks are configuration addresses. The external-facing addresses use reserved documentation space (`203.0.113.0/24` and `198.51.100.0/24`); they are not evidence of an active ISP circuit. The `LAB-*` names are the identifiers entered in PAN-OS and are retained exactly as configured.

[Back to contents](#contents)

---

<a id="restoration"></a>
## 02 — Appliance Restoration

The implementation began at the hardware console. Startup and storage initialization were reviewed before entering the maintenance recovery environment.

### Boot and maintenance environment

**Figure 2 — Bootloader and hardware initialization.** The console records device startup, memory initialization and storage detection. Unique hardware identifiers have been redacted.

![Bootloader and hardware initialization](./evidence/s1.png)

<br>

**Figure 3 — Maintenance Recovery Tool.** Recovery access was established through the appliance's local maintenance interface.

![Maintenance recovery interface](./evidence/s2.png)

<br>

### Factory restoration and storage scrub

**Figure 4 — Reset and scrub selection.** The maintenance interface shows the selected restoration and storage-scrub workflow.

![Factory reset and scrub selections](./evidence/s3.png)

<br>

**Figure 5 — Scrub execution.** Console output records the storage-scrub operation in progress.

![Storage scrub in progress](./evidence/s4.png)

<br>

**Figure 6 — Restoration result.** The maintenance screen reports `Scrub Status: Success` and `Factory Reset Status: Success`.

![Scrub and factory reset success](./evidence/s5.png)

The reset established a clean configuration starting point for the subsequent firewall changes. The success messages are recorded as device-reported results; no independent erasure certification is asserted.

[Back to contents](#contents)

---

<a id="baseline"></a>
## 03 — Console Preparation and Baseline Inspection

After restart, I accessed the PAN-OS operational CLI through the local serial console. An interactive administrator password change was completed without exposing the credential in the repository.

### Console output and system logs

```text
set cli terminal height 500
set cli terminal width 500
clear log system
show log system
```

**Command notes**

- `set cli terminal height 500` — Increases the CLI output height to reduce pagination during long configuration captures.
- `set cli terminal width 500` — Increases logical output width to reduce command wrapping; it does not resize the Tera Term window.
- `clear log system` — Clears the local system log after confirmation. This destructive action belongs to the recorded reset workflow and is not a routine production change.
- `show log system` — Queries system-log records after the clear operation; the captured result shows no entries at that point.

**Figure 7 — Administrative console preparation.** Login, CLI display settings, and the system-log query are recorded together.

![Console settings and system-log inspection](./evidence/s6.png)

<br>

### Existing configuration and storage inspection

```text
show vlan all
show config saved running-config.xml
show config saved candidate-config
show config running
show system disk-space
```

**Command notes**

- `show vlan all` — Queries VLAN configuration. The returned message, `Server error : vlan configuration not found`, is documented as observed.
- `show config saved running-config.xml` — Reads the saved running-configuration XML for baseline review.
- `show config saved candidate-config` — Reads saved candidate configuration data for comparison.
- `show config running` — Displays the committed running configuration, including existing system and shared settings.
- `show system disk-space` — Reports filesystem utilization and available local storage.

**Figure 8 — VLAN query and saved running configuration.**

![VLAN query and saved running configuration](./evidence/s7.png)

<br>

**Figure 9 — Saved candidate configuration inspection.**

![Saved candidate configuration](./evidence/s8.png)

<br>

**Figure 10 — Saved running-configuration inspection.**

![Saved running configuration XML](./evidence/s9.png)

<br>

**Figure 11 — Active running-configuration inspection.** The displayed administrative values are masked where necessary.

![Running configuration](./evidence/s10.png)

<br>

**Figure 12 — Device metadata and disk usage.** Storage availability was checked before subsequent configuration work.

![System metadata and storage utilization](./evidence/s11.png)

The baseline review established the state of the appliance after restoration and before the new interface, zone, routing and policy configuration.

[Back to contents](#contents)

---

<a id="network"></a>
## 04 — Layer 3 Interfaces, Security Zones and Routing

### Interface addressing

```text
configure
set network interface ethernet ethernet1/3 layer3 ip 10.10.10.1/24
set network interface ethernet ethernet1/4 layer3 ip 10.20.20.1/24
set network interface ethernet ethernet1/5 layer3 ip 203.0.113.2/30
```

`configure` enters candidate-configuration mode, where subsequent `set` operations are staged until committed.

Each `set network interface ethernet ... layer3 ip` command assigns a Layer 3 address to a specific Ethernet interface: `ethernet1/3` for clients, `ethernet1/4` for servers, and `ethernet1/5` for the external-facing segment. Addressing alone does not establish routing or permit interzone traffic.

### Security-zone membership

```text
set zone LAB-TRUST network layer3 ethernet1/3
set zone LAB-SERVERS network layer3 ethernet1/4
set zone LAB-UNTRUST network layer3 ethernet1/5
```

These commands associate the three interfaces with their respective Layer 3 security zones. Zones supply the source and destination contexts used by the security and NAT rulebases.

**Figure 13 — Interface configuration and initial zone / virtual-router assignments.**

![Layer 3 interfaces, zones and virtual router configuration](./evidence/s12.png)

<br>

### Virtual-router membership and static route

```text
set network virtual-router LAB-VR interface ethernet1/3
set network virtual-router LAB-VR interface ethernet1/4
set network virtual-router LAB-VR interface ethernet1/5
set network virtual-router LAB-VR routing-table ip static-route LAB-DEFAULT destination 0.0.0.0/0 interface ethernet1/5 nexthop ip-address 203.0.113.1
```

The three interface-membership commands place the client, server and external-facing interfaces into the same virtual router, `LAB-VR`.

The final command creates the named static route `LAB-DEFAULT`, identifying `ethernet1/5` as the egress interface and `203.0.113.1` as the next hop for destinations matching `0.0.0.0/0`. It defines a route in configuration; operational installation is checked separately after commit.

The CLI capture containing the completed route entry continues directly into the address-object work shown in the following section.

[Back to contents](#contents)

---

<a id="objects"></a>
## 05 — Address and Service Objects

```text
set address LAB-CLIENT-NET ip-netmask 10.10.10.0/24
set address LAB-SERVER-NET ip-netmask 10.20.20.0/24
set address LAB-ADMIN ip-netmask 10.10.10.10/32
set address LAB-SERVER ip-netmask 10.20.20.10/32
set service LAB-HTTPS protocol tcp port 443
```

The first two commands define reusable network objects for the client and server subnets. `LAB-ADMIN` and `LAB-SERVER` create individual `/32` host objects, allowing the SSH exception to match one administrator and one server rather than an entire subnet.

`set service LAB-HTTPS protocol tcp port 443` creates a TCP/443 service object. The recorded outbound SSL policy uses `application-default`, so this service object was created but is not shown as the selected service in that rule.

**Figure 14 — Completed default route and address/service object creation.**

![Static route completion and address and service object commands](./evidence/s13.png)

[Back to contents](#contents)

---

<a id="policies"></a>
## 06 — Security Policy Implementation

Three application-aware rules were defined in the security rulebase. The narrowly scoped administrator SSH exception is positioned ahead of the broader server-access deny.

### Administrator SSH — allow

```text
set rulebase security rules LAB-Allow-Admin-SSH from LAB-TRUST to LAB-SERVERS source LAB-ADMIN destination LAB-SERVER application ssh service application-default action allow
```

This rule allows the `ssh` application from the single administrator address (`10.10.10.10/32`) to the designated server (`10.20.20.10/32`) across the trusted and server zones. `application-default` restricts the service to the application's default port behavior.

### Other client-to-server access — deny

```text
set rulebase security rules LAB-Deny-Server-Access from LAB-TRUST to LAB-SERVERS source LAB-CLIENT-NET destination LAB-SERVER-NET application any service any action deny
```

This rule denies traffic that matches the client-to-server zone and subnet conditions, regardless of application or service. Rule order is significant: the specific administrator SSH allowance must be evaluated before the broader deny for the intended exception to apply.

### Outbound SSL — allow and log

```text
set rulebase security rules LAB-Allow-HTTPS from LAB-TRUST to LAB-UNTRUST source LAB-CLIENT-NET destination any application ssl service application-default action allow
set rulebase security rules LAB-Allow-HTTPS log-end yes
```

The first command allows recognized `ssl` traffic from the client subnet to the external-facing zone when the application and default service criteria match.

The second enables session-end logging for this rule. It sets the logging behavior; it does not prove that a session was generated or logged.

**Figure 15 — Security-policy entry.** This capture shows the final portion of rule creation and the SSL logging setting. The complete candidate rule definitions are inspected later with `show rulebase security`.

![Security policy and session-end logging commands](./evidence/s14.png)

[Back to contents](#contents)

---

<a id="nat"></a>
## 07 — Source NAT and Candidate Review

### Outbound source translation

```text
set rulebase nat rules LAB-OUTBOUND-SNAT from LAB-TRUST to LAB-UNTRUST source LAB-CLIENT-NET destination any service any to-interface ethernet1/5 source-translation dynamic-ip-and-port interface-address interface ethernet1/5
```

`LAB-OUTBOUND-SNAT` matches traffic originating in `LAB-TRUST`, directed toward `LAB-UNTRUST`, sourced from `LAB-CLIENT-NET`, and egressing through `ethernet1/5`.

`source-translation dynamic-ip-and-port` selects dynamic source-address and port translation. `interface-address interface ethernet1/5` selects the configured egress interface address as the translated source. This is source NAT, not destination NAT or port forwarding.

### Candidate-configuration inspection

```text
show network
show zone
show address
show rulebase security
show rulebase nat
```

- `show network` — Reviews configured interfaces, addressing and virtual-router settings in candidate configuration.
- `show zone` — Confirms zone membership and interface bindings.
- `show address` — Lists the client, server and host address objects.
- `show rulebase security` — Shows the candidate security rules and matching criteria.
- `show rulebase nat` — Shows the NAT rule and its source-translation definition.

**Figure 16 — Network configuration review.**

![Candidate network configuration](./evidence/s15.png)

<br>

**Figure 17 — Security-zone inspection.** The configured `LAB-*` zones appear alongside pre-existing lowercase `trust` and `untrust` virtual-wire zones.

![Security zone configuration](./evidence/s16.png)

<br>

**Figure 18 — Address objects and security-rule review.** This captures the configured objects and the start of the ordered security rulebase.

![Address and security rulebase review](./evidence/s17.png)

<br>

**Figure 19 — NAT rule inspection and validation submission.** The candidate NAT output includes `LAB-OUTBOUND-SNAT`, followed by a `validate full` request.

![NAT configuration and validation submission](./evidence/s18.png)

[Back to contents](#contents)

---

<a id="commit"></a>
## 08 — Validation and Commit

The reviewed candidate configuration was submitted for full validation before manual commit.

```text
validate full
exit
show jobs all
configure
commit
exit
show jobs all
```

`validate full` submits a full candidate-configuration validation job. After leaving configuration mode with `exit`, `show jobs all` is used to inspect the job result rather than assuming that submission alone indicates success.

`configure` re-enters candidate-configuration mode. `commit` requests that PAN-OS apply the candidate changes to the running configuration. The final `exit` and `show jobs all` provide the post-commit job record.

**Figure 20 — Validation result and manual commit.** The CLI records validation job `4` and the subsequent manual commit job `5`.

![Validation job and commit execution](./evidence/s19.png)

<br>

**Figure 21 — Commit job history.** Validation job `4` and manual commit job `5` both report **`FIN / OK`**.

![PAN-OS validation and commit job results](./evidence/s20.png)

An earlier automatic commit, job `2`, appears in the historical job list with **`FIN / FAIL`**. Its cause is not established by the captured output. The later successful validation and manual commit are recorded separately, without attributing an unverified cause to the earlier failure.

[Back to contents](#contents)

---

<a id="verification"></a>
## 09 — Post-Change Verification

After the manual commit, operational commands were used to inspect system metadata, link state, routing, the deployed security rulebase, source NAT, and final configuration state.

```text
show system info
show interface all
show routing route
show running security-policy
show running nat-policy
show jobs all
set cli config-output-format set
show config running
```

**Command notes**

- `show system info` — Confirms platform, PAN-OS release and system metadata; device identifiers are redacted in the published capture.
- `show interface all` — Reviews interface addressing, zone/VR associations and physical link status.
- `show routing route` — Reads the operational routing table for the virtual router.
- `show running security-policy` — Displays the policy representation applied after commit.
- `show running nat-policy` — Displays the applied source-NAT policy.
- `show jobs all` — Rechecks validation and commit status.
- `set cli config-output-format set` — Requests set-style configuration output for supported displays; the captured follow-up output remains hierarchical.
- `show config running` — Reads the committed running configuration for final inspection.

**Figure 22 — System and software information.** The console identifies the PA-820 hardware and PAN-OS 11.1.4-h7.

![System information and PAN-OS version](./evidence/s21.png)

<br>

**Figure 23 — Interface state.** The defined interfaces and logical associations are visible; the relevant physical Ethernet links report **down**.

![Operational interface status](./evidence/s22.png)

<br>

**Figure 24 — Routing and running security policy.** `LAB-VR` reports **`total routes shown 0`**. The screenshot also begins the deployed security-policy output.

![Operational routes and running security rules](./evidence/s23.png)

<br>

**Figure 25 — Running NAT policy and job records.** The post-commit NAT output shows the outbound source translation and target interface.

![Running NAT policy and PAN-OS jobs](./evidence/s24.png)

<br>

**Figure 26 — Final NAT and running-configuration inspection.** The final capture includes running NAT, job status, the CLI output-format request and the beginning of running-configuration review.

![Final NAT and running configuration review](./evidence/s25.png)

The configured static route was present in the candidate configuration but **not active in the operational routing table** at the time of capture. With the physical interfaces down, no active forwarding path or end-to-end traffic result is claimed.

[Back to contents](#contents)

---

<a id="results"></a>
## 10 — Implementation Results

| Workstream | Recorded result |
|---|---|
| Appliance restoration | Factory reset and scrub each reported success in maintenance mode |
| Layer 3 configuration | Three interfaces addressed and assigned to security zones |
| Virtual routing | `LAB-VR` and named default-route configuration created |
| Security policy | SSH exception, interzone deny and SSL egress allow configured |
| Source NAT | Dynamic IP-and-port source translation defined for `ethernet1/5` |
| Configuration validation | Job `4` — **`FIN / OK`** |
| Manual commit | Job `5` — **`FIN / OK`** |
| Operational state | Interfaces down; operational routing table contained zero routes |

**Verification scope.** This record confirms restoration status, configured policy intent, candidate review, successful validation, successful commit, and post-change configuration inspection. It does not establish live packet delivery, policy-match test results, or a connected external network.

**Existing configuration.** The deployed security policy also includes a pre-existing `rule1` associated with the default lowercase `trust` and `untrust` zones. That rule is separate from the newly configured `LAB-*` policy and is not represented as part of the three-rule design.

**Evidence protection.** All 25 terminal screenshots have been redacted to exclude serial numbers, MAC addresses, sensitive device identifiers and credential hashes. The configured addresses, policy names and command output needed to explain the implementation remain visible.

[Back to contents](#contents)

---

<a id="repository"></a>
## 11 — Repository Structure

```text
.
├── README.md
├── assets/
│   ├── brandons-logo.png
│   └── topology.png
└── evidence/
    ├── s1.png
    ├── s2.png
    ├── ...
    └── s25.png
```

The `assets/` directory contains the project topology and personal branding. The `evidence/` directory contains 25 redacted terminal captures, embedded above in the order of the implementation record.

---

<p align="center">
  <strong>Brandon Stevenson</strong><br>
  IT Infrastructure · Enterprise Networking · Systems Administration · Software Programming
</p>

<p align="center">
  Microsoft Azure Fundamentals (AZ-900) · A.A.S. Computer Information Technology (Software Programming)<br>
  B.S. Cloud & Network Engineering — Western Governors University (In Progress)
</p>

<p align="center">
  <a href="https://www.brandons-resume.com/">Portfolio</a> &nbsp;·&nbsp;
  <a href="https://github.com/brandon-in-tech">GitHub</a> &nbsp;·&nbsp;
  <a href="https://www.linkedin.com/in/brandon-in-tech/">LinkedIn</a>
</p>
