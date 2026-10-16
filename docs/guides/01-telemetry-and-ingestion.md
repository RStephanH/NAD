# Guide 01 — Telemetry setup and ingestion design

[Français](01-telemetry-and-ingestion.fr.md) | **English**

**Audience:** anyone reproducing the prototype, or an AI assistant picking up
the work.
**Goal:** get real telemetry flowing from the gateway router to the NAD host,
then design the ingestor from real samples.
**Status:** in progress.

- Flow export: tested on 2026-10-15 (safe mode, reverted afterwards).
- SNMP v3: configured and tested on 2026-10-16, kept on the router (read-only,
  encrypted, restricted to the NAD host).
- Syslog, router load measurement and cross-check: not done yet.

Fill in the results log (section 11) as you go. Items marked **(verify)**
depend on the RouterOS or tool version and must be checked against current
documentation.

## 1. Why this order

The order is: configure the router, prepare the host to receive, capture real
samples, cross-check them, and only then write the ingestor.

- **Parsers written from imagined formats get rewritten.** Real flow records
  have field names and quirks that are only visible once you capture some.
- **Each step has a visible result.** If something is wrong (no packets, empty
  SNMP answers), you know which link of the chain is broken.
- **The cross-check protects the whole project.** If flow data does not roughly
  match the router's own counters, every later result would be built on sand.

## 2. Mental model

The router talks to NAD over three channels. In plain words: NAD can *ask* the
router questions (SNMP), and the router can *send* two kinds of messages on its
own (flow summaries and event logs).

| Channel | Direction | Who starts it | Carries | Port |
|---|---|---|---|---|
| SNMP | NAD to router | NAD asks | Counters, CPU, memory | UDP 161 |
| Flows (NetFlow/IPFIX) | Router to NAD | Router pushes | Summaries of conversations | UDP 2055 (convention) |
| Syslog | Router to NAD | Router pushes | Events (interface down, login failed) | UDP 514 |

What follows from this:

- Configuring the router means telling it **where to send** and **who may ask**.
- The host must be **listening before** the router starts sending.
- Flows and syslog travel over UDP: there are no retries, so packets can be lost
  silently. This is why a cross-check exists.
- The router is only the **messenger**: on the host, every flow packet comes
  from the router, but its content describes the conversations of every user.

## 3. Prerequisites

- Access to the router (SSH or WinBox) with permission to change telemetry
  settings.
- A Linux host on the same LAN as the router (the future NAD host), preferably
  on a cable rather than Wi-Fi (flows and syslog are UDP, with no retries), with
  a fixed address (static DHCP lease).
- On the host: `tcpdump`, `nc` (netcat), `jq`, the Net-SNMP client tools
  (`snmpget`, `snmpwalk`: package `net-snmp` on Arch, `snmp` on
  Debian/Ubuntu), `rsyslog`, and `goflow2` (a release binary such as
  `goflow2-2.2.7-linux-amd64` works without installing Go).
- The router's clock synchronized (see step 1.4).

## 4. Workflow overview

```mermaid
flowchart LR
  S0["Step 0<br/>Back up router config"] --> S1["Step 1<br/>Enable telemetry on router"]
  S1 --> S2["Step 2<br/>Prepare receivers on host"]
  S2 --> S3["Step 3<br/>Capture real samples"]
  S3 --> S4["Step 4<br/>Load measurement and cross-check"]
  S4 --> S5["Step 5<br/>Design the ingestor"]
```

In practice the host side comes first for each channel: the SNMP credentials
(step 2) before the SNMP user on the router (step 1.2), and the flow receiver
(step 2) before the flow export (step 1.1).

---

## 5. Step 0 — Safety net

**Why:** telemetry changes are small, but a mistake on a gateway can cut
connectivity. A backup and a way to undo make experimenting safe.

```
# Free storage first: the export is written to the router's own flash.
# Also gives the hardware model and RouterOS version (results log)
/system resource print

# Save the current configuration (before-nad.rsc) and check it exists
/export file=before-nad
/file print where name~"before-nad"

# FastTrack: no line under the "Flags" header means no FastTrack rule at all
/ip firewall filter print where action=fasttrack-connection
/ip firewall connection print count-only where fasttrack

# LAN layout: bridge name (often not just "bridge"), member ports
# (flag H = hardware offload), and subnets per interface
/interface bridge print
/interface bridge port print
/ip address print

# Existing SNMP setup and input firewall chain
/snmp print
/snmp community print
/ip firewall filter print where chain=input
```

From the host, download the backup and keep it outside the repository. The
router's storage is nearly full on this model, so remove the file afterwards:

```bash
mkdir -p ~/nad-private
scp <admin-user>@<router-ip>:before-nad.rsc ~/nad-private/
```

```
/file remove before-nad.rsc
```

- **Safe mode:** `Ctrl+X` turns it on and the prompt shows `<SAFE>`. Leaving
  safe mode with `Ctrl+X` **keeps** the changes; `Ctrl+D`, or a dropped
  session, **undoes** everything done since safe mode was turned on.
- **Do not commit the export file.** It describes the network and includes the
  router's serial number.
- **Write down** from `/ip address print`: the LAN subnet, the WAN address (it
  may be dynamic) and any VPN subnet. The ingestor needs them (step 5).
- **About FastTrack:** on old RouterOS versions it hid traffic from flow
  export; the documentation states this restriction was removed in 6.33
  **(verify)**. Do not disable it on a production router: record it, and let
  the cross-check (step 4) show whether it matters.

## 6. Step 1 — Enable telemetry on the router

Replace `<nad-ip>` with the NAD host address, `<router-lan-ip>` with the
router's LAN address and `<lan-bridge>` with the bridge name found in step 0.
Check each command with `?` or `print`, because syntax can vary between
RouterOS versions **(verify)**.

### 1.1 Flows (who talks to whom)

**Why:** flow records show conversations (source, destination, ports, bytes).
They are the richest signal for detecting scans, floods and unusual
destinations.

```
# Turn flow export on for the LAN bridge.
# active-flow-timeout: report long transfers every minute instead of only at the end
# inactive-flow-timeout: close idle flows quickly
/ip traffic-flow set enabled=yes interfaces=<lan-bridge> active-flow-timeout=1m inactive-flow-timeout=15s

# Send flow records to the NAD host, NetFlow v9, UDP 2055.
# src-address is required: without it the router may send from 0.0.0.0,
# and Linux silently drops packets with that source address.
/ip traffic-flow target add dst-address=<nad-ip> src-address=<router-lan-ip> port=2055 version=9

# Verify
/ip traffic-flow print
/ip traffic-flow target print
```

**Observed on RouterOS 7.12 (hAP ac²):** exporting from the LAN bridge alone
records **both directions**, each flow once. Outbound flows carry the internal
host address. Inbound flows are recorded before reverse NAT, so their
destination is the router's WAN address, not the internal host. The flow cache
(32k entries by default) used about 6 MB of RAM.

Traffic between two LAN ports in hardware offload is switched by the switch
chip and never reaches the CPU, so it does not appear in flows. This is a known
limit, acceptable because most traffic goes to the Internet.

The real cost of flow export on the router CPU is measured in step 4.

### 1.2 SNMP (counters and device health)

**Why:** SNMP gives interface throughput, CPU and memory. It is also the
reference against which flow data is checked.

Prepare the host credentials first (step 2, "SNMP client"), then on the router,
in safe mode:

**a) Look at what exists.** Note whether a `public` community exists, which
addresses it accepts, and whether the `input` firewall chain has static rules.

```
/snmp print
/snmp community print
/ip firewall filter print where chain=input
```

**b) Disable the default community, before enabling the service.** `public` is
the default name everyone knows, open to every address, with no encryption.
Disabling it first means the router never answers everyone.

```
# Default entries usually cannot be deleted, so disable it (verify)
/snmp community disable [find name=public]

# Check: the "public" line must show an X (disabled)
/snmp community print
```

If the command is refused, restrict it instead:
`/snmp community set [find name=public] addresses=<nad-ip>/32`.

**c) Create the SNMPv3 user** (one line, with the `AUTH` and `PRIV` values
generated on the host):

```
/snmp community add name=nad addresses=<nad-ip>/32 read-access=yes write-access=no security=private authentication-protocol=SHA1 authentication-password=<AUTH> encryption-protocol=AES encryption-password=<PRIV>
```

In RouterOS, `/snmp community` holds access profiles. With `security=private`
the entry is an **SNMPv3 user** (authenticated and encrypted), and `name`
becomes the user name: it must match `defSecurityName` on the host. Recent
RouterOS versions offer stronger algorithms than SHA1 **(verify)**; if you pick
one, change `defAuthType` on the host too.

**d) Enable the service and check:**

```
/snmp set enabled=yes
/snmp print
/snmp community print
```

`nad` must be active and restricted to `<nad-ip>/32`; `public` must be disabled.
`trap-community: public` in `/snmp print` is harmless: without a `trap-target`,
no trap is sent.

**e) Keep or undo.** Once the tests of step 2 pass, the SNMP setup can stay:
read-only, SNMPv3, encrypted, one allowed address. Leave safe mode with
`Ctrl+X` to keep it. A dropped session in safe mode would otherwise undo SNMP
in the middle of a long measurement (step 4).

**Observed (hAP ac², RouterOS 7.12):** the SNMP service was disabled, the
`public` community existed (default, all addresses `::/0`, security `none`),
and the `input` chain only had dynamic rules created by the Hotspot. With no
static rule, nothing blocks or protects access to the router itself.

### 1.3 Syslog (events)

**Why:** device events (interface changes, failed logins) are useful context.
NAD uses them as counts per time window. **Not tested yet.**

```
# Define a destination: the NAD host, UDP port 514
/system logging action add name=nad target=remote remote=<nad-ip> remote-port=514 src-address=<router-lan-ip>

# Send informational and more severe messages to that destination
/system logging add topics=info,warning,error,critical action=nad

# Verify
/system logging action print
/system logging print
```

Setting `src-address` here too avoids the same source-address problem as flows
**(verify)**.

### 1.4 Time

**Why:** NAD aggregates everything into one-minute windows. If the router clock
is off, its data lands in the wrong window.

```
# Check that the router clock is synchronized
/system ntp client print
/system clock print
```

### 1.5 Undo

**Flows, after each test session:**

```
# Remove the export target and restore the defaults
/ip traffic-flow target remove [find dst-address=<nad-ip>]
/ip traffic-flow set enabled=no interfaces=all active-flow-timeout=30m inactive-flow-timeout=15s

# Proof: an export shows only non-default settings, so no /ip traffic-flow
# line must appear below the header
/ip traffic-flow export
```

Then leave safe mode with `Ctrl+D` (undo), never `Ctrl+X` (keep).

**SNMP, only if it must be removed completely:**

```
/snmp set enabled=no
/snmp community remove [find name=nad]
```

Leave `public` disabled even then: it is safer than the original state.

---

## 7. Step 2 — Prepare the receivers on the host

**Why:** the router pushes flows and syslog without waiting. If nothing is
listening, those messages are lost. For SNMP, the host asks, so it needs the
credentials before the router user exists.

If a host firewall is active (check with `sudo ufw status verbose`), allow the
router to reach the flow and syslog receivers. The rule only matches if the
router really sends from `<router-lan-ip>` (see `src-address` in step 1.1).
SNMP is started by the host, so it needs no inbound rule.

```bash
# Allow flows and syslog from the router only
sudo ufw allow from <router-lan-ip> to any port 2055 proto udp comment 'NAD netflow'
sudo ufw allow from <router-lan-ip> to any port 514 proto udp comment 'NAD syslog'
sudo ufw status numbered | grep NAD
```

### SNMP client

```bash
# Client tools only. Do NOT enable the snmpd service: that is the agent,
# which NAD does not need.
sudo pacman -S --needed net-snmp      # Debian/Ubuntu: sudo apt install snmp
snmpget --version
```

Generate two passphrases (Net-SNMP requires at least 8 characters) and store
them in a file only you can read:

```bash
AUTH=$(openssl rand -base64 24 | tr -d '/+=' | head -c 20)
PRIV=$(openssl rand -base64 24 | tr -d '/+=' | head -c 20)

mkdir -p ~/.snmp
(umask 077; cat > ~/.snmp/snmp.conf <<EOF
defVersion 3
defSecurityLevel authPriv
defSecurityName nad
defAuthType SHA
defAuthPassphrase $AUTH
defPrivType AES
defPrivPassphrase $PRIV
EOF
)

# Show them once: they are needed for the router command (step 1.2 c)
echo "AUTH=$AUTH"; echo "PRIV=$PRIV"
```

Clear the screen after copying the two values. With this file, commands shrink
to `snmpget <ip> <oid>`, and no password appears in the shell history or in
`ps`.

### SNMP tests, from simplest to most useful

```bash
R=<router-lan-ip>

# 1) Does the router answer? System description and uptime
snmpget $R 1.3.6.1.2.1.1.1.0 1.3.6.1.2.1.1.3.0

# 2) Interface names with their index (the index is the last number of the OID)
snmpwalk $R 1.3.6.1.2.1.31.1.1.1.1

# 3) CPU load, one value per core
snmpwalk -Oqv $R 1.3.6.1.2.1.25.3.3.1.2

# 4) Storage: description, allocation unit, size, used
snmpwalk $R 1.3.6.1.2.1.25.2.3.1.3
snmpwalk $R 1.3.6.1.2.1.25.2.3.1.4
snmpwalk $R 1.3.6.1.2.1.25.2.3.1.5
snmpwalk $R 1.3.6.1.2.1.25.2.3.1.6

# 5) 64-bit byte counters (in, out) for one interface
snmpget $R 1.3.6.1.2.1.31.1.1.1.6.<idx> 1.3.6.1.2.1.31.1.1.1.10.<idx>
```

| Test | Expected result |
|---|---|
| 1 | A description mentioning the router model, then the uptime |
| 2 | One line per interface, such as `...1.8 = STRING: bridge1` |
| 3 | One percentage per core (four on the hAP ac²) |
| 4 | Find "main memory" in the descriptions; used % = `used / size` on the same row (index 65536 on this router) |
| 5 | Two large numbers that **increase** when you run the command again 10 seconds later |

Notes:

- Write down the **indexes** of the LAN bridge and the WAN interface in the
  results log: the cross-check needs them.
- Counters are read **from the router's point of view**: on the LAN bridge,
  "in" is what LAN machines send out (upload), "out" is what the router delivers
  to them (download).
- Numeric OIDs work without any MIB installed. Names such as `ifName` need MIB
  files, which some distributions do not ship.
- Interface indexes can change when dynamic interfaces appear (VPN sessions).
  For continuous collection, resolve indexes to names (`ifName`) at each read
  instead of relying on a fixed number.

### Flow receiver

```bash
mkdir -p ~/nad-samples

# Receive NetFlow on UDP 2055, decode it, print JSON lines, and keep a copy
# (check the exact flags with -h: they differ between versions)
./goflow2-2.2.7-linux-amd64 -listen netflow://:2055 -format json | tee ~/nad-samples/flows.jsonl
```

```bash
# In another terminal: do raw packets arrive, and from which source?
sudo tcpdump -ni <host-interface> udp port 2055
```

Use the real interface (for example `eth0`) rather than `-i any`: the `any`
pseudo-device prints a harmless "promiscuous mode not supported" warning.

Diagnose in three levels, in this order:

1. **`tcpdump` shows nothing:** router configuration or network path.
2. **`tcpdump` shows packets, but `nc` receives nothing:** something drops them
   between the network card and the application. `tcpdump` captures **before**
   the host firewall, so seeing packets proves nothing about delivery. Check the
   source address in `tcpdump` (`0.0.0.0` is dropped by Linux) and the firewall
   rule.
3. **`nc` receives, but `goflow2` writes nothing:** decoding. A few
   `template error` warnings at start are normal: NetFlow v9 data cannot be
   decoded until the router sends its template.

```bash
# Level 2 test: stop goflow2 first, then listen with a minimal tool
timeout 15 nc -ul 2055 | xxd | head -5
```

goflow2 also serves metrics on port 8080. If another service already uses that
port, start goflow2 with `-addr :9081`.

### Syslog receiver

```bash
sudo mkdir -p /var/log/nad
sudoedit /etc/rsyslog.d/10-nad.conf
```

```
# Listen for syslog on UDP 514
module(load="imudp")
input(type="imudp" port="514")

# Messages from the router go to their own file; "& stop" keeps them out of the
# general system log
if $fromhost-ip == '<router-lan-ip>' then /var/log/nad/router.log
& stop
```

```bash
sudo systemctl restart rsyslog
tail -f /var/log/nad/router.log
```

If the file stays empty, generate an event on the router (for example a login)
and check that the log file is writable by the syslog user.

---

## 8. Step 3 — Capture real samples

**Why:** real records are what the ingestor will be designed from.

- Run the receivers for 20 to 30 minutes while the network is used normally.
- Keep raw captures **outside Git**. Only a small anonymized excerpt belongs in
  `data/samples/`.
- Inspect one record to learn the field names, then summarize the capture:

```bash
F=~/nad-samples/flows.jsonl

# Pretty-print the first flow record
head -n 1 $F | jq .

# Direction split (adjust the LAN prefix and the WAN address)
jq -r 'if (.src_addr|startswith("<lan-prefix>")) then "outbound"
       elif .dst_addr=="<wan-ip>" then "inbound"
       else "other" end' $F | sort | uniq -c

# Distinct internal hosts seen
jq -r 'select(.src_addr|startswith("<lan-prefix>")) | .src_addr' $F | sort -u | wc -l

# Volume, to estimate storage per day
ls -lh $F

# The capture describes real people's activity
chmod 600 $F
```

**Volume:** each raw JSON line is about 1.3 KB, most of it unused fields (BGP,
MPLS, IPv6). On the reference network this is in the order of 1 GB of raw JSON
per day, hence the ingestor's field selection and file rotation.

A test session validates the **pipeline**, not the baseline. The baseline is
built later from continuous collection.

## 9. Step 4 — Load measurement and cross-check

### 4.1 Router load (phases A/B/C)

**Why:** flow export costs CPU and RAM on a modest router that serves the whole
company. One reading is not enough: the load varies with usage. Three phases of
10 minutes each, measured the same way, separate the cost of the export from
the natural variation:

| Phase | Flow export | Purpose |
|---|---|---|
| A | Off | Reference load |
| B | On | Load with export |
| C | Off again | Checks that the load returns to the reference (control) |

Avoid large downloads during the measurement, and run all three phases in a
row, at a similar time of day.

```bash
R=<router-lan-ip>
mkdir -p ~/nad-samples
# Columns: UTC time, mean CPU %, busiest core %, memory used %
while true; do
  cpu=$(snmpwalk -Oqv $R 1.3.6.1.2.1.25.3.3.1.2 | awk '{s+=$1; if ($1>m) m=$1} END {printf "%.1f %d", s/NR, m}')
  mem=$(snmpget -Oqv $R 1.3.6.1.2.1.25.2.3.1.6.65536 1.3.6.1.2.1.25.2.3.1.5.65536 | paste -sd' ' | awk '{printf "%.1f", 100*$1/$2}')
  echo "$(date -u +%FT%TZ) $cpu $mem"
  sleep 10
done | tee -a ~/nad-samples/cpu.log
```

The busiest core matters: one core can saturate before the average looks high.
Note the start time of each phase, then compute the averages per phase:

```bash
# Mean CPU, busiest-core maximum and mean memory between two UTC times
awk -v s="<start>" -v e="<end>" '$1>=s && $1<e {c+=$2; if ($3>m) m=$3; r+=$4; n++}
  END {printf "samples=%d cpu_mean=%.1f core_max=%d mem_mean=%.1f\n", n, c/n, m, r/n}' ~/nad-samples/cpu.log
```

### 4.2 Cross-check flows against SNMP

**Why:** flow export can undercount (accelerated forwarding, hardware offload)
or overcount (the same traffic exported from several interfaces), and UDP can
drop packets. SNMP counters come from the interface itself and serve as the
reference. **Not tested yet.**

Use the LAN bridge only: its two counters cover both directions, and unlike the
WAN they exclude VPN encapsulation and traffic dropped by the firewall.

| SNMP counter on the LAN bridge | Matches flows with | Direction |
|---|---|---|
| In (`ifHCInOctets`, OID ...31.1.1.1.6) | `in_if` = bridge index | Outbound (upload) |
| Out (`ifHCOutOctets`, OID ...31.1.1.1.10) | `out_if` = bridge index | Inbound (download) |

```bash
R=<router-lan-ip>
B=<lan-ifindex>

# 1) Read both counters, start the flow capture on a fresh file, wait 5 minutes,
#    read both counters again
snmpget -Oqv $R 1.3.6.1.2.1.31.1.1.1.6.$B 1.3.6.1.2.1.31.1.1.1.10.$B
sleep 300
snmpget -Oqv $R 1.3.6.1.2.1.31.1.1.1.6.$B 1.3.6.1.2.1.31.1.1.1.10.$B

# 2) Over the same 5 minutes, sum flow bytes per direction
F=~/nad-samples/flows-crosscheck.jsonl
jq -s --argjson b $B 'map(select(.in_if == $b) | .bytes) | add' $F    # outbound
jq -s --argjson b $B 'map(select(.out_if == $b) | .bytes) | add' $F   # inbound
```

Compare each SNMP difference with the matching flow total. Expect rough
agreement, not equality: the bridge counters also include traffic to and from
the router itself (DNS, management, the SNMP polling) and Ethernet framing,
which flows do not count. Flow export also stops being comparable if the
capture misses the first or last seconds. A large gap points to one of the
causes in section 12.

## 10. Step 5 — Design the ingestor

### Concept

In plain words, the ingestor is a sorting center. It reads raw files, drops each
event into the box for its minute, counts what is in each box, and writes one
row per box.

Four stages, each independent so a source can be swapped without touching the
rest:

1. **Read:** one adapter per source (flows, SNMP output, probes, logs).
2. **Normalize:** convert each event to a common internal form (UTC timestamp,
   direction, bytes, addresses).
3. **Aggregate:** assign events to one-minute windows and compute the features.
4. **Write:** store one row per window in the database.

### Design decisions

| Decision | Why |
|---|---|
| All timestamps in UTC, windows aligned to the minute | Avoids time zone and daylight-saving confusion; windows from different sources line up |
| Close a window only after a delay | Flow records arrive late (router timeouts); closing instantly would undercount |
| Idempotent writes (upsert on the window key) | Re-running the ingestor must never duplicate rows |
| Remember the read position per file | Resume after a stop without re-reading or skipping data |
| Direction from `in_if`/`out_if`, not from addresses | The WAN address is dynamic (DHCP); interface indexes are more stable |
| Interface indexes resolved from names at startup | Indexes can change when dynamic interfaces (VPN sessions) appear |
| Per-host features from outbound flows only | Inbound flows carry the WAN address (recorded before reverse NAT) |
| Address classes as configuration: internal (LAN, VPN), upstream (Starlink LAN), Internet | VPN traffic is internal company traffic; the Starlink LAN is neither internal nor Internet |
| Keep only the needed fields at read time | Raw records are ~1.3 KB, mostly unused fields |
| Assign a flow to the window of its end time | Simple and deterministic; with a 1-minute active timeout, a flow never spans more than two windows |
| Raw files compressed and rotated after ingestion | Raw flows are the bulky part; windows are tiny |

Fields used from goflow2 (v2.2.7): `time_flow_start_ns`, `time_flow_end_ns`,
`src_addr`, `dst_addr`, `src_port`, `dst_port`, `proto`, `bytes`, `packets`,
`in_if`, `out_if`, `tcp_flags`, `sampler_address`. Timestamps are in
nanoseconds.

SNMP values used: per-core CPU (`hrProcessorLoad`, OID 1.3.6.1.2.1.25.3.3.1.2),
main memory (`hrStorageSize` and `hrStorageUsed`, index 65536 on this router),
64-bit interface counters (`ifHCInOctets`, `ifHCOutOctets`), interface names
(`ifName`).

Example configuration (values from the reference network):

```yaml
internal_subnets:
  - 192.168.30.0/24   # LAN users
  - 10.0.0.0/24       # WireGuard VPN (internal company traffic)
upstream_subnets:
  - 192.168.1.0/24    # Starlink router LAN (neither internal nor Internet)
lan_interface: bridge1   # resolved to its index (8 on 2026-10-16) at startup
wan_interface: ether1    # resolved to its index (1, to confirm) at startup
```

### Draft schema (adjust after seeing real records)

```sql
-- One row per one-minute window, from flows, counters and logs
CREATE TABLE IF NOT EXISTS windows (
  window_start      INTEGER PRIMARY KEY,  -- unix seconds, UTC, aligned to the minute
  flows             INTEGER,
  bytes_up          INTEGER,
  bytes_down        INTEGER,
  packets_up        INTEGER,
  packets_down      INTEGER,
  distinct_hosts    INTEGER,              -- from outbound flows
  distinct_dests    INTEGER,
  distinct_ports    INTEGER,
  top_talker_share  REAL,                 -- from outbound flows
  cpu_pct           REAL,                 -- mean of cores
  cpu_core_max_pct  REAL,                 -- busiest core
  mem_pct           REAL,
  log_events        INTEGER,
  ingested_at       INTEGER
);

-- One row per window and probe target (several targets per window)
CREATE TABLE IF NOT EXISTS probe_windows (
  window_start  INTEGER,
  target        TEXT,                     -- e.g. Starlink router (first hop), Internet targets
  rtt_mean_ms   REAL,
  rtt_p95_ms    REAL,
  loss_pct      REAL,
  PRIMARY KEY (window_start, target)
);
```

Probes live in their own table because each window has one row per target.
Probing both the Starlink router and Internet targets tells a local link
problem apart from a satellite link problem.

## 11. Results log

Fill this in after each session. These values feed the decisions in
`docs/tools/tooling.md`.

| Item | Flow test (2026-10-15) | SNMP setup (2026-10-16) | Continuous collection |
|---|---|---|---|
| Router model | hAP ac² (RBD52G-5HacD2HnD), 4 × ARM 672 MHz, 128 MB RAM | Confirmed by SNMP | |
| RouterOS version | 7.12 (stable) | | |
| Router uptime at measurement | 22 min, then 1 h 21 | 23 h 45 | |
| Router free storage | ~0.4 MB of 15.2 MB | 0.32 MB (97.9 % used) | |
| FastTrack enabled | No (no rule configured) | | |
| CPU load, export off | 10 % (just after a reboot) | 8 % mean (cores 6/4/10/12), single reading | |
| CPU load, export on | 31 % (not comparable) | | |
| Free memory, export off | 46.9 MB (22 min uptime) | 36.6 MB (71.4 % used) | |
| Free memory, export on | 41.1 MB | | |
| Interface indexes | Flows: inbound `in_if` 1, outbound `in_if` 8 | `ether1`–`ether5` = 1–5, `wlan1`/`wlan2` = 6/7, `bridge1` = 8, then `wg-vpn`, `l2tp-vpn`, `ovpn-vpn` | |
| WAN interface | `ether1` (carries the WAN address) | Index 1, to confirm with counters | |
| LAN bridge counters | | In 8.7 GB, out 55.5 GB over 23 h 45 (≈ 0.8 Mbit/s up, 5.2 Mbit/s down) | |
| Flow export interface choice | LAN bridge only: both directions, no double counting | | |
| Flows total vs SNMP counters (ratio) | | Not measured yet | |
| Addresses in flows | Outbound: internal hosts. Inbound: router WAN address | | |
| SNMP setup | | v3 user `nad`, authPriv (SHA1/AES), restricted to the NAD host; `public` disabled; kept on the router | |
| Syslog format observed | Not tested yet | | |
| Notes | Double NAT behind the Starlink router; LAN ports in hardware offload | Hotspot active (dynamic `input` rules); no static `input` firewall rule; three VPN interfaces (WireGuard, L2TP, OpenVPN) | |

## 12. Troubleshooting

### Flows

| Symptom | Likely cause | What to check |
|---|---|---|
| No packets in `tcpdump` on port 2055 | Router not exporting, wrong target address | `/ip traffic-flow target print`, same LAN |
| `tcpdump` shows packets, `nc` and `goflow2` receive nothing | Source address `0.0.0.0` (dropped by Linux as "martian") or host firewall | Source in `tcpdump`; target `src-address`; `sudo ufw status` |
| Firewall rule added but still nothing | Rule matches the router address, packets come from `0.0.0.0` | Set `src-address` on the target first |
| `template error` warnings in goflow2 | Template not received yet | Normal for the first seconds; restart goflow2 if it lasts more than 2 minutes |
| "promiscuous mode not supported" warning | `tcpdump -i any` | Harmless; use the real interface |
| goflow2 metrics unreachable on port 8080 | Port used by another service | `sudo ss -tlnp \| grep 8080`; start goflow2 with `-addr :9081` |
| Flow bytes much lower than SNMP | Accelerated forwarding or hardware offload, packet loss | FastTrack rule, bridge port flags, UDP drops on the host |
| Flow bytes higher than SNMP | Same traffic exported from several interfaces | Export interface choice (`<lan-bridge>` versus `all`) |
| Router CPU rises noticeably | Flow export load | Phases A/B/C (step 4.1); adjust timeouts or enable sampling |

### SNMP

| Symptom | Likely cause | What to check |
|---|---|---|
| `Timeout: No Response from <ip>` | Service disabled, wrong address in the community, `input` firewall rule | `/snmp print`; `addresses=` equals the host IP; `input` chain |
| `Unknown user name` | Community name differs from `defSecurityName` | `/snmp community print`; `~/.snmp/snmp.conf` |
| `Authentication failure` | Different `AUTH` passphrase or algorithm (SHA1 versus another) | Same value and algorithm on both sides |
| `Decryption error` | Different `PRIV` passphrase or cipher | Same value on both sides |
| `No Such Object available` | OID not exposed by the router | `snmpwalk` the parent branch |
| A counter does not change | Wrong interface index | Repeat test 2 |
| "Unknown Object Identifier" warning | Names used without MIB files | Use numeric OIDs |
| `command not found` | Client tools not installed | `sudo pacman -S --needed net-snmp` |

### Other

| Symptom | Likely cause | What to check |
|---|---|---|
| Syslog file empty | Port 514 blocked, wrong filter address, file permissions | `tcpdump` on UDP 514, `fromhost-ip` value, ownership of `/var/log/nad` |
| Windows misaligned between sources | Router clock not synchronized | `/system ntp client print` |
| Configuration gone after closing SSH | Session closed while still in safe mode | Expected for tests; `Ctrl+X` keeps changes when you want them permanent |

## 13. Next steps

1. Confirm the WAN interface index with its counters.
2. Router load, phases A/B/C (step 4.1).
3. Cross-check flows against the LAN bridge counters (step 4.2).
4. Syslog (step 1.3).
5. Telegraf configuration for SNMP and probes, with index-to-name resolution
   (guide 02).
6. Ingestor skeleton under `src/nad/collect`, with adapters and tests on the
   captured samples.
7. Ansible roles that install and configure the receivers on the NAD host.

Later hardening, outside this guide: a static `input` firewall rule that only
accepts UDP 161 from the NAD host. Not now: the router runs a Hotspot, and
changing a production gateway's firewall is a separate project.

## Glossary

- **Flow:** a summary of one conversation between two machines (addresses,
  ports, bytes, packets).
- **SNMP:** a standard protocol to ask a device for its counters and status.
- **Community (SNMP v1/v2c):** a shared password sent in clear text with every
  request. `public` is the universal default.
- **SNMPv3 user:** an account with authentication and, at `authPriv` level,
  encryption. In RouterOS it is an entry of `/snmp community` with
  `security=private`.
- **OID:** the numeric address of a value in SNMP, such as
  `1.3.6.1.2.1.1.3.0` for the uptime.
- **MIB:** a catalogue that gives names to OIDs. Numeric OIDs work without it.
- **ifIndex:** the number SNMP and flow records use to identify an interface.
- **Syslog:** a standard protocol for devices to send event messages.
- **Window:** a fixed slice of time (one minute) into which all events are
  grouped.
- **Idempotent:** running an operation twice gives the same result as running
  it once.
- **FastTrack:** a MikroTik feature that speeds up forwarding by bypassing part
  of the packet processing.
- **NAT:** the router replaces internal addresses with its own before sending
  traffic out. Behind the Starlink router, there are two NAT layers (double NAT).
- **Bridge:** a virtual switch grouping several ports into one LAN interface.
- **Hardware offload:** traffic switched by the switch chip without reaching the
  CPU; it does not appear in flow export.
- **Hotspot:** a MikroTik feature for guest access with a login page; it adds
  dynamic firewall rules.
- **Template (NetFlow v9):** a message describing the field layout; data
  records cannot be decoded before it arrives.
- **Safe mode:** a RouterOS session mode where changes are undone unless you
  explicitly keep them.
- **Martian packet:** a packet with an impossible source address (such as
  `0.0.0.0`), dropped by the Linux kernel.
