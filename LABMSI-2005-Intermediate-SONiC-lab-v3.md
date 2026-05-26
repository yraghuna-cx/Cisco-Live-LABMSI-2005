# LABMSI-2005 - Try SONiC { Intermediate } - Open Network Operating System powering the Next-gen AI-Data Center

**Objective:** Go beyond the basics of SONiC — trace a BGP prefix end-to-end through every SONiC layer (FRR → RIB → kernel FIB → Redis → ASIC), configure and validate control-plane and data-plane ACLs, stream telemetry via gNMI, and learn the right way to gather logs and artifacts for TAC engagements.

**Duration:** 45–60 minutes

**Prerequisites:**
- Completion of **LABMSI-1011** (Introductory SONiC) or equivalent SONiC familiarity
- A running SONiC device (lab leaf, e.g. `leaf00`) reachable over SSH
- Admin credentials (`admin` user)
- Basic familiarity with `vtysh`, `redis-cli`, and Linux CLI

---

## Table of Contents

- [Lab Overview](#lab-overview)
- [Lab Topology](#lab-topology)
- [Lab Setup Verification](#lab-setup-verification)
- [Exercise 1: Trace a BGP Route End-to-End (15 min)](#exercise-1-trace-a-bgp-route-end-to-end-15-min)
- [Exercise 2: Control-Plane and Data-Plane ACLs (18 min)](#exercise-2-control-plane-and-data-plane-acls-18-min)
- [Exercise 3: Stream Telemetry with gNMI (10 min)](#exercise-3-stream-telemetry-with-gnmi-10-min)
- [Exercise 4: Troubleshoot Like a Pro (7 min)](#exercise-4-troubleshoot-like-a-pro-7-min)
- [Lab Wrap-Up & Cheat Sheet](#lab-wrap-up)

---

## Lab Overview

For participants who have completed the introductory SONiC lab (or have basic SONiC familiarity), this hands-on session takes you deeper into how SONiC works internally. You will:

- **Trace a BGP prefix end-to-end** — FRR → RIB → kernel FIB → Redis APPL_DB → ASIC_DB → NPU.
- **Configure & validate ACLs** — apply both control-plane (iptables) and data-plane (TCAM) ACLs.
- **Stream telemetry with gNMI** — collect operational data via OpenConfig and `sonic-db` origins.
- **Troubleshoot like a pro** — gather the right logs and artifacts for effective TAC engagement.

> **Note:** The lab device is a **containerlab** node running real SONiC on a virtual **Cisco-8101-32FH-O**. All CLI, Redis, and config behaviors match physical hardware.

---

## Lab Topology

![alt text](image-2.png)

> **Note:** Some nodes may be unavailable due to virtual-host resource limits. You need access to at least one leaf (`leaf00` or `leaf01`) to complete this lab.

---

## Lab Setup Verification

### Step 1: SSH to the Virtual Host

```bash
ssh cisco@198.18.200.11
# password: cisco123
cd /home/cisco/cisco8000e/sonic-clab/c8101/ansible
```

### Step 2: Confirm Reachability

```bash
ansible all -i hosts -m ping
```

**Expected:** `leaf00`, `leaf01`, `spine00`, `spine01` (or a subset) all report `SUCCESS / pong`.

### Step 3: SSH to a Reachable Node

```bash
ssh admin@leaf00
# Default password: password
```

Expected: a prompt like `admin@leaf00:~$`.

---

## Exercise 1: Trace a BGP Route End-to-End (15 min)

### Objective
Trace an IPv4 BGP-learned prefix across every SONiC software layer — from FRR's BGP table down through the kernel FIB, the Redis databases (APPL_DB → ASIC_DB), and finally the NPU hardware forwarding table.

### The SONiC Route Pipeline

```
   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐   ┌──────────┐   ┌──────────┐
   │   BGP    │──▶│   RIB    │──▶│  Kernel  │──▶│ APPL_DB │──▶│ ASIC_DB  │──▶│   NPU    │
   │  (bgpd)  │   │ (zebra)  │   │   FIB    │   │ (db 0)  │   │  (db 1)  │   │ (SAI HW) │
   └──────────┘   └──────────┘   └──────────┘   └─────────┘   └──────────┘   └──────────┘
       FRR           FRR           Linux         fpmsyncd       orchagent      syncd/SDK
```

| DB # | Name | Purpose |
|------|------|---------|
| 0 | `APPL_DB` | App-level intent (what apps *want* in the dataplane) |
| 1 | `ASIC_DB` | SAI objects programmed in the ASIC |
| 2 | `COUNTERS_DB` | Counters |
| 4 | `CONFIG_DB` | User configuration |
| 6 | `STATE_DB` | Runtime state |

We'll use prefix **`10.0.0.3/32`** (loopback advertised by a remote leaf, ECMP via the spines).

---

### Task 1.1: Confirm the Prefix in BGP, RIB, and Kernel FIB

```bash
ssh admin@leaf00
show ip bgp 10.0.0.3
show ip route 10.0.0.3
ip route show 10.0.0.3
```

**Expected (summarized):**

- `show ip bgp 10.0.0.3` → 2 paths via `10.1.1.0` and `10.1.1.4` (ECMP), both `valid, external, multipath`; one is `best`.
- `show ip route 10.0.0.3` → `Known via "bgp"`, two next-hops out `Ethernet0` and `Ethernet8`.
- `ip route show 10.0.0.3` → `proto bgp`, two `nexthop via … dev Ethernet0/8 weight 1`.

**Observation**

- BGP best-path → zebra RIB → Linux kernel FIB (via Netlink). From the kernel, `fpmsyncd` mirrors the route into Redis APPL_DB.

---

### Task 1.2: Trace into APPL_DB (Redis DB 0)

```bash
sudo redis-cli -n 0 HGETALL 'ROUTE_TABLE:10.0.0.3'
```

**Expected output:**

```text
1) "protocol"   2) "bgp"
3) "nexthop"    4) "10.1.1.0,10.1.1.4"
5) "ifname"     6) "Ethernet0,Ethernet8"
```

**Observation**

- The route is now expressed as application intent in `APPL_DB`. The `protocol=bgp` tag is `fpmsyncd` recording its origin. `orchagent` will pick this up next.

---

### Task 1.3: Trace into ASIC_DB (Redis DB 1)

#### 1.3.1: Find the SAI Route Entry

```bash
sudo redis-cli -n 1 KEYS '*' | grep '10.0.0.3'
sudo redis-cli -n 1 HGETALL 'ASIC_STATE:SAI_OBJECT_TYPE_ROUTE_ENTRY:{"dest":"10.0.0.3/32","switch_id":"oid:0x21000000000000","vr":"oid:0x3000000000022"}'
```

**Expected output:**

```text
ASIC_STATE:SAI_OBJECT_TYPE_ROUTE_ENTRY:{"dest":"10.0.0.3/32",...}

1) "SAI_ROUTE_ENTRY_ATTR_NEXT_HOP_ID"
2) "oid:0x50000000004e9"
```

**Observation**

- `orchagent` translated APPL_DB intent into a SAI route object. The single attribute `NEXT_HOP_ID` points to a forwarding object (here, an ECMP group).

#### 1.3.2: Walk the Next-Hop Group and Its Members

The `NEXT_HOP_ID` may point to either a single `NEXT_HOP` or a `NEXT_HOP_GROUP`. For ECMP it's a group, and members back-reference the group ID.

```bash
GRP=oid:0x50000000004e9

# Confirm it's an ECMP group
sudo redis-cli -n 1 HGETALL "ASIC_STATE:SAI_OBJECT_TYPE_NEXT_HOP_GROUP:$GRP"

# Find all members that point back to this group (one-shot scan)
for k in $(sudo redis-cli -n 1 KEYS 'ASIC_STATE:SAI_OBJECT_TYPE_NEXT_HOP_GROUP_MEMBER:*'); do
  echo "--- $k ---"
  sudo redis-cli -n 1 HGETALL "$k"
done | grep -B1 -A3 "$GRP"
```

**Expected (summarized):**

- Group attribute: `SAI_NEXT_HOP_GROUP_ATTR_TYPE = SAI_NEXT_HOP_GROUP_TYPE_DYNAMIC_UNORDERED_ECMP`.
- The loop prints all `NEXT_HOP_GROUP_MEMBER` objects; **two** of them reference `oid:0x50000000004e9` and each carries a `NEXT_HOP_ID` (e.g. `oid:0x40000000004d9`, `oid:0x40000000004db`). Other members belong to different groups and are ignored.

**Observation**

- SAI has no "list members of group" call — you scan members and filter by the back-reference `NEXT_HOP_GROUP_ID`. The two matching members are the ECMP paths.

#### 1.3.3: Resolve One Member to a Port

```bash
NH=oid:0x40000000004d9

# Next-hop IP and the router interface it uses
sudo redis-cli -n 1 HGETALL "ASIC_STATE:SAI_OBJECT_TYPE_NEXT_HOP:$NH"

# Router interface → physical port OID
RIF=$(sudo redis-cli -n 1 HGETALL "ASIC_STATE:SAI_OBJECT_TYPE_NEXT_HOP:$NH" | grep -A1 ROUTER_INTERFACE_ID | tail -1)
sudo redis-cli -n 1 HGETALL "ASIC_STATE:SAI_OBJECT_TYPE_ROUTER_INTERFACE:$RIF"

# Port OID → interface name (from COUNTERS_DB port-name map)
sudo redis-cli -n 2 HGETALL COUNTERS_PORT_NAME_MAP | grep -B1 "oid:0x1000000000002"
```

**Expected (summarized):**

- `NEXT_HOP` → IP `10.1.1.0`, RIF `oid:0x60000000004d7`.
- `ROUTER_INTERFACE` → type `PORT`, port `oid:0x1000000000002`, src-mac, MTU 9100.
- Port-name map → `Ethernet0`.

**Full chain for `10.0.0.3/32` ECMP member #1:**

```
ROUTE_ENTRY 10.0.0.3/32
 → NEXT_HOP_GROUP  oid:0x50000000004e9  (DYNAMIC_UNORDERED_ECMP)
   → MEMBER        oid:0x2d0000000004ea
     → NEXT_HOP    oid:0x40000000004d9  (IP 10.1.1.0)
       → RIF       oid:0x60000000004d7
         → PORT    oid:0x1000000000002  →  Ethernet0
```

---

### Task 1.4: Confirm Hardware Programming in the NPU

`syncd` translates `ASIC_DB` into vendor SDK calls. The platform CLI exposes the resulting NPU table:

```bash
sudo show platform npu router route-table | grep -E "router|10.0.0.3"
```

**Expected output:**

```text
| router-id | ip-prefix    | dest-type       | dest-gid | dest-oid | dest-info        | drop  | ip-ver |
| 0x0       | 10.0.0.3/32  | multipath-group | 0x0      | 0x0      | ['LEVEL_2', '2'] | False | 4      |
```

**Observation**

- `drop = False` → packets forwarded; `ip-ver = 4` → IPv4; `dest-type = multipath-group` → ECMP confirmed in silicon.

### Validation Checklist

- [ ] Prefix is **best** in BGP, present in zebra RIB and kernel FIB
- [ ] `ROUTE_TABLE:<prefix>` exists in APPL_DB with correct next-hops
- [ ] `SAI_OBJECT_TYPE_ROUTE_ENTRY` resolves to a valid Next-Hop / Next-Hop-Group OID
- [ ] `show platform npu router route-table` shows the prefix with `drop=False`

---

## Exercise 2: Control-Plane and Data-Plane ACLs (18 min)

### Concept — Two Enforcement Paths, One Model

SONiC enforces packet filtering in **two places** driven by the **same JSON / CONFIG_DB model**. The **ACL table `type`** decides where:

| Property | **Data-plane ACL** (`L3`, `L3V6`, `MIRROR`) | **Control-plane ACL** (`CTRLPLANE`) |
|---|---|---|
| **Programmed in** | ASIC TCAM | Linux iptables / ip6tables |
| **Filters** | Transit traffic on ports/LAGs/VLANs | Traffic destined **to** the CPU |
| **Performance** | Line rate, zero CPU | Software, CPU bound |
| **Counters** | `aclshow`, COUNTERS_DB | `iptables -L -v -n` |
| **Driver** | `orchagent` → SAI | `caclmgrd` → iptables |

```
   acl.json ──▶ CONFIG_DB ──┬─▶ orchagent ─▶ ASIC TCAM   (L3 / L3V6 / MIRROR)
                            └─▶ caclmgrd  ─▶ iptables    (CTRLPLANE)
```

---

### Task 2.1: Baseline iptables + Configuration Snapshot

`caclmgrd` writes baseline iptables rules at boot for control-plane protection (BGP/LLDP/DHCP/SSH/etc.).

```bash
sudo iptables -L INPUT -v -n --line-numbers
sudo iptables-save > /tmp/iptables.before.txt
sudo config checkpoint baseline-configs
sudo config list-checkpoints
```

**Expected (summarized):**

- INPUT chain accepts loopback, ESTABLISHED/RELATED, ICMP types 0/3/8/11, DHCP, BGP (`tcp dpt:179`), TTL<2 traceroute returns, and ends with explicit DROPs for the device's own loopback/transit IPs.
- `caclmgrd` is the **only** entity that should edit these rules — manual edits get overwritten on its next reconcile.
- `config checkpoint baseline-configs` writes `/etc/sonic/checkpoints/baseline-configs.cp.json`.

> [!TIP]
> Keep `/tmp/iptables.before.txt` — you'll diff against it after applying the CTRLPLANE ACL.

---

### Task 2.2: Control-Plane ACL — Restrict SSH

Narrow SSH to the management subnet `172.20.2.0/24`. `caclmgrd` will render this as iptables rules.

#### 2.2.1: Create the JSON

```bash
sudo vi /tmp/acl_ctrlplane_ssh.json
```

```json
{
  "ACL_TABLE": {
    "SSH_ONLY": {
      "type": "CTRLPLANE",
      "policy_desc": "Restrict SSH to mgmt subnet",
      "services": ["SSH"],
      "stage": "ingress"
    }
  },
  "ACL_RULE": {
    "SSH_ONLY|RULE_10": {
      "PRIORITY": "9999",
      "SRC_IP": "172.20.2.0/24",
      "IP_PROTOCOL": "6",
      "L4_DST_PORT": "22",
      "PACKET_ACTION": "ACCEPT"
    },
    "SSH_ONLY|DEFAULT_RULE": {
      "PRIORITY": "1",
      "ETHER_TYPE": "0x0800",
      "PACKET_ACTION": "DROP"
    }
  }
}
```

#### 2.2.2: Apply and Verify

```bash
sudo config load /tmp/acl_ctrlplane_ssh.json -y
show acl table SSH_ONLY
show acl rule  SSH_ONLY
```

**Expected (summarized):** table `SSH_ONLY` with type `CTRLPLANE`, binding `SSH`, status `Active`; two rules (`RULE_10` ACCEPT, `DEFAULT_RULE` DROP).

#### 2.2.3: Confirm It Landed in iptables (Not TCAM)

```bash
sudo iptables-save > /tmp/iptables.after.txt
diff -u /tmp/iptables.before.txt /tmp/iptables.after.txt | sed -n '1,40p'

# And confirm aclshow has no data-plane counters for this table
aclshow -a | grep SSH_ONLY
```

**Expected diff (new lines):**

```diff
+-A INPUT -s 172.20.2.0/24 -p tcp -m tcp --dport 22 -j ACCEPT
+-A INPUT -p tcp -m tcp --dport 22 -j DROP
+-A INPUT -j DROP
```

**Expected `aclshow`:** `PACKETS COUNT` and `BYTES COUNT` show `N/A` — confirming this rule is **not** in TCAM.

| Added iptables line | Maps to JSON rule | Effect |
|---|---|---|
| `ACCEPT … 172.20.2.0/24 … dport 22` | `SSH_ONLY\|RULE_10` (PRIO 9999) | Mgmt subnet can SSH |
| `DROP … dport 22` | Service-scoped deny by `caclmgrd` | SSH from elsewhere → timeout |
| `INPUT -j DROP` (end of chain) | `SSH_ONLY\|DEFAULT_RULE` (PRIO 1) | Chain-wide default-deny |

> [!WARNING]
> The chain-wide `-A INPUT -j DROP` affects **all** control-plane traffic. If you depend on NTP/SNMP/BFD/etc., add their explicit ACCEPTs before applying this in production.

#### 2.2.4: Functional Test

```bash
# From the lab host (172.20.2.x) — should succeed
ssh admin@leaf00
```

---

### Task 2.3: Data-Plane ACL — Block ICMP at Ethernet0

This rule **must** land in TCAM, **not** iptables.

#### 2.3.1: Confirm Baseline Connectivity

From `host00` (192.168.1.1) → `host11` (192.168.2.1):

```bash
ping -c 3 192.168.2.1
```

**Expected:** 3 replies. (If this fails, skip Task 2.3 — topology issue.)

#### 2.3.2: Apply the Data-Plane ACL

```bash
sudo vi /tmp/acl_dataplane_icmp.json
```

```json
{
  "ACL_TABLE": {
    "DP_BLOCK_ICMP": {
      "type": "L3",
      "policy_desc": "Drop ICMP echo at ingress",
      "ports": ["Ethernet0"],
      "stage": "ingress"
    }
  },
  "ACL_RULE": {
    "DP_BLOCK_ICMP|RULE_10": {
      "PRIORITY": "20",
      "IP_PROTOCOL": "1",
      "ICMP_TYPE": "8",
      "PACKET_ACTION": "DROP"
    },
    "DP_BLOCK_ICMP|RULE_20": {
      "PRIORITY": "10",
      "PACKET_ACTION": "FORWARD"
    }
  }
}
```

```bash
sudo config load -y /tmp/acl_dataplane_icmp.json
show acl table DP_BLOCK_ICMP
show acl rule  DP_BLOCK_ICMP
```

**Expected:** Table `DP_BLOCK_ICMP` type `L3`, binding `Ethernet0`, status `Active`.

#### 2.3.3: Re-Test and Verify TCAM Programming

```bash
# From host00 — should now fail/timeout
ssh root@host00 ping -c 5 192.168.2.1

# Back on leaf00 — counters increment in aclshow
aclshow -a -t DP_BLOCK_ICMP

# And confirm it is NOT in iptables
sudo iptables -L -v -n | grep DP_BLOCK_ICMP || echo "Not in iptables — correct."
```

**Expected `aclshow`:** `RULE_10` shows non-zero `PACKETS COUNT` / `BYTES COUNT`. `iptables` grep returns nothing → confirms TCAM enforcement.

---

### Task 2.4: Cleanup (Informational)

```bash
sudo config rollback baseline-configs
```

> **Note:** On this simulated platform, `config rollback` fails at the patch-apply stage. The mechanism (`config checkpoint` / `list-checkpoints` / `rollback`) is the production-safe way to revert; on real hardware it restores `CONFIG_DB` to the snapshot. For this lab, leaving the ACLs in place is fine.

---

## Exercise 3: Stream Telemetry with gNMI (10 min)

`gnmic` works with both **OpenConfig** models and SONiC's **`sonic-db:` origin** (direct Redis DB access).

> **Note:** gNMI **Set** support on this image is limited — this exercise focuses on **Get** for operational telemetry.

### Task 3.1: Validate `gnmic` on the Host and gNMI on the Switch

```bash
# On the lab host
gnmic version

# On the switch
ssh admin@leaf00 'show feature status | grep gnmi'
```

**Expected:** `gnmic` ≥ 0.45.x; `gnmi   enabled   enabled`.

---

### Task 3.2: Discover Supported Models

```bash
gnmic -a 172.20.2.100:8080 --insecure capabilities
```

**Expected (summarized):** gNMI version `0.7.0`; models include `openconfig-interfaces`, `openconfig-system`, `openconfig-platform`, `openconfig-lldp`, `openconfig-acl`, and `sonic-db`. Encodings: `JSON`, `JSON_IETF`, `PROTO`.

> **Note:** Available models live on the switch under `/usr/models/yang/`.

---

### Task 3.3: Get Software Version (OpenConfig)

```bash
gnmic -a 172.20.2.100:8080 --insecure -u admin -p password get \
  --path "/openconfig-system:system/state/software-version"
```

**Expected (summarized):** returns a single value such as `"202405c.38617-int-20260325.164610"`. This is the gNMI equivalent of `show version`.

---

### Task 3.4: Interface Telemetry (OpenConfig)

```bash
gnmic -a 172.20.2.100:8080 --insecure -u admin -p password get \
  --path "/openconfig-interfaces:interfaces/interface[name=Ethernet0]"
```

**Expected (summarized):** rich JSON for `Ethernet0` including `admin-status`, `oper-status`, MTU, speed (`SPEED_400GB`), full RX/TX counter set, IPv4 (`10.1.1.1/31`) and IPv6 sub-interface addresses.

**Observation**

- SONiC exposes interface state and counters through standard OpenConfig — same paths you'd use on any OC-speaking platform.

---

### Task 3.5: Query Redis Directly via the `sonic-db` Origin

#### 3.5.1: APPL_DB port table

```bash
gnmic -a 172.20.2.100:8080 --insecure -u admin -p password get \
  --target APPL_DB --path "/PORT_TABLE/Ethernet0"
```

**Expected (summarized):** the same key/values you'd see from `redis-cli -n 0 HGETALL PORT_TABLE:Ethernet0` — `admin_status`, `oper_status`, `mtu`, `speed`, `lanes`, `flap_count`, `last_up_time`.

#### 3.5.2: CONFIG_DB device metadata

```bash
gnmic -a 172.20.2.100:8080 --insecure -u admin -p password get \
  --target CONFIG_DB --path "/DEVICE_METADATA/localhost"
```

**Expected (summarized):** `bgp_asn`, `hostname`, `hwsku`, `platform`, `mac`, `timezone` — the same fields the box uses at boot.

**Observation**

- Any Redis DB (`APPL_DB`, `CONFIG_DB`, `STATE_DB`, `COUNTERS_DB`, `ASIC_DB`) is reachable as a gNMI `--target`, so any monitoring you can build with `redis-cli` can be streamed off-box via gNMI.

---

## Exercise 4: Troubleshoot Like a Pro (7 min)

A quick tour of **where to look** when something breaks. Run any one or two commands per section — they're meant as a working reference.

### 4.1: `journalctl` — Boot and Service View

Use for boot/reload issues and watching a specific service in real time.

```bash
sudo journalctl --list-boots               # every boot remembered
sudo journalctl -b -p err                  # this boot's errors and above
sudo journalctl -u swss -f                 # follow swss live
sudo journalctl -u bgp  -f                 # follow bgp live
```

### 4.2: `/var/log/syslog` — Unified Application Log

The first place to look for `orchagent`, `fpmsyncd`, `portmgrd`, `frr` messages — all containers interleaved by timestamp.

```bash
sudo tail -F /var/log/syslog
sudo grep -iE 'err|fail|crit' /var/log/syslog | tail -n 50
sudo zgrep -h Ethernet0 /var/log/syslog*    # includes rotated archives
```

### 4.3: Per-Container Logs

Use when the issue is scoped to one service (BGP, syncd, swss, lldp, …).

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}'
docker logs --tail 100 bgp
docker logs -f swss
docker exec -it bgp cat /var/log/frr/bgpd.log | tail -n 50
```

### 4.4: Component Log Levels

Bump verbosity when default `INFO` isn't enough.

```bash
sudo swssloglevel -p                       # show current levels
sudo swssloglevel -l DEBUG -c orchagent    # set orchagent to DEBUG
```

> [!WARNING]
> DEBUG fills `/var/log` quickly — revert with `-l INFO` after collecting.

### 4.5: `show techsupport` — The TAC Bundle

The single artifact to attach to a TAC case.

```bash
sudo show techsupport --since "30 minutes ago"
ls -lh /var/dump/sonic_dump_*.tar.gz
# Copy off the box
scp admin@leaf00:/var/dump/sonic_dump_*.tar.gz .
```

**The tarball contains:**

| Section | Contents |
|---|---|
| `log/` | Full `/var/log/` (syslog, frr, swss, …) |
| `dump/` | All Redis DBs (CONFIG, APPL, ASIC, STATE, COUNTERS) |
| `etc/sonic/` | `config_db.json`, `minigraph.xml` |
| `show_*.txt` | Output of every key `show` command |
| `dmesg`, `lspci`, `ip a`, `ip route` | Linux / system state |

> [!TIP]
> - Boot / service / crash issues → `journalctl`
> - Cross-component timeline → `/var/log/syslog`
> - Scoped to one service → `docker logs <name>` or `/var/log/<app>/`
> - Handing to TAC → `show techsupport`

---

## Lab Wrap-Up

You should now be able to:

- [ ] Trace an IPv4 BGP prefix end-to-end: FRR → zebra → kernel FIB → APPL_DB → ASIC_DB → NPU
- [ ] Distinguish data-plane (TCAM) and control-plane (iptables) ACL enforcement paths in SONiC
- [ ] Author and apply `CTRLPLANE` and `L3` ACLs via JSON, and verify enforcement on each path
- [ ] Use `config checkpoint` / `config rollback` to snapshot and restore configuration
- [ ] Query SONiC telemetry with `gnmic` using both OpenConfig and `sonic-db` origins
- [ ] Pick the right log/artifact (`journalctl`, syslog, `docker logs`, `show techsupport`) for the symptom

### Cheat Sheet

```bash
# --- BGP route tracing ---
vtysh -c "show ip bgp 10.0.0.3"                              # BGP table
vtysh -c "show ip route 10.0.0.3"                            # zebra RIB
ip route show 10.0.0.3                                       # Linux kernel FIB
sudo redis-cli -n 0 HGETALL 'ROUTE_TABLE:10.0.0.3'           # APPL_DB
sudo redis-cli -n 1 KEYS '*' | grep 10.0.0.3                 # ASIC_DB
sudo show platform npu router route-table | grep 10.0.0.3    # NPU

# --- ACLs ---
sudo iptables -L -v -n --line-numbers                        # CTRLPLANE view
sudo config load /tmp/acl_ctrlplane_ssh.json -y              # Apply ACL JSON
show acl table; show acl rule <TABLE>                        # CLI view
aclshow -a                                                   # Data-plane counters
sudo config checkpoint <name>                                # Snapshot
sudo config rollback  <name>                                 # Restore

# --- gNMI / telemetry ---
gnmic -a <ip>:8080 --insecure capabilities
gnmic -a <ip>:8080 --insecure -u admin -p password get \
  --path  "/openconfig-interfaces:interfaces/interface[name=Ethernet0]"
gnmic -a <ip>:8080 --insecure -u admin -p password get \
  --target APPL_DB --path "/PORT_TABLE/Ethernet0"

# --- Troubleshooting ---
sudo journalctl -u swss -f                                   # Follow a service
sudo tail -F /var/log/syslog                                 # Unified app log
docker logs --tail 100 bgp                                   # Per-container
sudo show techsupport --since "30 minutes ago"               # TAC bundle
```

---

**End of Lab**
