<div align="center">

# Palo Alto PA-820 | Offline NGFW Configuration & Policy Implementation

**Physical Appliance Restoration · PAN-OS CLI · Layer 3 Segmentation · Security Policy · Source NAT · Configuration Validation**

**Platform:** Palo Alto Networks PA-820 &nbsp;|&nbsp; **PAN-OS:** 11.1.4-h7 &nbsp;|&nbsp; **Access:** Local serial console / Tera Term  
**Implementation Date:** November 9, 2026 &nbsp;|&nbsp; **Environment:** Offline, isolated hardware

</div>

---

## Project Overview

This project documents an **executed configuration on a physical Palo Alto Networks PA-820 next-generation firewall**. The work began in the appliance's boot and maintenance environment, progressed through a factory-reset and post-reset CLI inspection, and continued into a three-zone Layer 3 firewall design. Using PAN-OS CLI commands over the serial console, I defined Ethernet interfaces, security zones, a virtual router, a static default-route configuration, reusable address and service objects, application-aware security rules, and a source NAT policy.

The implementation was then inspected in the candidate configuration, validated, committed, and reviewed using operational commands. **The validation job and manual commit completed with `FIN / OK`.** The screenshots also document the important boundary of this demonstration: the configured physical interfaces were down, the runtime routing table reported zero routes, and no live traffic-flow or policy-match test results are captured. This is therefore **configuration and commit verification**, not proof of end-to-end network connectivity.

### What the evidence establishes

| Workstream | Recorded outcome |
|---|---|
| Appliance recovery | Bootloader, maintenance recovery screen, selected reset/scrub workflow, and success screens captured |
| Administrative preparation | Console login, password-change interaction, CLI output formatting, and system-log commands captured |
| Layer 3 network model | `ethernet1/3`, `ethernet1/4`, `ethernet1/5` configured with test addressing |
| Segmentation and routing | Three named Layer 3 security zones and virtual router `LAB-VR`; default-route configuration defined |
| Policy enforcement | Named SSH allow, server-access deny, and SSL allow security rules visible in the configuration |
| Address translation | `LAB-OUTBOUND-SNAT` dynamic IP-and-port translation configured and visible in deployed NAT policy |
| Change validation | PAN-OS validate job **4** and manual commit job **5** show **`FIN / OK`** |
| Offline limitation | Physical test interfaces show **down**; `show routing route` shows **0** active routes |

> **Evidence handling:** All 25 terminal screenshots have been preserved under [`evidence/`](./evidence/) with serial numbers, MAC addresses, unique IDs, credential hashes, and other sensitive identifiers redacted. Test IPs and rule names are intentionally retained because they explain the configuration. The reset/scrub result is an on-screen device status, **not an independent sanitization certification**.

## Table of Contents

1. [Environment and network design](#01--environment-and-network-design)
2. [Bootloader and factory restoration](#02--bootloader-and-factory-restoration)
3. [Administrative console and initial CLI](#03--administrative-console-and-initial-cli)
4. [Baseline configuration inspection](#04--baseline-configuration-inspection)
5. [Layer 3 interface assignment](#05--layer-3-interface-assignment)
6. [Security zones, virtual router, and static route](#06--security-zones-virtual-router-and-static-route)
7. [Address and service objects](#07--address-and-service-objects)
8. [Security rulebase implementation](#08--security-rulebase-implementation)
9. [Outbound source NAT](#09--outbound-source-nat)
10. [Candidate review, validation, and commit](#10--candidate-review-validation-and-commit)
11. [Operational verification and evidence](#11--operational-verification-and-evidence)
12. [Test plan versus verified execution](#12--test-plan-versus-verified-execution)
13. [Results, boundaries, and engineering notes](#13--results-boundaries-and-engineering-notes)
14. [Repository layout and references](#14--repository-layout-and-references)

---

## 01 | Environment and Network Design

**Hardware:** Palo Alto Networks PA-820; direct serial connection through Tera Term. The appliance was configured offline, without a live production network. CLI output identifies **PAN-OS 11.1.4-h7**.

### Intended logical topology

```text
                 LAB-TRUST                         LAB-SERVERS
              10.10.10.0/24                       10.20.20.0/24
                    |                                   |
       ethernet1/3: 10.10.10.1/24       ethernet1/4: 10.20.20.1/24
                    |                                   |
                    +-------- PA-820 / LAB-VR -----------+
                                     |
                       ethernet1/5: 203.0.113.2/30
                                     |
                               LAB-UNTRUST
                                     |
                       configured next hop: 203.0.113.1
                         (documentation-only address)
```

| Interface | Zone | Address | Purpose |
|---|---|---|---|
| `ethernet1/3` | `LAB-TRUST` | `10.10.10.1/24` | Trusted client segment and administrator source |
| `ethernet1/4` | `LAB-SERVERS` | `10.20.20.1/24` | Isolated server segment |
| `ethernet1/5` | `LAB-UNTRUST` | `203.0.113.2/30` | Simulated outside-facing interface and source-NAT egress |
| `LAB-VR` | — | `0.0.0.0/0` via `203.0.113.1` | Configured default path; **not present as an active route** in the final capture |

`10.10.10.0/24` and `10.20.20.0/24` are private lab subnets. `203.0.113.0/24` and `198.51.100.0/24` are documentation-address ranges; they do not represent a working public ISP connection in this project. Existing factory virtual-wire interfaces and their lowercase `trust`/`untrust` zones are distinct from these `LAB-*` zones.

[↑ Back to Table of Contents](#table-of-contents)

---

## 02 | Bootloader and Factory Restoration

The evidence begins during the appliance startup sequence. The console exposed the PAN-OS bootloader and hardware initialization output before entering the maintenance recovery environment.

**Evidence 01 — Physical appliance boot and hardware initialization.** The screen reports bootloader information, memory initialization, and storage detection. Hardware-specific identifiers are redacted.

![PA-820 bootloader showing initial hardware startup and memory initialization](./evidence/s1.png)

**Evidence 02 — Maintenance Recovery Tool.** The appliance is in its maintenance environment. This is the recovery interface used for subsequent reset operations.

![PAN-OS maintenance recovery tool main screen](./evidence/s2.png)

**Evidence 03 — Factory reset and scrub selection.** The device presents reset options and indicates a selected scrub method. This is a destructive operation; the image documents the selected on-screen workflow, not proof of compliance with any external erasure standard.

![Factory reset screen with scrub and reset options](./evidence/s3.png)

**Evidence 04 — Storage scrubbing in progress.** The recovery console displays the scrubbing operation and its progress output.

![Console output showing scrub operation in progress](./evidence/s4.png)

**Evidence 05 — Reset status and success.** The screen explicitly reports **`Scrub Status: Success`** and **`Factory Reset Status: Success`**.

![Recovery status showing successful scrub and factory reset](./evidence/s5.png)

**Engineering purpose:** Restore the appliance to a clean configuration starting point before building new policy objects. A factory reset and a data-scrubbing workflow are different operations; the evidence records both statuses without making broader security-certification claims.

[↑ Back to Table of Contents](#table-of-contents)

---

## 03 | Administrative Console and Initial CLI

After reboot, the console displays administrator login, an interactive password-change sequence, and the operational PAN-OS prompt `admin@PA-820>`.

```text
set cli terminal height 500
set cli terminal width 500
clear log system
show log system
```

| Command / console action | What it does and why it was used |
|---|---|
| Interactive administrator password change | Replaces the reset credential during the first login flow. The secret itself is not recorded in this repository. |
| `set cli terminal height 500` | Raises the terminal's display-height limit so long outputs are less likely to pause or paginate. Useful when capturing evidence. |
| `set cli terminal width 500` | Expands the CLI's logical output width to reduce line wrapping during large configuration reviews. It does **not** guarantee that the Tera Term window itself displays 500 columns. |
| `clear log system` | Deletes the local system-log records after confirmation. This is destructive and was recorded during this isolated reset procedure; it should not be treated as a normal production-maintenance step. |
| `show log system` | Reads system-log entries after the clear operation. The captured table shows no entries at that moment. |

**Evidence 06 — Login, CLI display controls, and system-log check.**

![Administrator login and terminal settings, system log removal and verification](./evidence/s6.png)

> **Operational caution:** In a managed environment, log deletion should occur only with authorization and in accordance with log-retention and incident-preservation requirements.

[↑ Back to Table of Contents](#table-of-contents)

---

## 04 | Baseline Configuration Inspection

The initial inspection checked what configuration was stored on the firewall after recovery. This established the starting state and provided comparison points for the new Layer 3 design.

```text
show vlan all
show config saved running-config.xml
show config saved candidate-config
show config running
show system disk-space
```

| Command | Terminal behavior / engineering purpose |
|---|---|
| `show vlan all` | Queries Layer 2 VLAN configuration. The screen responds `Server error : vlan configuration not found`, which is an observed result—not evidence that the appliance has no networking configuration whatsoever. |
| `show config saved running-config.xml` | Reads the saved running-configuration XML file, allowing a baseline review of settings retained on the appliance. |
| `show config saved candidate-config` | Displays the saved candidate-configuration content that can be compared with the current running configuration. |
| `show config running` | Displays currently committed configuration in hierarchical form in these screenshots, including administrative and shared sections. |
| `show system disk-space` | Reports filesystem capacity, used space, and available space. Useful for assessing the health of local storage before major changes. |

**Evidence 07 — VLAN query and saved running configuration.**

![VLAN query and saved running configuration output](./evidence/s7.png)

**Evidence 08 — Saved candidate configuration.**

![Saved candidate configuration contents](./evidence/s8.png)

**Evidence 09 — Saved running-configuration XML inspection.**

![Additional saved running configuration inspection](./evidence/s9.png)

**Evidence 10 — Running configuration inspection.**

![Current running configuration reviewed in the console](./evidence/s10.png)

**Evidence 11 — System metadata and disk-space verification.** The upper part of the image continues system information; the lower part records `show system disk-space`.

![System information followed by disk-space usage](./evidence/s11.png)

[↑ Back to Table of Contents](#table-of-contents)

---

## 05 | Layer 3 Interface Assignment

Configuration commands are entered after moving from operational mode (`>`) into configuration mode (`#`):

```text
configure
set network interface ethernet ethernet1/3 layer3 ip 10.10.10.1/24
set network interface ethernet ethernet1/4 layer3 ip 10.20.20.1/24
set network interface ethernet ethernet1/5 layer3 ip 203.0.113.2/30
```

| Command | Technical function |
|---|---|
| `configure` | Switches to PAN-OS candidate-configuration mode. `set` commands modify the pending configuration until committed. |
| `set network interface ethernet ethernet1/3 layer3 ip 10.10.10.1/24` | Assigns the trusted segment's Layer 3 gateway address to the third Ethernet interface. |
| `set network interface ethernet ethernet1/4 layer3 ip 10.20.20.1/24` | Assigns the server segment's Layer 3 gateway address to the fourth Ethernet interface. |
| `set network interface ethernet ethernet1/5 layer3 ip 203.0.113.2/30` | Sets a reserved documentation IP on the outside-facing test interface. This does not itself establish WAN connectivity. |

**Design rationale:** An explicit Layer 3 interface assignment allows routing and policy evaluation between separate IP subnets. The interfaces are later associated with security zones and `LAB-VR`; IP assignment alone does not complete forwarding or access-control configuration.

**Evidence 12 — CLI entry into configuration mode, three IP assignments, and the beginning of zone and virtual-router commands.**

![Creation of Layer 3 interfaces, security zones, and virtual router membership](./evidence/s12.png)

[↑ Back to Table of Contents](#table-of-contents)

---

## 06 | Security Zones, Virtual Router, and Static Route

### Security-zone membership

```text
set zone LAB-TRUST network layer3 ethernet1/3
set zone LAB-SERVERS network layer3 ethernet1/4
set zone LAB-UNTRUST network layer3 ethernet1/5
```

| Command | Technical function |
|---|---|
| `set zone LAB-TRUST network layer3 ethernet1/3` | Associates the trusted client-facing interface with the named Layer 3 security zone `LAB-TRUST`. |
| `set zone LAB-SERVERS network layer3 ethernet1/4` | Associates the server-facing interface with `LAB-SERVERS`. |
| `set zone LAB-UNTRUST network layer3 ethernet1/5` | Associates the simulated external interface with `LAB-UNTRUST`. |

Zones provide the **source and destination contexts** that later security and NAT rules match. A zone is a logical policy boundary, not a VLAN and not a guarantee of traffic isolation without appropriate rules.

### Virtual-router membership and configured default route

```text
set network virtual-router LAB-VR interface ethernet1/3
set network virtual-router LAB-VR interface ethernet1/4
set network virtual-router LAB-VR interface ethernet1/5
set network virtual-router LAB-VR routing-table ip static-route LAB-DEFAULT destination 0.0.0.0/0 interface ethernet1/5 nexthop ip-address 203.0.113.1
```

| Command | Technical function |
|---|---|
| `set network virtual-router LAB-VR interface ethernet1/3` | Adds the trusted segment's interface to virtual router `LAB-VR`. |
| `set network virtual-router LAB-VR interface ethernet1/4` | Adds the server segment's interface to the same routing instance. |
| `set network virtual-router LAB-VR interface ethernet1/5` | Adds the outside-facing test interface to that routing instance. |
| `set network virtual-router LAB-VR routing-table ip static-route LAB-DEFAULT ...` | Defines the named default route `LAB-DEFAULT`, matching destinations not reached by more specific routes, using `ethernet1/5` and next hop `203.0.113.1`. The line is shown here unwrapped for readability; the actual terminal displays it across lines. |

**Configured does not mean active:** The later operational route query displays `total routes shown 0`. With the physical interfaces down, this README does not claim the configured default route was installed in the forwarding table or that a gateway responded.

**Evidence 13 — Continuation of static-route configuration and subsequent address-object commands.**

![Continued default route setup and address object commands](./evidence/s13.png)

[↑ Back to Table of Contents](#table-of-contents)

---

## 07 | Address and Service Objects

Reusable PAN-OS objects make rulebases more consistent and auditable by keeping subnet and host definitions separate from individual rules.

```text
set address LAB-CLIENT-NET ip-netmask 10.10.10.0/24
set address LAB-SERVER-NET ip-netmask 10.20.20.0/24
set address LAB-ADMIN ip-netmask 10.10.10.10/32
set address LAB-SERVER ip-netmask 10.20.20.10/32
set service LAB-HTTPS protocol tcp port 443
```

| Command | Technical function |
|---|---|
| `set address LAB-CLIENT-NET ip-netmask 10.10.10.0/24` | Creates a named subnet object covering the trusted client network. |
| `set address LAB-SERVER-NET ip-netmask 10.20.20.0/24` | Creates a named subnet object covering the server network. |
| `set address LAB-ADMIN ip-netmask 10.10.10.10/32` | Defines one administrator endpoint as an exact `/32` host object. |
| `set address LAB-SERVER ip-netmask 10.20.20.10/32` | Defines one server endpoint as an exact `/32` host object. |
| `set service LAB-HTTPS protocol tcp port 443` | Creates a reusable TCP/443 service object. The documented SSL allow rule uses `application-default` instead of this object, so the object is created but not shown as referenced by that rule. |

**Evidence:** These commands appear in the second half of [s13.png](./evidence/s13.png), immediately following the routing configuration. The full image is embedded in Section 06. Object creation is independently checked later with `show address` in Section 10.

[↑ Back to Table of Contents](#table-of-contents)

---

## 08 | Security Rulebase Implementation

The policy design contains **three named lab rules**: a narrowly scoped SSH exception, a broad interzone server-access deny, and an application-aware SSL egress allow rule. PAN-OS processes security policy according to its ordered matching rules; rule placement and overlapping defaults matter.

### Rule A — Administrator SSH to a specific server

```text
set rulebase security rules LAB-Allow-Admin-SSH from LAB-TRUST to LAB-SERVERS source LAB-ADMIN destination LAB-SERVER application ssh service application-default action allow
```

This creates an allow rule from the trusted zone to the server zone **only when** the source is `LAB-ADMIN` (`10.10.10.10/32`), destination is `LAB-SERVER` (`10.20.20.10/32`), and identified application is `ssh`. `service application-default` requires the application to use its expected service ports, rather than permitting arbitrary TCP/UDP ports. The matching rule is an exception to the broader restriction defined next.

### Rule B — Deny other trusted-to-server access

```text
set rulebase security rules LAB-Deny-Server-Access from LAB-TRUST to LAB-SERVERS source LAB-CLIENT-NET destination LAB-SERVER-NET application any service any action deny
```

This defines an explicit deny for client-network traffic reaching the server-network subnet in the indicated source/destination zones, across all applications and services **that match this rule**. The narrower administrator SSH exception must precede the deny rule for the intended behavior.

### Rule C — Application-aware SSL egress

```text
set rulebase security rules LAB-Allow-HTTPS from LAB-TRUST to LAB-UNTRUST source LAB-CLIENT-NET destination any application ssl service application-default action allow
set rulebase security rules LAB-Allow-HTTPS log-end yes
```

The first command permits recognized `ssl` application traffic from the trusted client subnet toward the lab untrust zone, subject to the application's default service. The second enables **session-end logging** for that specific rule, providing an audit point when qualifying traffic is actually generated. Creating the logging setting does not itself prove that any traffic was logged.

**Evidence 14 — Security-rule entry and SSL session-end logging.** The screenshot captures the final portion of the policy configuration, including the explicit `log-end yes` command. The full rule definitions are subsequently inspected with `show rulebase security` and `show running security-policy`.

![Creation of security rules and SSL end-of-session logging](./evidence/s14.png)

> **Existing configuration note:** Final running-policy output also contains a factory/default-style `rule1` in lowercase `trust` → `untrust` zones. Those are separate from `LAB-TRUST`/`LAB-UNTRUST`. Its continued presence is explicitly documented rather than implying that the newly defined lab rules are the entire device policy.

[↑ Back to Table of Contents](#table-of-contents)

---

## 09 | Outbound Source NAT

```text
set rulebase nat rules LAB-OUTBOUND-SNAT from LAB-TRUST to LAB-UNTRUST source LAB-CLIENT-NET destination any service any to-interface ethernet1/5 source-translation dynamic-ip-and-port interface-address interface ethernet1/5
```

**What the command establishes:**

| Field | Technical meaning |
|---|---|
| `LAB-OUTBOUND-SNAT` | Descriptive identifier for the NAT rule |
| `from LAB-TRUST to LAB-UNTRUST` | Original-packet ingress and destination-zone match |
| `source LAB-CLIENT-NET` | Translation applies to traffic sourced from the configured client subnet |
| `destination any service any` | Does not narrow NAT eligibility by destination host or service |
| `to-interface ethernet1/5` | Matches egress via the specified outside test interface |
| `source-translation dynamic-ip-and-port` | Performs dynamic source-address/port translation (PAT) for matching sessions |
| `interface-address interface ethernet1/5` | Uses the selected interface's IP address as the translated source (configured `203.0.113.2`) |

This is an **outbound source translation**, not destination NAT, port forwarding, or proof that internet traffic was transmitted. PAN-OS security-policy allowance and routing must also support any intended flow.

The NAT rule is inspected in **[s18.png](./evidence/s18.png)** and later visible as deployed policy in **[s24.png](./evidence/s24.png)** and **[s25.png](./evidence/s25.png)**.

[↑ Back to Table of Contents](#table-of-contents)

---

## 10 | Candidate Review, Validation, and Commit

### Candidate-configuration review

```text
show network
show zone
show address
show rulebase security
show rulebase nat
```

| Command | What was checked |
|---|---|
| `show network` | Displays the pending network configuration: Layer 3 interfaces, their IP addresses, and related network settings. |
| `show zone` | Confirms named security-zone objects and their interface associations. |
| `show address` | Confirms the reusable client, server, admin, and host address objects. |
| `show rulebase security` | Inspects rule definitions, their match conditions, actions, and displayed ordering in the candidate configuration. |
| `show rulebase nat` | Confirms the dynamic source-NAT rule and its interface-address translation behavior. |

**Evidence 15 — Network configuration review.**

![Candidate network configuration including Layer 3 addresses](./evidence/s15.png)

**Evidence 16 — Zone associations.** The lower sections contain `LAB-TRUST`, `LAB-SERVERS`, and `LAB-UNTRUST` mapped to their corresponding Ethernet interfaces.

![Security-zone configuration with lab interfaces assigned](./evidence/s16.png)

**Evidence 17 — Address objects and security-rulebase review.** The output confirms four address objects and begins displaying the policy rule definitions, including administrator SSH and the server-access deny.

![Address object inspection and security-rulebase output](./evidence/s17.png)

### Validation and transactional commit

```text
validate full
exit
show jobs all
configure
commit
exit
show jobs all
```

| Command | Effect and interpretation |
|---|---|
| `validate full` | Submits a full candidate-configuration validation job. A queued job is not the same as successful validation; its final status is checked separately. |
| `exit` (from `#`) | Returns from candidate-configuration mode to operational mode (`>`). |
| `show jobs all` | Lists background operations with job IDs, state, result, and completion data. Here it is used to inspect validation and commit results. |
| `configure` | Returns to configuration mode for a manual commit. |
| `commit` | Requests that the validated candidate configuration be applied as the running configuration. This is the actual deployment step; unlike a `set` command, it makes the change operational when successful. |
| `exit` (after commit) | Returns to operational mode for verification. |
| `show jobs all` (final) | Confirms the outcome of the committed change, rather than inferring success from the absence of an immediate CLI error. |

**Evidence 18 — NAT candidate review and validation submission.** `validate full` returns **validation job ID 4**.

![NAT rule configuration and full validation job queued](./evidence/s18.png)

**Evidence 19 — Job status review and commit execution.** The image includes the validation results and the manual invocation of `commit`, with **commit job ID 5** in progress.

![Validation job review and commit launch](./evidence/s19.png)

**Evidence 20 — Completed job status.** The later `show jobs all` confirms **job 4: `FIN / OK` (validate)** and **job 5: `FIN / OK` (commit)**. An **earlier automatic commit job 2 shows `FIN / FAIL`** in the same historical list; that older status is not concealed or conflated with the successful manual commit.

![Job history confirming final full validation and manual commit successful](./evidence/s20.png)

**Change-control interpretation:** The successful manual commit establishes that PAN-OS accepted and applied the candidate configuration. It does **not** independently establish reachability, routing convergence, physical link state, or successful data-plane sessions.

[↑ Back to Table of Contents](#table-of-contents)

---

## 11 | Operational Verification and Evidence

Once the manual commit completed, the console was used to inspect runtime and committed state.

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

| Command | Reason for use and observed significance |
|---|---|
| `show system info` | Identifies the physical platform, installed PAN-OS build, hostname, and management/system metadata; device-specific identifiers were redacted. Shows **PA-820 / PAN-OS 11.1.4-h7**. |
| `show interface all` | Displays physical and logical interfaces, link state, configured addresses, and zone/VR assignments. The test Ethernet ports are listed as **down**, while the logical configuration is present. MAC addresses are redacted. |
| `show routing route` | Queries the operational routing table. The image displays **`VIRTUAL ROUTER: LAB-VR`** followed by **`total routes shown 0`**—a decisive distinction from the configured static-route statement. |
| `show running security-policy` | Displays the compiled/deployed running security policy with indexed rules and effective match conditions; it demonstrates the lab rules are present after commit. |
| `show running nat-policy` | Displays the deployed NAT policy and shows `LAB-OUTBOUND-SNAT` translating to the `ethernet1/5` interface address. |
| `show jobs all` | Rechecks final management-plane job results, including the successful validation and manual commit. |
| `set cli config-output-format set` | Requests set-style configuration output for supported CLI configuration displays. The subsequent captured `show config running` view is hierarchical; this README does not misrepresent it as a displayed line-by-line `set` export. |
| `show config running` | Performs a final inspection of the committed running configuration. Sensitive credential values in the displayed output have been masked. |

**Evidence 21 — Device metadata and PAN-OS software version.**

![Show system info with PA-820 hardware and PAN-OS version](./evidence/s21.png)

**Evidence 22 — Physical and logical interface state.** The `LAB-*` zone/virtual-router mappings are visible, but Ethernet link state is **down**. This is consistent with the explicitly offline setup.

![Show interface all confirming lab interfaces and their down operational state](./evidence/s22.png)

**Evidence 23 — Operational routing table and running security policy.** The route query reports **zero active routes** in `LAB-VR`; the same capture begins the post-commit running security policy output.

![Operational routing table with zero routes and deployed security policy](./evidence/s23.png)

**Evidence 24 — Running NAT policy and job history.** The NAT output identifies the source translation and egress interface, followed by commit and validation job evidence.

![Deployed NAT policy and final job history](./evidence/s24.png)

**Evidence 25 — NAT verification and final running-configuration capture.** This final screen includes deployed NAT, successful job records, a CLI output-format request, and the start of a running-configuration inspection.

![Final NAT policy, job results, and running configuration review](./evidence/s25.png)

[↑ Back to Table of Contents](#table-of-contents)

---

## 12 | Test Plan Versus Verified Execution

The supplied CLI guide also contains these **planned offline policy-matching commands**. They are included for reproducibility, but **the 25 screenshots do not show these commands being executed or their results**. They must not be presented as completed tests.

```text
# Planned test A — authorized administrator SSH
 test security-policy-match from LAB-TRUST to LAB-SERVERS source 10.10.10.10 destination 10.20.20.10 application ssh protocol 6 destination-port 22

# Planned test B — another client attempting SSH to the same server
 test security-policy-match from LAB-TRUST to LAB-SERVERS source 10.10.10.20 destination 10.20.20.10 application ssh protocol 6 destination-port 22

# Planned test C — client SSL traffic toward a documentation endpoint
 test security-policy-match from LAB-TRUST to LAB-UNTRUST source 10.10.10.20 destination 198.51.100.10 application ssl protocol 6 destination-port 443

# Planned test D — matching an outbound NAT flow
 test nat-policy-match from LAB-TRUST to LAB-UNTRUST source 10.10.10.20 destination 198.51.100.10 source-port 50000 destination-port 443 protocol 6 to-interface ethernet1/5
```

| Planned CLI command | What it is intended to establish | Evidence status |
|---|---|---|
| `test security-policy-match` (admin SSH) | Which security rule would match an SSH flow from the named admin host to the protected server | **Not captured** |
| `test security-policy-match` (other client SSH) | Whether a non-admin client would be matched by the broader deny rule | **Not captured** |
| `test security-policy-match` (SSL) | Which security rule would match a candidate TLS/SSL egress session | **Not captured** |
| `test nat-policy-match` | Which source-NAT rule would be selected for a proposed outbound TCP session | **Not captured** |

These are **control-plane match simulations**, not substitutes for generating and inspecting actual end-to-end traffic. They are preserved here only as unexecuted follow-up steps from the original written guide.

[↑ Back to Table of Contents](#table-of-contents)

---

## 13 | Results, Boundaries, and Engineering Notes

### Confirmed outcomes

- Restored the PA-820 through its maintenance factory-reset workflow; screenshots display both scrub and factory-reset success statuses.
- Accessed the appliance via serial CLI and performed initial administrative inspection.
- Built a three-interface, three-zone Layer 3 configuration and assigned the interfaces to a dedicated virtual router.
- Defined client, server, and administrator address objects; an additional TCP/443 service object was created.
- Defined administrator SSH allow, server-segment access deny, and SSL egress allow rules, including SSL end-of-session logging.
- Defined dynamic-IP-and-port outbound source NAT against the interface address of `ethernet1/5`.
- Confirmed **full configuration validation and manual commit completed successfully** (`FIN / OK`) and inspected running security/NAT policies.

### Boundaries visible in the capture

- **No traffic verification:** There are no screenshots of successful packets, active sessions, traffic logs, ping responses, or completed policy-match tests.
- **No active routes:** `show routing route` reports **0** routes for `LAB-VR` at verification time. The configured static default route should not be described as an active working route.
- **Interfaces offline:** `show interface all` reports test Ethernet links down, so there is no evidence of physical uplink connectivity.
- **Pre-existing policy:** The running policy includes an existing `rule1` involving default lowercase zones. These were retained in the visible configuration and would need review before any production deployment.
- **No production authorization implied:** The isolated addressing and observed validation do not demonstrate production readiness. Security profiles, logging retention, management exposure, HA design, threat-content currency, and live connectivity need independent review for deployment.
- **Credential and ID protection:** Evidence is intentionally sanitized. The README does not publish login passwords, credential hashes, serial numbers, or MAC addresses.

### Engineering takeaway

This project demonstrates an end-to-end **management-plane implementation workflow** on physical enterprise firewall hardware: recovery, baseline assessment, object-oriented configuration, interzone access control, source-NAT definition, validation, transactional commit, and evidence-based post-change review. The final documentation distinguishes **accepted configuration** from **observed operational behavior**, which is essential in defensible infrastructure engineering reporting.

[↑ Back to Table of Contents](#table-of-contents)

---

## 14 | Repository Layout and References

```text
PA820-Offline-NGFW/
├── README.md
└── evidence/
    ├── s1.png
    ├── s2.png
    ├── ...
    └── s25.png
```

All screenshot paths are relative to `README.md`. To publish the project on GitHub, upload **the README and the entire `evidence/` directory together** so every inline image resolves correctly.

The original three-page CLI guide supplied with the project was used to cross-reference command intent. The screenshots—not the guide alone—determine whether this README calls an action executed, validated, committed, or unverified.

---

<div align="center">

### Brandon Stevenson

**IT Infrastructure · Enterprise Networking · Systems Administration · Programming & Automation**

Microsoft Azure Fundamentals (AZ-900) · A.A.S. in Computer Information Technology (Software Programming)  
B.S. Cloud & Network Engineering — Western Governors University (In Progress)

[**Professional Portfolio**](https://www.brandons-resume.com/) &nbsp;·&nbsp; [**GitHub**](https://github.com/brandon-in-tech) &nbsp;·&nbsp; [**LinkedIn**](https://www.linkedin.com/in/brandon-in-tech/)

</div>
