# NAD — Tooling: comparison and decisions

**English** | [Français](tooling.fr.md)

This document compares the candidate tools for each NAD component, records the
decision taken for the prototype, and says when that decision should be
revisited. It is written to be read on its own, by a person or by an AI
assistant that has not seen the rest of the repository.

**Status:** prototype decisions, to be re-validated once the real environment is
known. Items marked **(verify)** are claims that depend on tool versions or on
the target environment and must be checked against current documentation.

**Last updated:** 2026-10-19 — added WireGuard as the telemetry transport (D12);
Terraform/OpenTofu (D10) marked as deferred; main router (RB1100AHx4) and
network size (about thirty users) filled in.

## 1. Context

| Item | Value |
|---|---|
| Goal | Detect abnormal network behavior automatically (unsupervised) |
| Reference environment | Company network, MikroTik RouterBOARD RB1100AHx4 main router (RouterOS version to be confirmed) connected to the Starlink uplink; all other switches and routers connect downstream of it |
| Network size | About thirty users; number of devices (probably higher) to be confirmed on site |
| Router access | The main router is not yet available for tests; a laptop temporarily plays the router's role in the tunnel |
| Placement | NAD runs on a separate Linux host, never in the traffic path |
| Telemetry transport | Outbound WireGuard tunnel from the company network to a remote host with a public address (see 4.12), or a machine on the local network |
| Operator profile | One engineer; comfortable with Linux, Python, Go; CLI-first |
| Hard rule | Read-only on production devices: telemetry export only |

## 2. Evaluation criteria

Every candidate is judged on the same questions:

1. **Passive and read-only:** does it work from exported telemetry, without
   touching routing or firewall behavior?
2. **Moving parts:** how many services, databases and agents does it add?
3. **Reproducibility:** can it be installed and configured by Ansible?
4. **ML-friendliness:** can its data be exported to tabular form (SQL, CSV,
   Parquet) without a custom decoder?
5. **Vendor neutrality:** does it still work if the router is replaced?
6. **Resource footprint:** does it fit a modest Linux host?
7. **Maturity:** is it maintained, documented and widely used?
8. **Skill fit:** does it match Python/Go and CLI workflows?

## 3. Decision summary

| ID | Component | Decision | Runner-up |
|---|---|---|---|
| D1 | Flow collection | goflow2 | nfdump/nfcapd |
| D2 | Device metrics (SNMP) | Telegraf `inputs.snmp` | Prometheus + snmp_exporter |
| D3 | Active probes | Telegraf `inputs.ping` and `inputs.dns_query` | fping scripts |
| D4 | Logs | rsyslog | syslog-ng |
| D5 | Storage | SQLite (WAL) + Parquet export | InfluxDB |
| D6 | Detector | Isolation Forest + seasonal baseline | LOF / One-Class SVM (comparison only) |
| D7 | Dashboard | Streamlit | Grafana |
| D8 | Alerting | Optional webhook from the detector | Grafana / Alertmanager |
| D9 | Deployment | Ansible + systemd | Docker Compose |
| D10 | Lab provisioning (optional, **deferred**) | Terraform or OpenTofu + libvirt | Ansible `community.libvirt` |
| D11 | Uplink status (optional) | Starlink dish gRPC tools | none |
| D12 | Telemetry transport | WireGuard, remote host as rendezvous point | Tailscale / Headscale |

---

## 4. Components

### 4.1 Flow collection (D1)

**What it does, in plain words:** the router can send a short summary of every
conversation it carries (who talked to whom, on which port, how many bytes).
A flow collector receives these summaries and turns them into readable records.

| Tool | Specialty | Best when | Limits |
|---|---|---|---|
| **goflow2** | Lightweight collector written in Go; decodes NetFlow v5/v9, IPFIX and sFlow; emits JSON | You build your own pipeline and want clean, parseable output | Does not store, aggregate or display anything; you write the ingestion |
| nfdump / nfcapd | Classic collector with binary storage and a powerful CLI for aggregation | You prefer querying with command-line tools, or analyze offline | Binary format needs a conversion step; less convenient for streaming |
| pmacct (nfacctd) | Aggregation at collection time, many output backends (SQL, files, brokers) | You want pre-aggregated data written straight to a database | Heavier configuration, steeper learning curve |
| ntopng | Web interface with real-time host statistics | Quick visibility or a live demo | Less suited to exporting a dataset; some features depend on the license tier **(verify)** |
| Telegraf `inputs.netflow` | Same agent as SNMP and probes | You want one agent for everything | Availability depends on the Telegraf version **(verify)**; less control over raw fields |
| Filebeat / Logstash netflow | Integration into an Elastic stack | ELK already exists | Heavy for this use case |
| Zeek / Suricata on mirrored traffic | Passive capture with rich per-connection logs | A switch mirror port is available | Needs a mirror port or tap; higher CPU and storage |

**Decision D1: goflow2.** It does one job, is easy to script against (JSON
lines), and fits a Go/Python skill set. Each flow record is read by the Python
ingestor and aggregated into time windows.

**Revisit if:**

- Telegraf's netflow input is confirmed on the installed version: merging flows
  and SNMP into one agent removes a component.
- A mirror port exists: add Zeek for deeper per-connection metadata.
- The company already runs ntopng or a flow tool: reuse it rather than duplicate.

**Verify on site:** whether exported flows carry internal or translated (NAT)
addresses. Per-user attribution depends on it. Also check whether accelerated
forwarding on the router (for example FastTrack on MikroTik) hides part of the
traffic from flow export, and whether flow export raises router CPU noticeably.
On the RB1100AHx4, also check which ports use hardware switching (bridge
offload): traffic switched only between ports of the same bridge may never
reach the CPU, and therefore may not appear in the flows **(verify)**.

### 4.2 Device metrics via SNMP (D2)

**In plain words:** SNMP is a standard way to ask a device "how much traffic
went through this port, how busy is your CPU?". A poller asks every few seconds
and records the answers.

| Tool | Specialty | Best when | Limits |
|---|---|---|---|
| **Telegraf `inputs.snmp`** | Configuration-only poller, many output formats | You want zero code for a commodity task | SNMP table configuration is verbose |
| Python (pysnmp or similar) | Full control in your own code | Unusual OIDs or custom logic | You write and maintain it; library maintenance has been uneven **(verify)** |
| Prometheus + snmp_exporter | Industry-standard monitoring with a large ecosystem | You want the Prometheus world (Grafana, Alertmanager) | Adds a time-series database and a pull model |
| Zabbix / LibreNMS / PRTG | Complete monitoring suites with discovery and alerting | The company already runs one | Heavy to install for this alone; data extraction for ML goes through an API or database |
| Vendor API (RouterOS API) | Richer, vendor-specific data (DHCP leases, wireless clients, queue stats) | You need to map an IP to a device or user | Vendor-specific; needs a dedicated adapter per vendor |

**Decision D2: Telegraf SNMP.** The poller becomes a configuration file that
Ansible renders from a template, which is both faster and more reproducible
than custom code. The vendor API stays an **optional complement** for IP-to-host
mapping (DHCP leases).

**Security notes:** use SNMPv3 where the device supports it, otherwise restrict
the community to the NAD host's address; use a read-only account. With the
WireGuard tunnel (D12), SNMP polling travels inside the encrypted tunnel, to
the router's tunnel address.

**Revisit if:** the company already has an SNMP-based monitoring system. Pull
from it instead of polling twice.

### 4.3 Active probes (D3)

**In plain words:** the NAD host regularly pings a few destinations and measures
response time and loss. This is the most direct view of how healthy the uplink is.

| Tool | Specialty | Best when | Limits |
|---|---|---|---|
| **Telegraf `inputs.ping` + `inputs.dns_query`** | Same agent as SNMP; also measures DNS lookup time | You want probes in the same pipeline | Needs permission to send ICMP |
| fping / ping scripts | Very simple | A handful of targets | You handle scheduling and parsing |
| Smokeping | Long-term latency and jitter history, good visuals | Human review of link quality | RRD storage is awkward for ML export |
| blackbox_exporter | ICMP, HTTP, DNS and TCP probes | You already run Prometheus | Requires Prometheus |

**Decision D3: Telegraf.** Probe several targets (the gateway, the next hop, and
two or three public destinations) so that a local problem can be told apart from
an upstream one. Add DNS and HTTP probes later if user-experience metrics are
needed.

**Caution:** probes launched from a remote host would measure that host's own
link, not the company network's. They must run from a host on the monitored
network, or target the router through the tunnel (which then also measures the
tunnel itself).

**Linux precaution:** on a minimal VPS, the `ping` command may lack permission
to open raw sockets (`cap_net_raw`). Fix it with
`sudo apt install --reinstall iputils-ping`, or `setcap cap_net_raw+p` on the
binary. The Telegraf service needs the same right.

### 4.4 Logs (D4)

**In plain words:** the router can report events (interface down, login failed)
to a log receiver. NAD uses them mainly as counts per time window.

| Tool | Specialty | Best when | Limits |
|---|---|---|---|
| **rsyslog** | Ubiquitous, simple receiver writing to files | A reliable inbox is enough | Parsing is your job |
| syslog-ng | Built-in filtering and parsing | You want structured fields early | Slightly more configuration |
| Telegraf `inputs.syslog` | Keeps everything in one agent | One-agent design | Expects a specific syslog format; device format compatibility must be checked **(verify)** |
| Loki + agent | Lightweight, label-based log storage | A Grafana-centric stack | Extra services |
| ELK / OpenSearch / Graylog | Full-text search and analytics | Heavy log analysis or an existing deployment | Resource-hungry |

**Decision D4: rsyslog.** Logs feed only count-type features (events per
window, by category), so a simple receiver is enough.

### 4.5 Storage (D5)

**In plain words:** somewhere to keep the one-row-per-minute table that the
detector reads, plus a way to export it as the project's dataset.

| Tool | Specialty | Best when | Limits |
|---|---|---|---|
| **SQLite (WAL mode)** | One file, SQL, no administration | One writer, modest volume | Limited concurrent writers; not for heavy ingestion |
| **Parquet files** | Compressed columnar files, ML-friendly | Dataset export and notebooks | Not designed for row-by-row appends |
| DuckDB | Fast analytical SQL over Parquet and SQLite | Exploration and notebooks | Single-writer; not a streaming store |
| InfluxDB | Time-series native, retention and downsampling | Continuous metrics with a dashboard | Extra service; query language differs between major versions **(verify)** |
| TimescaleDB | PostgreSQL plus time-series features | The team already runs PostgreSQL | Administration overhead |
| VictoriaMetrics / Prometheus TSDB | Efficient metrics storage | Metrics-only pipelines | Poor fit for flow records |
| ClickHouse | Very large volumes of flow data | Heavy retention or large networks | Overkill at this scale |
| Elasticsearch / OpenSearch | Search and analytics | An ELK stack exists | Resource-hungry |

**Decision D5: SQLite in WAL mode for the operational store, Parquet for
exports.** Aggregated windows are tiny (one row per minute). The bulky part is
raw flow files, which should be compressed and rotated after ingestion.
DuckDB is a convenient optional tool for notebooks.

**Revisit if:** several processes must write at once, retention grows to many
months, or flow volume becomes large.

### 4.6 Detector (D6)

**In plain words:** the detector learns what a normal minute looks like at a
given time of the week and flags minutes that look unusual. Two complementary
methods are combined: one looks at many measurements together, the other knows
the weekly rhythm.

| Method | Specialty | Best when | Limits |
|---|---|---|---|
| **Isolation Forest** | Fast, few assumptions, many features | First model; mixed numeric features | Has no notion of time; threshold needs tuning |
| **Seasonal baseline** (robust z-score per hour-of-week, or STL / Holt-Winters via statsmodels) | Explicit daily and weekly cycles, very explainable | Strong rhythms in usage | Needs enough history per bucket; sparse buckets need a fallback (for example hour-of-day only) |
| Local Outlier Factor | Local density comparison | Clusters of different density | Sensitive to neighbor count; needs novelty mode to score new data |
| One-Class SVM | Boundary around normal data | Small datasets | Scales poorly; sensitive to kernel and parameters; needs scaling |
| Elliptic Envelope / Mahalanobis | Cheap, statistical | Roughly Gaussian features | Weak on multimodal traffic |
| Forecast-residual models (Prophet-style) | Seasonality plus holidays | Long histories | Heavier dependency; more setup |
| Autoencoder / LSTM | Complex temporal patterns | Large datasets | Data-hungry, harder to explain, heavy tuning |
| PyOD | Unified API over many detectors | Side-by-side comparison experiments | Another dependency |

**Decision D6: Isolation Forest combined with a robust seasonal baseline.** The
forest handles combinations of features; the baseline handles time. Each anomaly
event stores its top contributing features so it can be explained. LOF and
One-Class SVM are run only as comparisons in a notebook.

**Uplink instability:** satellite links have natural latency variation and
brief drops. Keep link-quality features separate, and judge them against their
own baseline, so that normal uplink variation is not reported as internal
misbehavior.

**Behavior on a modest-size network (about thirty users):** a few devices
(downloads, updates, video calls) can still dominate the traffic of a given
minute, so normal bursts may look large. Prefer robust statistics (median and
MAD instead of mean and standard deviation), consider wider windows (for
example 5 minutes) to smooth short bursts, and start with tolerant thresholds,
tightening them using the observed false-alarm rate. At this scale, diversity
features (distinct hosts, distinct destinations) carry a modest but usable
signal, and per-device profiles remain feasible once enough history has
accumulated; they can be added later.

**Revisit if:** the history is long enough for forecast-based methods, or the
seasonal buckets are too sparse (fall back to coarser buckets).

### 4.7 Dashboard (D7)

| Tool | Specialty | Best when | Limits |
|---|---|---|---|
| **Streamlit** | Pure-Python web UI | Custom rendering (anomalies highlighted, event table) | No built-in alerting; no built-in authentication |
| Grafana | Standard monitoring dashboards, alerting, many data sources | Data lives in a compatible time-series store | Highlighting model scores needs extra setup; SQLite support relies on a plugin **(verify)** |
| Dash (Plotly) | More control than Streamlit | A polished, structured app | More boilerplate |
| Textual / Rich (terminal UI) | CLI-first status view | Quick checks over SSH | Not suited to sharing with non-technical staff |
| Kibana | Integrated with ELK | ELK already exists | Only worthwhile with that stack |

**Decision D7: Streamlit.** The anomaly view is custom logic, which is simplest
in Python. Streamlit has no authentication of its own: bind it to an internal
interface, or place it behind a reverse proxy or VPN. On a VPS with a public
address, never expose it on the public address: bind it to the WireGuard
tunnel address (D12), or to `127.0.0.1` with an SSH tunnel.

### 4.8 Alerting (D8)

| Option | Best when | Limits |
|---|---|---|
| **Events table + dashboard (default)** | Nothing more is needed | Someone must look at the dashboard |
| Webhook from the detector (chat tool, ntfy, Telegram-style bot) | Simple push notifications | Channel choice depends on the company |
| Email (SMTP) | The company relies on email | Needs a reachable mail relay |
| Grafana alerting / Alertmanager | A Grafana or Prometheus stack exists | Requires that stack |

**Decision D8:** write events to the store and show them on the dashboard; add
an optional webhook once the company's preferred channel is known.

### 4.9 Deployment and configuration (D9)

| Tool | Specialty | Best when | Limits |
|---|---|---|---|
| **Ansible + systemd** | Agentless, idempotent roles over SSH | One or a few hosts, no container layer wanted | Requires Python on the target; manual edits cause drift |
| Docker Compose | Packaged, portable multi-service stacks | A server with a container engine | Adds an engine; UDP receivers need careful port or host-network setup |
| Ansible deploying Compose | Both worlds | A server-class host | Two layers to understand |
| Shell scripts | Quick | Throwaway setups | Not idempotent |
| Puppet / Salt / Chef | Large fleets | Many hosts | Overkill here |
| Kubernetes / Nomad | Orchestration at scale | Large platforms | Far beyond this scope |

**Decision D9: Ansible with systemd services.** The same roles must work for the
lab host and for a real host, so roles must not assume how the host was created.
Keep one inventory per environment, and keep secrets out of Git (Ansible Vault
or environment files). The WireGuard configuration (keys, config file,
firewall) is a good candidate for a dedicated Ansible role once the manual
procedure is validated.

### 4.10 Lab provisioning, optional and deferred (D10)

**Status: deferred.** Terraform (or OpenTofu) is not used in the project so far.
It is planned for the repository, but it is not urgent. The current
infrastructure (VPS, WireGuard tunnel, lab) is set up by hand and then
documented; Ansible (D9) comes first.

The lab recreates the stack in virtual machines so it can be tested without
touching the real network. It is not needed in production.

| Tool | Specialty | Best when | Limits |
|---|---|---|---|
| **Terraform + libvirt provider** | Declarative VMs on KVM | You want infrastructure as code for the lab | State handling; provider schema has changed across versions, so pin the version **(verify)** |
| OpenTofu | Open-source fork of Terraform, largely compatible | License preference | Same provider caveats |
| Ansible `community.libvirt` | Manage VMs from Ansible alone | You want fewer tools | Less declarative than Terraform |
| Vagrant | Quick developer VMs | Simple local setups | Plugin maintenance; HashiCorp licensing |
| virt-install + cloud-init scripts | Minimal | One-off VMs | No state tracking |
| Incus / LXD | System containers, light and fast | Low-resource labs | Not full VMs |
| Virtual router images (for example RouterOS CHR) | Test router automation safely | You need a router without touching hardware | x86 image; lacks wireless and some hardware features |

**Decision D10: Terraform or OpenTofu with libvirt, optional, deferred.** Treat
it as a lab convenience, never as a dependency of the product.

**Revisit when:** the Ansible roles are stable and the lab should be made
reproducible from the repository (also useful for demonstrations). If the VPS
itself is ever to be described as code, first check whether the hosting
provider has a maintained Terraform provider **(verify)**.

### 4.11 Uplink status, optional (D11)

| Tool | Specialty | Best when | Limits |
|---|---|---|---|
| **Starlink dish gRPC tools (community)** | Dish status: latency, obstruction, outages, throughput | Telling uplink faults apart from internal ones | Unofficial interface that may change with firmware; only reachable from the dish's own network **(verify)** |

**Decision D11:** an optional extra source behind a collector adapter. It must
not be on the critical path.

### 4.12 Telemetry transport (D12)

**In plain words:** the router sits behind the Starlink dish and a second level
of address translation (NAT). Nobody on the Internet can "knock on its door",
but it can call out. So we place a rendezvous point with a fixed public address
(a VPS). The router calls it and keeps the line open. That line is an encrypted
tunnel: telemetry (flows, SNMP, logs) travels through it, and the VPS can also
reach the router over the same channel. When NAD runs on a machine inside the
local network, no tunnel is needed: the router simply exports telemetry to it
locally.

Each device has a key pair (a private key, which never leaves the device, and
a public key, given to the other end) and an address in a small private network
that exists only inside the tunnel.

| Tool | Specialty | Best when | Limits |
|---|---|---|---|
| **WireGuard, VPS as rendezvous point** | Built into the Linux kernel and native in RouterOS 7; short configuration; a single UDP port; silent toward any packet it does not recognize | Few devices, and you need an encrypted path that is simple to explain | Manual key management; no automatic discovery; the UDP port must be open at every level (see below) |
| Tailscale / Headscale | Automatic key management and NAT traversal | Many devices, or fine-grained ACLs needed | Depends on a coordination service (or a self-hosted Headscale); no native client on RouterOS **(verify)** |
| IPsec / IKEv2 | Standard, native on RouterOS | The company mandates IPsec | More complex configuration, harder to debug |
| OpenVPN | Very widespread | Compatibility with an existing fleet | Heavier; limited RouterOS support **(verify)** |
| Reverse SSH tunnel | Very simple, no VPN | A single one-off TCP flow | NetFlow, SNMP and syslog are UDP: poor fit |
| Cloudflare Tunnel | Exposes web services without an open port | HTTP/TCP services | Oriented to HTTP/TCP; poor fit for UDP telemetry flows **(verify)** |

**Decision D12: WireGuard, with the VPS as the rendezvous point.** It has the
fewest moving parts, works on both sides without extra software, and stays under
the team's control.

**Tunnel addressing plan:**

| Role | Address inside the tunnel |
|---|---|
| VPS (listens on UDP 51820) | `10.100.0.1/24` |
| MikroTik (final target) | `10.100.0.2` |
| Laptop (temporary stand-in for the router) | `10.100.0.3` |

**Configuration principles:**

- The peer behind the NAT (router or laptop) calls the VPS first and sends a
  keepalive packet every 25 seconds (`PersistentKeepalive`): this keeps the
  "hole" open in the NATs.
- Each peer has its own keys and an `AllowedIPs` limited to its own address
  (`/32`) on the VPS side. On the peer side, `AllowedIPs` is limited to the
  VPS address (`/32`): only NAD traffic goes through the tunnel, not all
  Internet traffic.
- Private keys and configuration files never go into the repository
  (`.gitignore`); only the procedure is documented.
- NAD services (flow collector, syslog receiver, dashboard) listen on the
  tunnel address, never on `0.0.0.0`, so they are not exposed on the VPS's
  public address.
- On the RouterOS side, the default firewall drops anything that does not come
  from the LAN: an `input` rule allowing the WireGuard interface must be placed
  **before** the drop rule. Use Safe Mode when making the change remotely.
- The UDP port must be opened at **two independent levels**: the host firewall
  (for example `ufw`) **and** the VPS provider's firewall in its web panel.
  This was the cause of the prototype's first outage: the WireGuard
  configuration was correct, but the provider's firewall was blocking inbound
  UDP.

**Router prerequisite:** WireGuard exists only in RouterOS 7. The version
installed on the RB1100AHx4 must be confirmed. If it is RouterOS 6, there are
two paths: a major upgrade of the router (a production change, to be validated
and scheduled, with a configuration backup), or a setup that does not touch the
router: a small Linux host on the LAN (the future NAD host) carries the
WireGuard tunnel to the VPS, and the router exports its telemetry to it
locally. The second setup respects the read-only rule better, at the cost of
one more host to maintain on site.

**Revisit if:** the number of devices grows beyond a handful, key rotation
becomes a constraint, or the company already uses Tailscale or another VPN: in
that case, reuse what exists rather than maintaining a second system.

---

## 5. Stack profiles by environment

The pipeline only needs a normalized stream of windows, so the stack can be
swapped per environment by changing collectors and, if needed, storage and
dashboard.

| Component | A. Minimal (current prototype) | B. Capable router + server | C. Existing monitoring stack | D. Restricted access |
|---|---|---|---|---|
| Transport | WireGuard, VPS as rendezvous point | Local network, or WireGuard if the site is remote | Existing network and VPN | Depends on company policy |
| Flows | goflow2 | goflow2, plus mirror port and Zeek if available | goflow2 or reuse the company's flow tool | Passive capture on a mirror port, or none |
| Device metrics | Telegraf SNMP | Prometheus + snmp_exporter, or Telegraf | Pull from the existing system (API or database) | SNMP only if allowed |
| Probes | Telegraf | Telegraf or blackbox_exporter | Reuse existing probes if comparable | Telegraf (no router access needed) |
| Logs | rsyslog | Loki or ELK | Existing log platform | rsyslog if syslog export is allowed |
| Storage | SQLite + Parquet | TimescaleDB / VictoriaMetrics; ClickHouse if flow volume is high | Existing store plus SQLite for windows | SQLite + Parquet |
| Detector | Isolation Forest + seasonal | Same, with comparisons | Same | Same |
| Dashboard | Streamlit | Grafana (+ Alertmanager) | Existing dashboards plus Streamlit for scores | Streamlit |
| Deployment | Ansible + systemd | Ansible + Docker Compose | Ansible | Ansible |

## 6. Fallback modes

| Condition | How to notice it | Fallback | What is lost |
|---|---|---|---|
| WireGuard tunnel without handshake | `wg show` shows no `latest handshake`; `0 B received` on the peer side | Check in order: provider firewall, host firewall, endpoint IP, public keys on both sides; `tcpdump -ni any udp port 51820` on the VPS shows whether packets arrive | All telemetry while the tunnel is down |
| Handshake established but no ping | Recent handshake, but no reply | Check `AllowedIPs` on both sides and the router's `input` rule (order) | VPS access to the router |
| Flow export unavailable or incomplete | No flow packets arrive, or counters disagree with SNMP | SNMP counters and probes only | Attribution to hosts and destinations |
| SNMP blocked | Poller returns timeouts | Flows plus probes; vendor API if allowed | Interface and CPU metrics |
| Syslog export not allowed | No log events | Drop log features | Device event context |
| Router CPU too high with export enabled | Elevated CPU after enabling telemetry | Reduce export scope or sampling; poll less often | Granularity |
| No router access at all | Request refused | Passive capture from a switch mirror port | Router health metrics |
| Not enough history yet | Sparse seasonal buckets | Coarser buckets (hour-of-day); develop on public datasets and lab traffic | Weekly-cycle precision |
| Raw flow retention not permitted | Policy decision | Keep only aggregated windows; delete raw flow files after ingestion | Ability to re-aggregate or investigate |
| Dish status unreachable | gRPC tool cannot connect | Skip the optional source | Uplink-fault attribution |

## 7. Questions to answer on site

These decide which profile in section 5 applies.

| Question | Why it matters | Affects |
|---|---|---|
| RouterOS version on the RB1100AHx4 (the model is known)? | WireGuard exists only in RouterOS 7; which telemetry features are available | D1, D2, D12 |
| Is accelerated forwarding (FastTrack) enabled? | May hide traffic from flow export | D1 |
| Do exported flows show internal or NAT'd addresses? | Per-user attribution | D1, features |
| How many devices for the thirty or so users, and which are shared or unmanaged? | Window size, thresholds and usefulness of diversity features | D6, features |
| Is there a managed switch with a mirror port? | Passive capture and Zeek become possible | Fallbacks, profile B/D |
| What server hardware is available, and does it support virtualization? | Containers, lab, heavier storage | D9, D10, profile B |
| Is monitoring or log management already in place? | Reuse instead of duplicating | Profile C |
| Retention and privacy rules for flow data? | What may be stored and for how long | D5, section 6 |
| Where do DHCP, DNS and Wi-Fi run? | Mapping IPs to users and devices | Optional RouterOS API, features |
| Is the dish's management address reachable from the LAN? | Optional uplink source | D11 |
| Does the company allow an outbound tunnel to an external server, and may telemetry leave the site? | The VPS-based transport assumes flows leave the network | D12, section 6 |
| Who authorizes anomaly injection tests, and when? | Validation requires a window | Validation plan |

## 8. Checks to run now

### 8.1 On the router (read-only)

Before any code, confirm the telemetry chain end to end on the router, as soon
as access is possible. These are read-only checks on the router:

```
/system resource print
/ip firewall filter print where action=fasttrack-connection
/ip traffic-flow print
/snmp print
```

Then on the NAD host:

```bash
# Do flow packets arrive? (2055 is the common NetFlow/IPFIX port, check your config)
sudo tcpdump -ni any udp port 2055

# Does SNMP answer? (replace the address and credentials)
snmpwalk -v2c -c <community> <router-ip> 1.3.6.1.2.1.2.2.1.10

# Do syslog messages arrive?
sudo tcpdump -ni any udp port 514
```

Record the RouterOS version, CPU load with and without export enabled, and
whether flow byte counts roughly match SNMP interface counters. These results
feed directly into decisions D1 and D2 and into the on-site questions above.

### 8.2 The WireGuard tunnel

```bash
# On the VPS: peer should show an endpoint, a recent handshake and traffic
sudo wg show

# Does the stand-in peer (or the router) answer through the tunnel?
sudo ping -c3 10.100.0.3

# Do telemetry-like packets arrive on the tunnel interface?
sudo tcpdump -ni wg0 udp port 2055

# From the peer side, send a test packet through the tunnel
echo "test" | nc -u -w1 10.100.0.1 2055
```

If `ping` fails with "Operation not permitted" (`cap_net_raw`), it is not a
tunnel problem: fix the permissions of `ping` (see 4.3). A
`Destination Host Unreachable` with `Destination address required` means the
VPS does not know the peer's endpoint yet: the peer must call the VPS first.

## 9. Open items to verify

- Whether Telegraf's netflow input is available and sufficient on the installed
  version (would let D1 merge into D2/D3).
- Syslog format sent by the router versus what each receiver expects.
- Effect of accelerated forwarding on flow accuracy.
- Whether exported flows show pre-NAT or post-NAT addresses.
- Current state of the chosen libvirt provider version for Terraform or OpenTofu
  (D10, deferred).
- Reachability of the dish management interface from the LAN.
- The WireGuard configuration on the real MikroTik: the tunnel was validated
  with a laptop in place of the router, but not yet with the main router
  itself (`input` rule, address `10.100.0.2`).
- Where the collector runs (on the VPS or on another tunnel peer), since this
  decides which address the services must listen on.
- The WireGuard key rotation policy and the associated Ansible role.
- The RouterOS version on the RB1100AHx4: if it is RouterOS 6, WireGuard is not
  available and a choice must be made between upgrading the router and a local
  Linux host carrying the tunnel (see 4.12).
