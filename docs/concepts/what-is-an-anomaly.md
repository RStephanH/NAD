# What is a network anomaly?

**English** | [Français](what-is-an-anomaly.fr.md)

This document defines what NAD means by "anomaly", so that its results can be
read and judged correctly.

## In plain words

A network anomaly is **behavior that does not look like what the network
normally does**. It is not necessarily an attack, or even a problem. It is a
deviation.

Think of a guard who knows the rhythm of a building. A door opening at 2 p.m.
on a Tuesday is unremarkable; the same door opening at 3 a.m. on a Sunday is
not. NAD plays that role: it learns the normal rhythm, then reports what falls
outside it.

## Three shapes of anomaly

| Shape | Plain description | Example |
|---|---|---|
| **Point anomaly** | A single extreme value | Throughput ten times higher than usual for one minute |
| **Contextual anomaly** | A value that is ordinary in itself but unusual *at that moment* | Heavy traffic at 3 a.m. on a Sunday, which would be normal on a Monday morning |
| **Collective anomaly** | Each minute looks normal alone, but the sequence is strange | A machine contacting a new destination every 60 seconds exactly |

Contextual anomalies are the reason NAD learns weekly rhythms (per hour of the
week) rather than relying on fixed thresholds.

## What can cause one

| Origin | Example | What NAD sees |
|---|---|---|
| Fault or degradation | Satellite uplink losing packets, overloaded router | Rising latency and packet loss, high CPU |
| Misconfiguration | Network loop, a service retrying endlessly | Sudden explosion in flow or packet counts |
| Unusual but legitimate use | Large update, backup, video streaming | High upload or download volume |
| Security event | Port scan, connection flood, compromised machine, data leaving the network | Many distinct ports or destinations, very high flow counts, abnormal upload volume |
| New and legitimate event | New device, new application | A false alarm to be reviewed |

## What NAD says, and what it does not

NAD answers **"is this unusual?"**, not **"is this malicious?"**. An alert is a
starting point for a human investigation.

To support that investigation, each anomaly event records the features that
contributed most to it. An event can then be read as, for example, "flagged at
14:32 because the number of distinct destination ports rose forty-fold", which
points toward a scan rather than a fault.

## Limits to keep in mind

- **Whatever was present during the learning period becomes "normal".** If the
  network was misbehaving while the baseline was collected, that behavior is
  learned as a habit.
- **Quiet attacks can stay under the threshold.** Activity that remains within
  usual values, such as a very slow data leak, may not stand out.
- **Packet contents are not inspected.** NAD only sees metadata (counters,
  flows, latency). This is deliberate and outside the scope of the project.
- **It does not replace signature-based detection.** Rule-based intrusion
  detection systems (for example Suricata) recognize known patterns; NAD
  reports deviations even when no rule exists. The two approaches complement
  each other.

## Working definition

> An anomaly is a time window whose feature vector (traffic, diversity, link
> quality, device health) deviates from the learned baseline for its time-of-week
> context by more than a configurable threshold.

This definition makes the subject measurable, which is what validation relies
on.

## How this maps to validation

Validation injects known anomalies in an authorized window and checks that they
are detected. Each one illustrates a category above.

| Injected anomaly | Tool | Category illustrated | Expected signal |
|---|---|---|---|
| Bulk transfer | iperf3 | Unusual but legitimate use | Volume sharply up |
| Port scan | nmap | Security event | Distinct ports and destinations up |
| Connection flood | hping3 | Security event, misconfiguration | Flow and packet counts exploding |

Uplink outages cannot be injected, but they occur naturally and can be observed.
They are used to check that link problems are told apart from anomalies inside
the network, thanks to link-quality features kept separate from the others.

## Related documents

- [README](../../README.md): project overview and architecture
- [Tooling comparison](../tools/tooling.md): tools and decisions
- [Guide 01](../guides/01-telemetry-and-ingestion.md): telemetry setup and
  ingestion design
