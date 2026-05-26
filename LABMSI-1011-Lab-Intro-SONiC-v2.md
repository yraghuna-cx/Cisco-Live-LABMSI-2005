# LABMSI-1011 - Try SONiC { Introductory } - Open Network Operating System powering the Next-gen AI-Data Center

**Objective:** Get comfortable navigating a SONiC device — versions, interfaces, configuration files, the Redis database, and saving config.

**Duration:** 45–60 minutes

**Prerequisites:**
- A running SONiC device (lab leaf, e.g. `leaf00`) reachable over SSH
- Admin credentials (`admin` user)
- Basic Linux / CLI familiarity

---

## Table of Contents

- [Lab Overview](#lab-overview)
- [Lab Topology](#lab-topology)
- [Lab Setup Verification](#lab-setup-verification)
  - [Step 1: SSH to the Virtual Host](#step-1-ssh-to-the-virtual-host)
  - [Step 2: Confirm Reachability](#step-2-confirm-reachability)
  - [Step 3: SSH Access](#step-3-ssh-access)
- [Exercise 1: Device & Platform Identity](#exercise-1-device--platform-identity-10-minutes)
  - [Task 1.1: Check the SONiC Version](#task-11-check-the-sonic-version)
  - [Task 1.2: Platform Details](#task-12-platform-details)
  - [Task 1.3: Boot Information](#task-13-boot-information)
- [Exercise 2: Interfaces and Counters](#exercise-2-interfaces-and-counters-10-minutes)
  - [Task 2.1: Interface Status](#task-21-interface-status)
  - [Task 2.2: IP Interfaces](#task-22-ip-interfaces)
  - [Task 2.3: Interface Counters](#task-23-interface-counters)
  - [Task 2.4: Interface Errors](#task-24-interface-errors)
  - [Task 2.5: Realtime Monitoring of Counters](#task-25-realtime-monitoring-of-counters)
  - [Task 2.6: Clearing Counters](#task-26-clearing-counters)
- [Exercise 3: Running Configuration](#exercise-3-running-configuration-10-minutes)
  - [Task 3.1: SONiC Running Config](#task-31-sonic-running-config)
  - [Task 3.2: FRR / Routing Running Config](#task-32-frr--routing-running-config)
  - [Task 3.3: Inspect BGP Sessions and Routes](#task-33-inspect-bgp-sessions-and-routes)
- [Exercise 4: Startup Configuration via `jq`](#exercise-4-startup-configuration-via-jq-10-minutes)
  - [Task 4.1: View the Whole File](#task-41-view-the-whole-file)
  - [Task 4.2: Pull Specific Sections](#task-42-pull-specific-sections)
- [Exercise 5: SONiC Architecture via Redis](#exercise-5-sonic-architecture-via-redis-10-minutes)
  - [Task 5.1: List Keys in Config_DB (4)](#task-51-list-keys-in-config_db-4)
  - [Task 5.2: List Keys in Appl_DB (0)](#task-52-list-keys-in-appl_db0)
  - [Task 5.3: List Keys in ASIC (1)](#task-53-list-keys-in-asic1)
  - [Task 5.4: List Keys in State_DB (6)](#task-54-list-keys-in-state_db6)
  - [Task 5.5: List Keys in Counters_DB (2)](#task-55-list-keys-in-counters_db2)
  - [Task 5.6: Inspect Specific Entries](#task-56-inspect-specific-entries)
- [Exercise 6: Saving Configuration](#exercise-6-saving-configuration-5-minutes)
  - [Task 6.1: Change the Hostname](#task-61-change-the-hostname-of-the-device)
  - [Task 6.2: Save the Running Config](#task-62-save-the-running-config)
  - [Task 6.3: Verify the Save](#task-63-verify-the-save)
  - [Task 6.4: FRR / Routing Save](#task-64-frr--routing-save)
- [Lab Wrap-Up](#lab-wrap-up)
  - [Cheat Sheet](#cheat-sheet)

---

## Lab Overview

SONiC is Linux underneath, with a CLI layer (`show`, `config`), an FRR routing stack (`vtysh`), and a Redis-based state database that ties it all together. In this lab you will:

- SSH into a SONiC device and identify the image and platform
- Inspect interfaces, counters, and IP information
- Read the running and startup configurations
- Explore the SONiC architecture through Redis
- Save configuration changes the right way

> **Note:** The lab device is a **containerlab** node running real SONiC on a virtual **Cisco-8101-32FH-O** hardware. All CLI commands, Redis, and config files behave exactly as they do on physical hardware.

---

## Lab Topology

![alt text](image-2.png)

> **Note:** Some nodes may be unavailable due to virtual-host resource limit> s. You only need access to one of the four devices (leaf00/01 or spine00/01) to work through this introductory SONiC lab.

## Lab Setup Verification

### Step 1: SSH to the virtual host
Use your putty sessions and land on the virtual host

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa cisco@198.18.128.11
#password:cisco123
cd /home/cisco/cisco8000e/sonic-clab/c8101/ansible
```

**Expected Output:*

```
/home/cisco/cisco8000e/sonic-clab/c8101/ansible
```

### Step 2: Confirm Reachability

```bash
ping -c 3 leaf00
```

### Step 3: SSH Access
```bash
ssh admin@leaf00
# Default password: password (or as provided)
```
**Expected Output:**

```text
Linux leaf00 6.1.0-11-2-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.38-4 (2023-08-08) x86_64
You are on
  ____   ___  _   _ _  ____
 / ___| / _ \| \ | (_)/ ___|
 \___ \| | | |  \| | | |
  ___) | |_| | |\  | | |___
 |____/ \___/|_| \_|_|\____|

-- Software for Open Networking in the Cloud --

Unauthorized access and/or use are prohibited.
All access and/or use are subject to monitoring.

Help:    https://sonic-net.github.io/SONiC/

Last login: Mon May 25 18:16:52 2026 from 172.20.2.1
admin@leaf00:~$ 
```

Expected: a prompt like `admin@leaf00:~$`

---

## Exercise 1: Device & Platform Identity (10 minutes)

### Task 1.1: Check the SONiC Version
```bash
show version
```
**Expected Output:**

```text
SONiC Software Version: SONiC.202405c.38617-int-20260325.164610
SONiC OS Version: 12
Distribution: Debian 12.13
Kernel: 6.1.0-11-2-amd64
Build commit: 013c9d877
Build date: Thu Mar 26 01:13:11 UTC 2026
Built by: sonicci@sonic-ci-15-lnx.cisco.com

Platform: x86_64-8101_32fh_o-r0
HwSKU: Cisco-8101-32FH-O
ASIC: cisco-8000
ASIC Count: 1
Serial Number: CSNMFGJGPUQ
Model Number: 8101-32FH-O
Hardware Revision: 0.33
Uptime: 18:38:42 up  1:10,  1 user,  load average: 1.13, 1.49, 1.71
Date: Mon 25 May 2026 18:38:42

Docker images:
REPOSITORY                    TAG                                 IMAGE ID       SIZE
docker-orchagent              202405c.38617-int-20260325.164610   d67fa45d56de   385MB
docker-orchagent              latest                              d67fa45d56de   385MB
...
...

```
Decoding the SONiC version string

![alt text](image.png)

**Look for:**
- SONiC Software Version
- Distribution (Debian)
- Kernel
- ASIC type
- Hardware / platform string
- Uptime

**Quick filters:**
```bash
# Just the version line
show version | grep -i "software version"
```

**Expected Output:**

```text
SONiC Software Version: SONiC.202405c.38617-int-20260325.164610
```

```bash
# First 5 lines only
show version | head -n 5
```
**Expected Output:**

```text

SONiC Software Version: SONiC.202405c.38617-int-20260325.164610
SONiC OS Version: 12
Distribution: Debian 12.13
Kernel: 6.1.0-11-2-amd64
```

---

### Task 1.2: Platform Details
```bash
# Platform-specific version info (BIOS, ONIE, FPGA, etc.)
show platform summary
```

**Expected Output:**

```text
Platform: x86_64-8101_32fh_o-r0
HwSKU: Cisco-8101-32FH-O
ASIC: cisco-8000
ASIC Count: 1
Serial Number: CSNMFGJGPUQ
Model Number: 8101-32FH-O
Hardware Revision: 0.33
Switch Type: switch
```

**Observation**

- Cisco-8101-32FH-O is the platform being used.
- ASIC type and count: `cisco-8000`, single ASIC — confirms this is a **fixed** (non-modular) platform.

```bash
# Platform-specific version info (BIOS, ONIE, FPGA, etc.)
show platform version
```
**Expected Output:**

```text
Cisco Platform Release Version: test-tag-pull-145-gbc22210a
Cisco Platform Build commit Version: bc22210aa16352adcd7b5a787f4c2dc863aa814b
Cisco Whitebox BSP Version: 0.4-692-gb6df1ed8
Cisco Whitebox FPD Version: 0.9-109-g06a763e-bookworm
Cisco Q200 Silicon One SDK Version: 24.11.4140.18-sai-1.14.0-bullseye-9e9109f6320
Cisco Q200 Silicon One SDK Validation Version: 24.11.4140.18
```
**Observation**

- Cisco **Q200 Silicon One** ASIC is in use.
- Infra software **SDK Version is 24.11 train**.
- BSP / FPD versions are reported separately — useful when correlating with TAC for hardware diagnostics.

---

### Task 1.3: Boot Information

```bash
show boot
```

**Expected Output:**

```text
Current: SONiC-OS-202405c.38617-int-20260325.164610
Next: SONiC-OS-202405c.38617-int-20260325.164610
Available: 
SONiC-OS-202405c.38617-int-20260325.164610
```

**Observation**

SONiC supports dual-image boot (A/B partitions).

- `Current` is the running image (what `show version` reports).
- `Next` is what will boot after a reload — if it differs from `Current`, an upgrade is pending.
- `Available` lists all installed images (SONiC supports A/B dual-image boot for safe upgrade and rollback).
- In this output, only **one image** is installed — there is no rollback target. After an upgrade, you'll see two entries here.

---

## Exercise 2: Interfaces and Counters (10 minutes)

### Task 2.1: Interface Status
```bash
show interfaces status
```
**Expected Output:**

```text
  Interface                                    Lanes    Speed    MTU    FEC    Alias    Vlan    Oper    Admin                                             Type    Asym PFC
-----------  ---------------------------------------  -------  -----  -----  -------  ------  ------  -------  -----------------------------------------------  ----------
  Ethernet0  2304,2305,2306,2307,2308,2309,2310,2311     400G   9100    N/A     etp0  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
  Ethernet8  2320,2321,2322,2323,2324,2325,2326,2327     400G   9100    N/A     etp1  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
 Ethernet16  2312,2313,2314,2315,2316,2317,2318,2319     400G   9100    N/A     etp2  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
 Ethernet24  2056,2057,2058,2059,2060,2061,2062,2063     400G   9100    N/A     etp3  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
 Ethernet32  1792,1793,1794,1795,1796,1797,1798,1799     400G   9100    N/A     etp4  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
 Ethernet40  2048,2049,2050,2051,2052,2053,2054,2055     400G   9100    N/A     etp5  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
 Ethernet48  2560,2561,2562,2563,2564,2565,2566,2567     400G   9100    N/A     etp6  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
 Ethernet56  2824,2825,2826,2827,2828,2829,2830,2831     400G   9100    N/A     etp7  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
 Ethernet64  2832,2833,2834,2835,2836,2837,2838,2839     400G   9100    N/A     etp8  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
 Ethernet72  2816,2817,2818,2819,2820,2821,2822,2823     400G   9100    N/A     etp9  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
 Ethernet80  2568,2569,2570,2571,2572,2573,2574,2575     400G   9100    N/A    etp10  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
 Ethernet88  2576,2577,2578,2579,2580,2581,2582,2583     400G   9100    N/A    etp11  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
 Ethernet96  1536,1537,1538,1539,1540,1541,1542,1543     400G   9100    N/A    etp12  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet104  1800,1801,1802,1803,1804,1805,1806,1807     400G   9100    N/A    etp13  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet112  1552,1553,1554,1555,1556,1557,1558,1559     400G   9100    N/A    etp14  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet120  1544,1545,1546,1547,1548,1549,1550,1551     400G   9100    N/A    etp15  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet128  1296,1297,1298,1299,1300,1301,1302,1303     400G   9100    N/A    etp16  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet136  1288,1289,1290,1291,1292,1293,1294,1295     400G   9100    N/A    etp17  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet144  1280,1281,1282,1283,1284,1285,1286,1287     400G   9100    N/A    etp18  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet152  1032,1033,1034,1035,1036,1037,1038,1039     400G   9100    N/A    etp19  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet160          264,265,266,267,268,269,270,271     400G   9100    N/A    etp20  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet168          272,273,274,275,276,277,278,279     400G   9100    N/A    etp21  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet176                  16,17,18,19,20,21,22,23     400G   9100    N/A    etp22  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet184                          0,1,2,3,4,5,6,7     400G   9100    N/A    etp23  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet192          256,257,258,259,260,261,262,263     400G   9100    N/A    etp24  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet200                    8,9,10,11,12,13,14,15     400G   9100    N/A    etp25  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet208  1024,1025,1026,1027,1028,1029,1030,1031     400G   9100    N/A    etp26  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet216          768,769,770,771,772,773,774,775     400G   9100    N/A    etp27  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet224          520,521,522,523,524,525,526,527     400G   9100    N/A    etp28  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet232          776,777,778,779,780,781,782,783     400G   9100    N/A    etp29  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet240          512,513,514,515,516,517,518,519     400G   9100    N/A    etp30  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
Ethernet248          528,529,530,531,532,533,534,535     400G   9100    N/A    etp31  routed      up       up  QSFP-DD Double Density 8X Pluggable Transceiver         N/A
admin@leaf00:~$ 
```
**Observation**

- **32 front-panel ports** are present (`Ethernet0` … `Ethernet248`, aliased `etp0`–`etp31`) — matches the **8101-32FH-O** platform (32 × 400G QSFP-DD).
- **Port naming follows lane stride of 8**: `Ethernet0, 8, 16, 24, …`. Each port consumes **8 SerDes lanes** at 50G PAM4 = **400G** total. This is why port numbers jump by 8, not 1.
- **Alias `etpN`** = "Ethernet **P**ort **N**" — the *physical* faceplate label the operator sees. `EthernetX` is the internal SONiC name (lane-based). Always map alias ↔ Ethernet when talking to customers.
- **Lanes** column shows the actual ASIC SerDes lanes bound to that port (e.g. `Ethernet0` → lanes `2304–2311`). Useful for breakout planning and ASIC-level debugging.
- **Speed = 400G** on every port → no breakout configured. This platform supports breakout to 2×200G, 4×100G, 8×50G, etc., via `config interface breakout`.
- **MTU = 9100** everywhere → jumbo frames enabled (standard for fabric / AI workloads). Default SONiC MTU is 9100, not 1500.
- **FEC = N/A** → FEC mode is being inherited from the transceiver/SDK default (typically RS-FEC for 400G QSFP-DD). Explicit FEC config is empty in CONFIG_DB. For production fabrics, set FEC explicitly.
- **Vlan = routed** → every port is in L3 (routed) mode, not switched. This is a pure **routed leaf** posture — expected for an IP/BGP fabric (EVPN, AI fabric, etc.).
- **Oper = up / Admin = up** on all 32 ports → fabric is fully cabled and healthy at L1. No down/disabled ports.
- **Type = QSFP-DD Double Density 8X Pluggable Transceiver** → all 32 cages populated with 400G QSFP-DD optics/DACs. Use `show interfaces transceiver eeprom` to see vendor/PN per port.
- **Asym PFC = N/A** → asymmetric PFC not configured. For lossless RoCEv2 / AI traffic you'd typically enable PFC on specific priorities (3 or 4) via QoS profiles.

---

### Task 2.2: IP Interfaces
```bash
show ip interfaces
```

**Expected Output:**

```text
Interface    Master    IPv4 address/mask    Admin/Oper    BGP Neighbor    Neighbor IP
-----------  --------  -------------------  ------------  --------------  -------------
Ethernet0              10.1.1.1/31          up/up         N/A             N/A
Ethernet8              10.1.1.5/31          up/up         N/A             N/A
Loopback0              10.0.0.2/32          up/up         N/A             N/A
docker0                240.127.1.1/24       up/down       N/A             N/A
eth0                   172.20.2.100/24      up/up         N/A             N/A
eth4                   192.168.123.79/24    up/up         N/A             N/A
lo                     127.0.0.1/16         up/up         N/A             N/A
```
**Observation**

- **Two front-panel L3 links are configured** — `Ethernet0` (`10.1.1.1/31`) and `Ethernet8` (`10.1.1.5/31`). These are the **uplinks to the spines** (one /31 per spine), classic IP fabric design.
- **/31 masks on point-to-point links** → two usable IPs per subnet, no broadcast waste. Standard practice in BGP-routed fabrics.
- **`Loopback0 = 10.0.0.2/32`** → the device's **stable router-id / BGP source IP**. Loopbacks survive physical link flaps, so BGP sessions and overlays (VXLAN/EVPN VTEP) anchor here.
- **`eth0 = 172.20.2.100/24`** → out-of-band **management interface** (matches the `mgmt-ipv4` from the containerlab topology). Lives in its own mgmt VRF/namespace, not in the data-plane routing table.
- **`eth4 = 192.168.123.79/24`** → a second management/auxiliary link (lab-specific — often the host bridge in clab). Not part of the fabric.
- **`docker0 = 240.127.1.1/24`, Oper = down** → internal Docker bridge used by SONiC containers to talk to each other. `down` is **normal** here — it's only `up` when containers actively bind to it. Not a fault.
- **`lo = 127.0.0.1/16`** → standard Linux loopback (not `Loopback0`). Used by local processes; ignore for fabric purposes.
- **All interfaces show `Admin/Oper = up/up`** except `docker0` — fabric L3 is healthy.
- **`BGP Neighbor` and `Neighbor IP` columns are `N/A`** → **no BGP sessions are configured yet** on these interfaces. The plumbing (IPs, links up) is ready, but routing hasn't been brought up. Next step is configuring BGP under FRR (`vtysh`) or via `config bgp` to peer with the spines.
- **No VRF assignments** (`Master` column empty) → everything is in the **default VRF**. In multi-tenant or mgmt-VRF designs you'd see `mgmt` or tenant VRF names here.

---

### Task 2.3: Interface Counters
```bash
# Snapshot
show interfaces counters
```
**Expected Output:**

```
      IFACE    STATE    RX_OK     RX_BPS    RX_UTIL    RX_ERR    RX_DRP    RX_OVR    TX_OK     TX_BPS    TX_UTIL    TX_ERR    TX_DRP    TX_OVR
-----------  -------  -------  ---------  ---------  --------  --------  --------  -------  ---------  ---------  --------  --------  --------
  Ethernet0        U       12   0.00 B/s      0.00%         0         0         0      336   0.47 B/s      0.00%         0         0         0
  Ethernet8        U      348  13.75 B/s      0.00%         0         9         0      343  13.75 B/s      0.00%         0         0         0
 Ethernet16        U      103   0.75 B/s      0.00%         0         5         0      104   0.47 B/s      0.00%         0         0         0
 Ethernet24        U        0   0.00 B/s      0.00%         0         0         0      104   0.47 B/s      0.00%         0         0         0
 Ethernet32        U        0   0.00 B/s      0.00%         0         0         0      104   0.47 B/s      0.00%         0         0         0
 Ethernet40        U        0   0.00 B/s      0.00%         0         0         0      104   0.47 B/s      0.00%         0         0         0
 ...
 ...
```

----

### Task 2.4: Interface Errors

```bash
# Errors only
show interfaces counters errors
```

**Expected Output:**

```
      IFACE    STATE    RX_ERR    RX_DRP    RX_OVR    TX_ERR    TX_DRP    TX_OVR
-----------  -------  --------  --------  --------  --------  --------  --------
  Ethernet0        U         0         0         0         0         0         0
  Ethernet8        U         0         9         0         0         0         0
 Ethernet16        U         0         5         0         0         0         0
 Ethernet24        U         0         0         0         0         0         0
 ...
 ...
```

---

### Task 2.5: Realtime Monitoring of Counters

```bash
# Live (refresh every 5s)
watch -n 5 'show interfaces counters'
```
**Expected Output:**

```
Every 5.0s: show interfaces counters                                                                                                                                                                            leaf00: Mon May 25 19:06:36 2026

      IFACE    STATE    RX_OK     RX_BPS    RX_UTIL    RX_ERR    RX_DRP    RX_OVR    TX_OK     TX_BPS    TX_UTIL    TX_ERR    TX_DRP    TX_OVR
-----------  -------  -------  ---------  ---------  --------  --------  --------  -------  ---------  ---------  --------  --------  --------
  Ethernet0        U       12   0.00 B/s      0.00%         0         0         0      361  16.74 B/s      0.00%         0         0         0
  Ethernet8        U      369  28.91 B/s      0.00%         0         9         0      363  24.17 B/s      0.00%         0         0         0
 Ethernet16        U      110  32.81 B/s      0.00%         0         5         0      111  16.80 B/s      0.00%         0         0         0
 Ethernet24        U        0   0.00 B/s      0.00%         0         0         0      111  16.80 B/s      0.00%         0         0         0
 Ethernet32        U        0   0.00 B/s      0.00%         0         0         0      111  16.80 B/s      0.00%         0         0         0
 Ethernet40        U        0   0.00 B/s      0.00%         0         0         0      111  16.80 B/s      0.00%         0         0         0
 Ethernet48        U        0   0.00 B/s      0.00%         0         0         0      111  16.80 B/s      0.00%         0         0         0
 Ethernet56        U        0   0.00 B/s      0.00%         0         0         0      111  16.80 B/s      0.00%         0         0         0
 Ethernet64        U        0   0.00 B/s      0.00%         0         0         0      111  16.80 B/s      0.00%         0         0         0
 Ethernet72        U        0   0.00 B/s      0.00%         0         0         0      111  16.80 B/s      0.00%         0         0         0
 Ethernet80        U        0   0.00 B/s      0.00%         0         0         0      111  16.87 B/s      0.00%         0         0         0
 Ethernet88        U        0   0.00 B/s      0.00%         0         0         0      111  16.87 B/s      0.00%         0         0         0
 Ethernet96        U        0   0.00 B/s      0.00%         0         0         0      111  16.87 B/s      0.00%         0         0         0
Ethernet104        U        0   0.00 B/s      0.00%         0         0         0      111  16.94 B/s      0.00%         0         0         0
Ethernet112        U        0   0.00 B/s      0.00%         0         0         0      111  16.94 B/s      0.00%         0         0         0
Ethernet120        U        0   0.00 B/s      0.00%         0         0         0      111  16.94 B/s      0.00%         0         0         0
Ethernet128        U        0   0.00 B/s      0.00%         0         0         0      111  16.91 B/s      0.00%         0         0         0
...
...
```
> **Note:** Press Q to exit out of realtime monitoring counters

**Observation**

 - You can send `ping` traffic from `host00 to host01` and validate if counters increment. This is optional and depends on all nodes being up and running.

 ---

### Task 2.6: Clearing Counters

```bash
# Clear counters (non-disruptive)
sonic-clear counters
```
**Expected Output:**

```
Cleared counters
```
**Observation**
- Run `show interfaces counters` again and validate that the counters have been reset to zero.

---

## Exercise 3: Running Configuration (10 minutes)

In SONiC, the **running configuration** is the live state held in Redis (`CONFIG_DB`, DB #4) plus the live FRR state in `vtysh`. The **startup configuration** is the on-disk file `/etc/sonic/config_db.json` that gets reloaded after a reboot. This exercise inspects what's currently running.

### Task 3.1: SONiC Running Config

```bash
# Full SONiC running config (JSON view)
show runningconfiguration all
```

``` bash
# Specific sections
show runningconfiguration bgp
show runningconfiguration interfaces
show runningconfiguration acl
```

```
# Configuration mode
show runningconfiguration all | grep docker
``` 

**Expected Output**
```
"docker_routing_config_mode": "split",
```

**Observation**
- Lists the platform configuration in JSON format.
- There are **2 places** configurations are stored.
- Platform-specific configuration is stored in the **SONiC instance** (CONFIG_DB).
- Routing configuration is stored in the **routing instance — FRR**.

---

### Task 3.2: FRR / Routing Running Config
SONiC uses FRR for routing. The routing config lives in `vtysh`:
```bash
vtysh -c "show running-config"

# Or interactively
vtysh
show running-config
exit
```

**Observation**
- These show commands return the **running** FRR configuration (in memory). They are *not* persisted to disk until you run `write memory` (see Task 6.4).

---

### Task 3.3: Inspect BGP Sessions and Routes
SONiC uses FRR for routing. The routing config and runtime state live in `vtysh`. Use standard FRR show commands to review BGP peering and the IPv4 routing table:

```bash
vtysh 
show run 
show ip bgp summary
show ip bgp
```

**Expected output:**

```
show ip bgp 
BGP table version is 8, local router ID is 10.0.0.3, vrf id 0
Default local pref 100, local AS 65002
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

    Network          Next Hop            Metric LocPrf Weight Path
 *> 10.0.0.0/32      10.1.1.2                 0             0 65000 i
 *> 10.0.0.1/32      10.1.1.6                 0             0 65000 i
 *> 10.0.0.2/32      10.1.1.2                               0 65000 65001 i
 *=                  10.1.1.6                               0 65000 65001 i
 *> 10.0.0.3/32      0.0.0.0                  0         32768 i

Displayed  4 routes and 5 total paths

show ip bgp summary

IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.3, local AS number 65002 vrf-id 0
BGP table version 8
RIB entries 7, using 1568 bytes of memory
Peers 2, using 44 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
10.1.1.2        4      65000        52        53        0    0    0 00:44:51            2        4 N/A
10.1.1.6        4      65000       186       187        0    0    0 02:58:30            2        4 N/A

Total number of neighbors 2

```

**Observation**
- The FRR CLI is intentionally very similar to IOS-XE / Cisco-style routing CLI.
- Exit `vtysh` (type `exit`) to return to the SONiC CLI (CLICK-based `show` / `config` commands).

---

## Exercise 4: Startup Configuration via `jq` (10 minutes)

`jq` is a command-line JSON processor that comes pre-installed in SONiC because its entire configuration model is JSON. It's a standard open-source utility (like `grep` or `awk`, but for JSON instead of plain text).

SONiC stores configuration as structured JSON in two places:

- **On disk (startup config):** `/etc/sonic/config_db.json`
- **In memory (running config):** Redis `CONFIG_DB` (DB #4) — also represented as JSON when you query it

Use `jq` to read the on-disk startup config cleanly.

### Task 4.1: View the Whole File
```bash
jq . /etc/sonic/config_db.json | more
```

**Expected Output**
```
{
  "ASIC_SENSORS": {
    "ASIC_SENSORS_POLLER_INTERVAL": {
      "interval": "10"
    },
    "ASIC_SENSORS_POLLER_STATUS": {
      "admin_status": "enable"
    }
  },
  "AUTO_TECHSUPPORT": {
    "GLOBAL": {
      "available_mem_threshold": "10.0",
      "max_core_limit": "5.0",
      "max_techsupport_limit": "10.0",
      "min_available_mem": "200",
      "rate_limit_interval": "180",
      "since": "2 days ago",
      "state": "enabled"
    }
  },
  "AUTO_TECHSUPPORT_FEATURE": {
    "apm": {
      "available_mem_threshold": "10.0",
      "rate_limit_interval": "600",
      "state": "enabled"
    },
    "bgp": {
      "available_mem_threshold": "10.0",
      "rate_limit_interval": "600",
      "state": "enabled"
    },
    "database": {
      "available_mem_threshold": "10.0",
      "rate_limit_interval": "600",
      "state": "enabled"
    },
    ...
```

Press  Q to exit 

---

### Task 4.2: Pull Specific Sections
```bash
# Device metadata
jq '.DEVICE_METADATA' /etc/sonic/config_db.json
```

**Expected Output**
```
{
  "localhost": {
    "bgp_asn": "65001",
    "buffer_model": "traditional",
    "default_bgp_status": "up",
    "default_pfcwd_status": "enable",
    "docker_routing_config_mode": "split",
    "has_sonic_dhcpv4_relay": "True",
    "hostname": "leaf00",
    "hwsku": "Cisco-8101-32FH-O",
    "mac": "02:42:ac:14:06:64",
    "platform": "x86_64-8101_32fh_o-r0",
    "switch_type": "switch",
    "timezone": "UTC",
    "type": "not-provisioned"
  }
}
```

**Observation:**
The `DEVICE_METADATA.localhost` table is the **single source of truth for device-wide identity and global behavior**. Key fields:

- **`hostname: "leaf00"`** → The device name. Used in prompts, logs, BGP router-id derivation, and certificates. Change it with `sudo config hostname <name>` (then `config save -y`).
- **`bgp_asn: "65001"`** → The **global BGP Autonomous System number** for this device. FRR reads this at startup to build the `router bgp 65001` block. In a leaf/spine fabric, each leaf typically has a unique ASN (eBGP-to-spine design).
- **`hwsku: "Cisco-8101-32FH-O"`** → The Hardware SKU. Drives which `port_config.ini`, `sai.profile`, and platform plugins load. **Must match the physical hardware** — a wrong HwSKU = wrong port map = broken device.
- **`platform: "x86_64-8101_32fh_o-r0"`** → The platform string (matches `/usr/share/sonic/device/<platform>/`). Identifies the chassis family at the OS level. HwSKU lives *under* this platform.
- **`mac: "02:42:ac:14:06:64"`** → The device's **base MAC address**. SONiC derives interface MACs, BGP router-id (when not explicitly set), and LLDP chassis-id from this. Note the `02:42:ac:*` prefix → this is a **Docker-assigned MAC** (we're running in containerlab, not real hardware).
- **`switch_type: "switch"`** → Role hint. Values like `switch`, `spine`, `leaf`, `chassis-packet` influence default templates. `switch` is the generic default.
- **`docker_routing_config_mode: "split"`** → **Critical.** Tells SONiC that routing config is **owned by FRR** (`vtysh`), not generated from CONFIG_DB templates. This is why you have to look in *two* places for config — SONiC and FRR. Other modes: `unified` (FRR config generated from CONFIG_DB) or `separated` (legacy).
- **`default_bgp_status: "up"`** → New BGP neighbors come up `enabled` by default. If set to `down`, you'd have to manually unshut each new peer.
- **`default_pfcwd_status: "enable"`** → **PFC Watchdog** is on by default. PFC Watchdog detects and breaks PFC storms / deadlocks — important for lossless RoCE/AI fabrics. Leaving it enabled is best practice.
- **`buffer_model: "traditional"`** → Uses static (traditional) buffer profiles instead of dynamic threshold-based buffers. The other option is `dynamic`, which auto-balances buffer use across ports — preferred for modern AI/HPC workloads but requires platform support.
- **`has_sonic_dhcpv4_relay: "True"`** → This image includes the new DHCPv4 relay agent (vs the legacy isc-dhcp-relay). Just a feature-flag, not active config.
- **`timezone: "UTC"`** → System timezone. Affects log timestamps and `date` output. Keep as UTC in fabrics — easier correlation across regions.
- **`type: "not-provisioned"`** → Device role from the deployment system (ZTP / Sonic-Mgmt). Production devices typically show `LeafRouter`, `SpineRouter`, `ToRRouter`, etc. `not-provisioned` here confirms this is a lab device with no ZTP role applied.

```
# Just one port
jq '.PORT.Ethernet0' /etc/sonic/config_db.json
```
**Expected Output**

```
{
  "admin_status": "up",
  "alias": "etp0",
  "index": "0",
  "lanes": "2304,2305,2306,2307,2308,2309,2310,2311",
  "mtu": "9100",
  "speed": "400000"
}
```

**Observation:**

- The `PORT` key only holds **L1 / PHY level** information — it does not include IP addresses or FRR routing config (those live in `INTERFACE`, `LOOPBACK_INTERFACE`, and FRR).
- **`admin_status: "up"`** → The port is administratively enabled. Equivalent to `no shutdown` in IOS. Change with `sudo config interface startup Ethernet0` / `shutdown Ethernet0`.
- **`alias: "etp0"`** → The **operator-facing faceplate name** ("Ethernet **P**ort **0**"). What's printed on the chassis label and what NOC tools display. Always map `alias ↔ Ethernet<N>` when talking to customers — they live in `etpN`-land.
- **`index: "0"`** → The **physical front-panel position** (port 0 on the faceplate). Used by SNMP `ifIndex` derivation and by the platform layer to map this logical port to a specific QSFP cage. Two ports can share the same `index` only when broken out from the same cage.
- **`lanes: "2304,2305,...,2311"`** → The **eight ASIC SerDes lane IDs** bound to this port. 8 lanes × 50G PAM4 = **400G**. These numbers come straight from the platform's `port_config.ini` and are how SAI tells the Q200 silicon which physical lanes form this logical port. Changing the lane list = breakout/regrouping.
- **`mtu: "9100"`** → **Jumbo MTU**, the SONiC default. Sized for VXLAN/EVPN overhead on a 9000-byte payload. Production fabrics rarely deviate from 9100 unless edge-facing.
- **`speed: "400000"`** → Speed in **kbps** → 400,000 kbps = **400 Gbps**. SONiC always stores speed in kbps in JSON (CLI shows "400G"). For a 100G breakout port this would be `100000`.


```
# Interface IP assignments
jq '.INTERFACE, .LOOPBACK_INTERFACE' /etc/sonic/config_db.json
```
**Expected Output**

```
{
  "Ethernet0": {},
  "Ethernet0|10.1.1.1/31": {},
  "Ethernet0|2001:db8:1:1::1/127": {},
  "Ethernet8": {},
  "Ethernet8|10.1.1.5/31": {},
  "Ethernet8|2001:db8:1:1::5/127": {}
}
{
  "Loopback0": {},
  "Loopback0|10.0.0.2/32": {},
  "Loopback0|fc00:0:1002::1/128": {}
}
```

**Observation:**

This is how SONiC stores **L3 interface configuration** in CONFIG_DB. Notice that two separate tables are returned — `INTERFACE` (for physical / front-panel L3 ports) and `LOOPBACK_INTERFACE` (for logical loopbacks). They have **identical structure** but are kept apart so SAI/orchagent can treat them differently.

#### The two-key pattern (very important)

Each interface has **two kinds of keys** in the table:

| Key form | Meaning |
|---|---|
| `"Ethernet0": {}` | The **parent** entry — declares that this port is an L3 (routed) interface. Empty `{}` means no attributes (no VRF, no NAT zone, etc.). |
| `"Ethernet0\|10.1.1.1/31": {}` | A **child** entry — binds one IP address to that interface. The `\|` (pipe) is a key separator. Empty `{}` because the IP itself carries all the meaning. |


```
# List all top-level tables
jq 'keys' /etc/sonic/config_db.json
```

**Expected Output**
```
[
  "ASIC_SENSORS",
  "AUTO_TECHSUPPORT",
  "AUTO_TECHSUPPORT_FEATURE",
  "BANNER_MESSAGE",
  "BGP_DEVICE_GLOBAL",
  "CRM",
  "DEVICE_METADATA",
  "FEATURE",
  "FLEX_COUNTER_TABLE",
  "INTERFACE",
  "KDUMP",
  "LOGGER",
  "LOOPBACK_INTERFACE",
  "MGMT_INTERFACE",
  "MGMT_PORT",
  "NTP",
  "PASSW_HARDENING",
  "PORT",
  "SYSLOG_CONFIG",
  "SYSLOG_CONFIG_FEATURE",
  "SYSTEM_DEFAULTS",
  "VERSIONS"
]
```

**Observation:**


- This is the **schema view** of SONiC's configuration — every top-level key is a CONFIG_DB *table*. Think of it as the "list of objects this device knows about." Each table maps 1:1 to a Redis hash in CONFIG_DB (DB #4) and has a corresponding YANG model in `sonic-yang-models`. Understanding what each one does is the fastest way to navigate any SONiC device.
- Every table here has a corresponding YANG model. Knowing the table name lets you find its exact schema in `/usr/local/yang-models/` or the `sonic-yang-models` repo.

---

## Exercise 5: SONiC Architecture via Redis (10 minutes)


![alt text](image-1.png)

SONiC uses Redis as the **system-wide message bus and state store**: every container (orchagent, syncd, swss, pmon, FRR helpers, telemetry…) reads from and writes to Redis instead of talking to each other directly. This decouples the apps from the ASIC SDK and gives you a single, scriptable place to inspect *everything* the device knows.

SONiC's state is split into numbered databases:

| DB # | Name          | Purpose                                  |
|------|---------------|------------------------------------------|
| 0    | APPL_DB       | App-layer state (orchagent input)        |
| 1    | ASIC_DB       | What syncd programs into the ASIC        |
| 2    | COUNTERS_DB   | Live counters from the ASIC              |
| 4    | CONFIG_DB     | Running configuration                    |
| 6    | STATE_DB      | Operational state of components          |

### Task 5.1: List Keys in Config_DB (4)

CONFIG_DB is the JSON-shaped contract between you and the device. You write your intent into it; every SONiC subsystem reads from it to make the hardware behave accordingly.

```bash
# Config DB (running config)
redis-cli -n 4 KEYS '*' | head -20
```

**Expected output:**

```
INTERFACE|Ethernet0|10.1.1.1/31
FLEX_COUNTER_TABLE|QUEUE_WATERMARK
PORT|Ethernet248
LOGGER|SAI_API_ARS
PORT|Ethernet0
NTP|global
LOGGER|coppmgrd
LOGGER|stpd
SYSLOG_CONFIG_FEATURE|pmon
AUTO_TECHSUPPORT_FEATURE|sflow
LOGGER|SAI_API_ROUTE
AUTO_TECHSUPPORT_FEATURE|lldp
LOGGER|SAI_API_MCAST_FDB
AUTO_TECHSUPPORT_FEATURE|stp
LOGGER|vrfmgrd
LOGGER|tunnelmgrd
LOGGER|SAI_API_COUNTER
AUTO_TECHSUPPORT_FEATURE|eventd
SYSLOG_CONFIG_FEATURE|snmp
LOGGER|SAI_API_GENERIC_PROGRAMMABLE
```

**Observation**

- You can run `redis-dump -d 4 -y | jq` to dump the running instance of configuration from the Redis `CONFIG_DB`.

---

### Task 5.2: List Keys in Appl_DB(0)


#### Application DB (APPL_DB)
APPL_DB sits in the middle of SONiC's pipeline — between your configuration (CONFIG_DB) and the ASIC (ASIC_DB). It's where the software tells SAI what to actually program.

```bash
# Application DB
redis-cli -n 0 KEYS '*' | head -20
```

**Expected output:**

```
NEIGH_TABLE:Ethernet8:10.1.1.4
PORT_TABLE:Ethernet120
INTF_TABLE:Ethernet0:10.1.1.1/31
COPP_TABLE:queue4_group3
PORT_TABLE:Ethernet224
ROUTE_TABLE:fc00:0:1003::1
COPP_TABLE:default
ROUTE_TABLE:10.1.1.4/31
PORT_TABLE:Ethernet0
ROUTE_TABLE:10.0.0.2
SWITCH_TABLE:switch
PORT_TABLE:Ethernet40
ROUTE_TABLE:10.1.1.0/31
COPP_TABLE:queue4_group1
PORT_TABLE:Ethernet8
ROUTE_TABLE:2001:db8:1:1::4/127
INTF_TABLE:Ethernet8:10.1.1.5/31
INTF_TABLE:Loopback0:10.0.0.2/32
PORT_TABLE:Ethernet96
GEARBOX_TABLE_KEY_SET
```
**Observation**

- APPL_DB is what SONiC's apps decided to push toward the chip. It's the runtime, dynamic, multi-writer layer that turns intent into hardware programming.

---

### Task 5.3: List Keys in ASIC(1)

#### ASIC DB

ASIC_DB tells us what was actually programmed into the switch chip. It sits at the bottom of SONiC's pipeline — every route, port, ACL, neighbor, and nexthop that exists in hardware has a corresponding object here.

```bash
# ASIC DB
redis-cli -n 1 KEYS '*' | head -20
```
**Expected output:**

```
ASIC_STATE:SAI_OBJECT_TYPE_INGRESS_PRIORITY_GROUP:oid:0x1a000000000430
ASIC_STATE:SAI_OBJECT_TYPE_PORT:oid:0x100000000000a
ASIC_STATE:SAI_OBJECT_TYPE_QUEUE:oid:0x15000000000187
ASIC_STATE:SAI_OBJECT_TYPE_QUEUE:oid:0x15000000000146
ASIC_STATE:SAI_OBJECT_TYPE_POLICER:oid:0x120000000004c7
ASIC_STATE:SAI_OBJECT_TYPE_QUEUE:oid:0x1500000000016b
ASIC_STATE:SAI_OBJECT_TYPE_QUEUE:oid:0x1500000000006a
ASIC_STATE:SAI_OBJECT_TYPE_QUEUE:oid:0x1500000000003f
ASIC_STATE:SAI_OBJECT_TYPE_QUEUE:oid:0x1500000000002d
ASIC_STATE:SAI_OBJECT_TYPE_SCHEDULER_GROUP:oid:0x17000000000264
ASIC_STATE:SAI_OBJECT_TYPE_QUEUE:oid:0x15000000000162
ASIC_STATE:SAI_OBJECT_TYPE_INGRESS_PRIORITY_GROUP:oid:0x1a00000000041b
ASIC_STATE:SAI_OBJECT_TYPE_POLICER:oid:0x120000000004c5
ASIC_STATE:SAI_OBJECT_TYPE_QUEUE:oid:0x1500000000010a
ASIC_STATE:SAI_OBJECT_TYPE_SCHEDULER_GROUP:oid:0x170000000002f7
ASIC_STATE:SAI_OBJECT_TYPE_SCHEDULER_GROUP:oid:0x1700000000026f
ASIC_STATE:SAI_OBJECT_TYPE_INGRESS_PRIORITY_GROUP:oid:0x1a0000000003fb
ASIC_STATE:SAI_OBJECT_TYPE_QUEUE:oid:0x150000000000af
ASIC_STATE:SAI_OBJECT_TYPE_HOSTIF:oid:0xd0000000004af
ASIC_STATE:SAI_OBJECT_TYPE_QUEUE:oid:0x1500000000021c
```

**Observation**
- Every SAI object you see in Redis corresponds to a real entry in the ASIC's hardware tables (FIB, TCAM, MAC table, buffer config). Diffing APPL_DB vs ASIC_DB shows you exactly what the chip accepted vs rejected.

----

### Task 5.4: List Keys in State_DB(6)

#### State DB

STATE_DB holds the live operational state of every component on the device. Where CONFIG_DB stores intent and ASIC_DB stores programmed hardware objects, STATE_DB stores **observed reality**.

Every SONiC component (orchagent, pmon, swss, FRR helpers, dhcp_relay, lldp, syncd, telemetry) publishes its observed state to STATE_DB. Other components and the CLI subscribe to it.

```bash
# State DB
redis-cli -n 6 KEYS '*' | head -20
```

**Expected output:**

```
PHYSICAL_ENTITY_INFO|NPU1_TEMP_9
TEMPERATURE_INFO|MB_PORT_Sensor
ALL_SERVICE_STATUS|determine-reboot-cause
PROCESS_STATS|107
TEMPERATURE_INFO|NPU1_TEMP_7
CURRENT_INFO|TI_3.3V_R_IINSEN_BB
PROCESS_STATS|213
PROCESS_STATS|9629
MEMORY_STATS|Shared
PROCESS_STATS|14585
TRANSCEIVER_FIRMWARE_INFO|Ethernet104
PROCESS_STATS|53975
FIPS_STATS|state
PROCESS_STATS|101
TRANSCEIVER_FIRMWARE_INFO|Ethernet96
PROCESS_STATS|54080
TEMPERATURE_INFO|MB_JM11_L2_TEMP
PORT_TABLE|Ethernet184
PROCESS_STATS|47
TRANSCEIVER_DOM_THRESHOLD|Ethernet88
```

**Observation**

- STATE_DB is the device's "vital signs" dashboard — what every component currently observes to be true on this box.

---

### Task 5.5: List Keys in Counters_DB(2)

#### Counters DB

COUNTERS_DB holds the live numeric counters polled directly from the ASIC and from other subsystems. Every packet, byte, drop, error, queue depth, buffer occupancy, PFC pause, and watermark value the chip exposes ends up here.

This is where `show interfaces counters`, `show queue counters`, `show pfc counters`, `show priority-group watermark`, etc. get their numbers.

```bash
# Counters DB
redis-cli -n 2 KEYS 'COUNTERS:*' | head -20
```

**Expected output:**

```
COUNTERS:oid:0x15000000000184
COUNTERS:oid:0x1a0000000003f6
COUNTERS:oid:0x150000000001f8
COUNTERS:oid:0x1a0000000003bf
COUNTERS:oid:0x1a0000000003d2
COUNTERS:oid:0x15000000000134
COUNTERS:oid:0x1a00000000037a
COUNTERS:oid:0x150000000001de
COUNTERS:oid:0x1a00000000043a
COUNTERS:oid:0x15000000000070
COUNTERS:oid:0x1500000000009d
COUNTERS:oid:0x1500000000012a
COUNTERS:oid:0x150000000000c8
COUNTERS:oid:0x150000000000ec
COUNTERS:oid:0x1500000000008b
COUNTERS:oid:0x150000000000e1
COUNTERS:oid:0x100000000000d
COUNTERS:oid:0x1a0000000003bd
COUNTERS:oid:0x1500000000010b
COUNTERS:oid:0x150000000001c0
```

**Observation**
- COUNTERS_DB is the live stats firehose from the ASIC — every byte, drop, pause, and queue depth the chip can report.

---

### Task 5.6: Inspect Specific Entries

The purpose of this section is to gather specific stats from Redis tables. There are cases where a `show` CLI command may not exist for what you need, or you may want to script a custom telemetry pull — in those cases you can read directly from Redis.

#### Gather the configuration intent for Ethernet0

```bash
# Port config in CONFIG_DB
redis-cli -n 4 HGETALL "PORT|Ethernet0"
```


**Expected output:**

```
 1) "admin_status"
 2) "up"
 3) "alias"
 4) "etp0"
 5) "index"
 6) "0"
 7) "lanes"
 8) "2304,2305,2306,2307,2308,2309,2310,2311"
 9) "mtu"
10) "9100"
11) "speed"
12) "400000"
```

**Observation**
- Port has been configured for 400G with 8 SerDes lanes.
- Port is admin-up with MTU 9100.

#### Gather the port information for Ethernet0


```
# Port state in APPL_DB
redis-cli -n 0 HGETALL "PORT_TABLE:Ethernet0"
```

**Expected output:**

```
 1) "admin_status"
 2) "up"
 3) "alias"
 4) "etp0"
 5) "index"
 6) "0"
 7) "lanes"
 8) "2304,2305,2306,2307,2308,2309,2310,2311"
 9) "mtu"
10) "9100"
11) "speed"
12) "400000"
13) "description"
14) ""
15) "oper_status"
16) "up"
17) "flap_count"
18) "1"
19) "last_up_time"
20) "Mon May 25 18:14:30 2026"
```

**Observation**
- Information from `CONFIG_DB` has been successfully pushed into `APPL_DB`.
- APPL_DB adds **runtime fields** that CONFIG_DB does not have: `oper_status`, `flap_count`, `last_up_time`, `description`. These come from orchagent / portsyncd as the port comes up.

#### Gather the port stats for Ethernet0


```
# Port oper state in STATE_DB
redis-cli -n 6 HGETALL "PORT_TABLE|Ethernet0"
```

**Expected output:**

```
 1) "state"
 2) "ok"
 3) "netdev_oper_status"
 4) "up"
 5) "admin_status"
 6) "up"
 7) "mtu"
 8) "9100"
 9) "supported_speeds"
10) "200000,400000"
11) "supported_fecs"
12) "none,rs"
13) "host_tx_ready"
14) "true"
15) "speed"
16) "400000"
17) "fec"
18) "rs"
```
**Observation**
- Current port state is reflected in `STATE_DB`.
- `netdev_oper_status` → the Linux kernel says the `Ethernet0` netdev is **RUNNING**.
- `supported_speeds` / `supported_fecs` → capabilities reported by the platform/transceiver (useful for breakout and FEC planning).
- `host_tx_ready: true` → the host-side MAC has TX enabled (negotiated up).
- `fec: rs` → RS-FEC is active on the link.

#### Identify the OID for the port

```
# Counters for a port (need OID lookup)
redis-cli -n 2 HGET COUNTERS_PORT_NAME_MAP Ethernet0
```

**Expected output:**

```
"oid:0x1000000000002"
```

**Observation**
- This command is the **name-to-OID translation step** required to read any per-port counters from COUNTERS_DB.



#### Poll the OID to gather stats around the Port Ethernet0

```
# Use the returned OID:
redis-cli -n 2 HGETALL "COUNTERS:oid:0x1000000000002"
```

**Expected output:**

```
  1) "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S14"
  2) "0"
  3) "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S11"
  4) "0"
  5) "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S10"
  6) "0"
  7) "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S9"
  8) "0"
  9) "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S8"
 10) "0"
 11) "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S7"
 12) "0"
 13) "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S5"
 14) "0"
 ...
 ...
```
**Observation**
- This output captures all the stats around the Ethernet0 interface as seen by the hardware.
- This is the **authoritative per-port statistics hash** that every `show` / telemetry tool reads from — packets, octets, RMON histogram, FEC symbol bins, PFC, and discards all live in this single Redis key.

---

## Exercise 6: Saving Configuration (5 minutes)

By default, SONiC does **not** persist runtime changes — a reboot will reload `config_db.json`. Save explicitly:

### Task 6.1: Change the hostname of the device


```bash
# Change the hostname of the device
sudo config hostname test-leaf01
exit
ssh admin@leaf01
```

**Expected output:**

```

logout
Connection to leaf01 closed.

Warning: Permanently added 'leaf01' (RSA) to the list of known hosts.
Debian GNU/Linux 12 \n \l

admin@leaf01's password: 
Linux test-leaf01 6.1.0-11-2-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.38-4 (2023-08-08) x86_64
You are on
  ____   ___  _   _ _  ____
 / ___| / _ \| \ | (_)/ ___|
 \___ \| | | |  \| | | |
  ___) | |_| | |\  | | |___
 |____/ \___/|_| \_|_|\____|

-- Software for Open Networking in the Cloud --

Unauthorized access and/or use are prohibited.
All access and/or use are subject to monitoring.

Help:    https://sonic-net.github.io/SONiC/

Last login: Mon May 25 21:19:06 2026 from 172.20.2.1
admin@test-leaf01:~$ 
```

**Observation**

- `config hostname` writes the new value to **CONFIG_DB only** (runtime). The on-disk `config_db.json` is unchanged until you run `config save -y` (Task 6.2).

```bash
# Spot-check a recent change from CONFIG_DB (DB #4) — runtime Redis database
redis-cli -n 4 HGET 'DEVICE_METADATA|localhost' hostname
```
**Expected output:**

```
"test-leaf01"
```

```bash
# Spot-check the same field in the on-disk startup config
jq '.DEVICE_METADATA.localhost.hostname' /etc/sonic/config_db.json
```

**Expected output:**

```
"leaf01"
```
**Observation**

- Runtime changes are **not** reflected in the saved DB yet.
- On reboot or reload, SONiC reads from `config_db.json` (startup) — any runtime-only changes would be **lost**.

---

### Task 6.2: Save the Running Config


```bash
# Save running config to /etc/sonic/config_db.json
sudo config save -y
```
**Expected output:**

```
Running command: /usr/local/bin/sonic-cfggen -d --print-data > /etc/sonic/config_db.json
```

**Observation**

-- Runtime changes are saved into the config_db.json

---

### Task 6.3: Verify the Save
```bash
# Confirm file timestamp updated
uptime
ls -l /etc/sonic/config_db.json

# Spot-check a recent change
jq '.DEVICE_METADATA.localhost.hostname' /etc/sonic/config_db.json
```

**Expected output:**

```
"test-leaf01"
```

**Observation**
- Runtime changes are now persisted to `/etc/sonic/config_db.json` — startup and runtime are in sync. After a reboot, the device will come up with the saved hostname.

---

### Task 6.4: FRR / Routing Save
FRR has its own persistence — save it separately:
```bash
vtysh -c "write memory"
```
**Expected output:**

```
Note: this version of vtysh never writes vtysh.conf
Building Configuration...
Configuration saved to /etc/frr/zebra.conf
Configuration saved to /etc/frr/bgpd.conf
Configuration saved to /etc/frr/staticd.conf
Configuration saved to /etc/frr/bfdd.conf
```

**Observation**

- FRR persists its config to `/etc/frr/*.conf` (`zebra.conf`, `bgpd.conf`, `staticd.conf`, `bfdd.conf`) — **separate from** `/etc/sonic/config_db.json`. Without `write memory`, any BGP / route-map / prefix-list change you made in `vtysh` would be lost on a reboot.

**⚠️ Important:** `config save` saves SONiC tables. `write memory` (in `vtysh`) saves FRR routing config. Do **both** after routing changes.

---

## Lab Wrap-Up

You should now be able to:

- [ ] Identify SONiC version, platform, and boot image
- [ ] Read interface status, IPs, and counters
- [ ] Inspect both SONiC and FRR running configurations
- [ ] Parse `config_db.json` with `jq`
- [ ] Navigate the Redis databases that drive SONiC
- [ ] Save SONiC and FRR configuration the right way

---

### Cheat Sheet
```bash
show version                                       # SONiC + platform
show platform summary                              # HwSKU / ASIC
show interfaces status                             # Link state
show ip interfaces                                 # L3 view
show interfaces counters                           # Port stats
sonic-clear counters                               # Reset counters
show runningconfiguration all                      # SONiC config (JSON)
vtysh -c "show running-config"                     # FRR config
vtysh -c "show ip bgp summary"                     # BGP peers
jq . /etc/sonic/config_db.json                     # Startup config
redis-cli -n 4 KEYS '*'                            # CONFIG_DB keys
redis-cli -n 4 HGET 'DEVICE_METADATA|localhost' hostname   # Field lookup
sudo config save -y                                # Persist SONiC
vtysh -c "write memory"                            # Persist FRR
```

---

**End of Lab**
