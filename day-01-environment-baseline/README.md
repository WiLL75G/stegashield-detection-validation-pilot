
# Day 01/07 — Environment Baseline

**StegaShield Detection Validation Pilot**

**Investigation:** Laboratory Readiness, Network Monitoring, Endpoint Telemetry and Sysmon Event ID 3 Recovery

**Date:** November 5, 2026

**Technical Status:** Core baseline verified. Evidence publication pending.

---

## 1. Objective

The objective of Day 01 was to establish a reliable laboratory baseline before conducting StegaShield detection experiments.

I wanted to verify that the laboratory systems could communicate, that the monitoring services were operational, and that endpoint and network telemetry was being collected correctly.

The investigation focused on:

- Laboratory connectivity
- Splunk Enterprise availability
- Windows event forwarding
- Sysmon process and network telemetry
- Zeek network and HTTP logging
- Docker readiness
- Identification and investigation of telemetry gaps

The purpose was not simply to confirm that the tools were installed.

I needed evidence that the tools were actually producing data that could support a SOC investigation.

---

## 2. Laboratory Environment

The pilot uses a new laboratory running on an Apple Silicon Mac with Windows and Ubuntu virtual machines.

| Component | Technology | Purpose |
|---|---|---|
| Analyst workstation | macOS, Splunk Enterprise 10.6.1.1 | Centralized monitoring and investigation |
| Windows endpoint | Windows 11 ARM64 | Controlled endpoint activity |
| Endpoint monitoring | Sysmon 15.22 | Process and network connection telemetry |
| Log forwarding | Splunk Universal Forwarder | Windows event ingestion |
| Linux analysis system | Ubuntu 24.04 LTS ARM64 | Network monitoring and future image analysis |
| Network sensor | Zeek 9.0.1 | Connection and HTTP metadata |
| Container runtime | Docker | Future StegaShield deployment |

All evidence in this investigation comes from the new laboratory.

No results or screenshots from the retired environment are reused.

---

## 3. Practical Work Performed

### 3.1 Splunk Enterprise Availability

I first verified that Splunk Enterprise was running on the Mac host.

Using a terminal command, I inspected the TCP listening state of the Splunk management service.

The output confirmed that the Splunk daemon was listening on its management port.

![Splunk Enterprise Listening Service](evidence/01-splunk-service.png)

**Observed:** Splunk Enterprise had an active listening service.

**Not verified by this check alone:** End-to-end event ingestion from the Windows endpoint.

### 3.2 Ubuntu Connectivity to Splunk

From Ubuntu, I performed a controlled HTTPS request to the Splunk management endpoint.

The request returned an HTTP 200 OK response.

The response headers also identified the responding service as Splunkd.

![Ubuntu to Splunk Connectivity](evidence/02-splunk-connectivity.png)

**Observed:** Ubuntu could reach the Splunk management endpoint.

**Interpretation:** The tested network path and Splunk management service were accessible.

**Limitation:** This did not establish Ubuntu log forwarding or StegaShield integration.

### 3.3 Docker Readiness

I checked the existing Docker installation on Ubuntu.

The Docker service was active, and the container inventory showed no deployed containers at the time of inspection.

This established basic container-runtime readiness.

**Observed:** Docker was installed and operational.

**Not verified:** StegaShield compatibility, deployment or API availability.

---

## 4. Zeek Network Monitoring Baseline

### 4.1 Zeek Configuration and Running State

I inspected Zeek's standalone configuration.

The configuration identified the monitored Ubuntu interface.

I started Zeek and checked its operational status using ZeekControl.

The status output confirmed that the standalone Zeek instance was running.

![Zeek Running Status](evidence/03-zeek-status.png)

**Observed:** Zeek was running in standalone mode.

**Interpretation:** The sensor was available for collecting network activity visible on its configured interface.

### 4.2 Connection Logging

I inspected Zeek's connection log.

The output contained actual network records, not only log headers.

Observed activity included:

- TCP sessions
- Rejected TCP connections
- UDP traffic
- DHCP activity
- ICMP traffic

![Zeek Connection Records](evidence/04-zeek-connections.png)

**Observed:** Zeek successfully generated connection metadata.

**Limitation:** This did not prove that Zeek could observe all traffic exchanged between other virtual machines.

### 4.3 HTTP Logging

I inspected Zeek's HTTP log.

The output contained a real HTTP transaction consistent with an Ubuntu connectivity check.

| Field | Observed Value |
|---|---|
| Method | GET |
| Protocol | HTTP |
| Request path | `/` |
| Response code | 204 |
| Response message | No Content |

![Zeek HTTP Request](evidence/05-zeek-http.png)

**Observed:** Zeek successfully recorded HTTP request and response metadata.

**SOC interpretation:** The observed request was consistent with routine connectivity-check activity.

There was no evidence in this record of malicious behavior, image steganography or data exfiltration.

**Limitation:** The HTTP log did not establish visibility into encrypted HTTPS image contents.

---

## 5. Sysmon Network Telemetry Investigation

### 5.1 Initial Observation

I verified that Sysmon was installed and running on the Windows endpoint.

The Sysmon driver was registered, active and reported version 15.22.

Process creation and termination events were present in the local Sysmon Operational log.

However, network connection Event ID 3 was missing.

This was important because the later HTTPS investigation requires the ability to associate network activity with a Windows process.

### 5.2 Investigation Finding

**Finding ID:** D01-F01

**Title:** Missing Sysmon Event ID 3 Network Connection Telemetry

**Expected behavior:**

When network connection monitoring is enabled, Sysmon should generate corresponding events for qualifying network activity.

**Actual behavior:**

Sysmon recorded process activity, but Event ID 3 was absent from the inspected retained log.

A controlled TCP connection succeeded, yet no matching network connection event was found under the original configuration.

This demonstrated a difference between successful network connectivity and actual endpoint telemetry collection.

### 5.3 Inspecting the Active Configuration

I queried Sysmon's active configuration.

The output showed:

| Setting | Initial State |
|---|---|
| Network connection monitoring | Enabled |
| Hashing algorithm | SHA256 |
| DNS lookup | Enabled |
| Event filtering rules | None installed |

![Initial Sysmon Configuration](evidence/06-sysmon-initial-config.png)

**Observed:** Network monitoring was reported as enabled.

**Problem:** Event ID 3 was still missing.

This meant I could not assume that the enabled setting guaranteed successful event collection.

### 5.4 Additional Troubleshooting

Before changing the configuration, I investigated several possible explanations.

I verified:

- Sysmon service and driver state
- Registered driver configuration
- Driver binary presence
- Executable and driver versions
- Driver digital signature
- Supported Sysmon event schema
- Retained Sysmon error events
- Existing network monitoring settings

The driver was running, the binary was present, and Windows reported a valid digital signature.

The Event ID 255 query returned no matching retained records.

These checks did not establish the root cause of the missing network telemetry.

They did, however, provide evidence against a missing or stopped driver as the immediate explanation.

### 5.5 Configuration Remediation

I prepared a minimal Sysmon XML configuration with explicit event collection rules.

The configuration targeted:

- ProcessCreate — Event ID 1
- NetworkConnect — Event ID 3
- ProcessTerminate — Event ID 5

I preserved and verified a report of the original active configuration before making the change.

I then applied the new XML configuration.

Sysmon reported that the configuration file was validated and successfully updated.

![Sysmon Configuration Update](evidence/07-sysmon-config-update.png)

I queried the active configuration again.

The resulting output confirmed that Sysmon was using the new XML configuration and that the three intended event collection rules were present.

**Observed:** The new configuration was successfully applied.

**Not yet established at this point:** Whether Event ID 3 was being generated.

### 5.6 Controlled Network Test

After applying the new configuration, I generated another controlled TCP connection to an existing laboratory service.

The network test succeeded.

I then searched the local Sysmon Operational log for recent Event ID 3 records.

This time, the query returned actual network connection events.

### 5.7 Local Event ID 3 Verification

The newly observed Sysmon records contained:

- Timestamp
- Process GUID
- Process ID
- Executable path
- User context
- Protocol
- Source address and port
- Destination address and port
- Connection initiation status

One event associated the Splunk Universal Forwarder process with an outbound TCP connection to the Splunk receiving service.

Additional browser-related network events were also visible.

![Local Sysmon Event ID 3](evidence/08-sysmon-event3-local.png)

**Observed:** Event ID 3 was now being generated locally.

**Interpretation:** The configuration change was followed by successful network-event collection.

**Limitation:** The exact root cause of the original missing Event ID 3 was not conclusively determined.

---

## 6. Splunk Event Ingestion Verification

### 6.1 Windows Event Forwarding

The Windows Splunk Universal Forwarder was configured to collect Windows event channels, including Sysmon Operational.

During the baseline preparation, the forwarder initially encountered an access-denied error when subscribing to the Sysmon event channel.

The forwarder service identity was added to the local Event Log Readers group, and the service was restarted.

Subsequent Splunk searches confirmed Sysmon ingestion.

This was a separate issue from the missing locally generated Event ID 3 records.

The investigation therefore distinguished between:

1. Local event generation
2. Event channel access
3. Forwarding
4. Centralized ingestion
5. Searchable fields

### 6.2 Event ID 3 in Splunk

After restoring local network telemetry, I searched Splunk for Event ID 3.

The query returned network connection records from the Windows endpoint.

Fields included:

- Host
- Process image
- Source address
- Destination address
- Destination port
- Protocol
- Event timestamp

![Sysmon Event ID 3 in Splunk](evidence/09-splunk-event3.png)

One record linked the Splunk Universal Forwarder process with an outbound TCP connection to the receiving service.

Other records showed browser-related network activity.

**Observed:** Sysmon Event ID 3 was successfully ingested and searchable in Splunk.

This verified the endpoint telemetry path:

**Windows activity → Sysmon → Splunk Universal Forwarder → Splunk Enterprise**

### 6.3 Final Event Coverage

I performed a final Splunk search to confirm that the new network connection telemetry was available alongside existing process telemetry.

The query examined a 30-minute window.

| Event ID | Description | Count |
|---|---|---:|
| 1 | Process creation | 215 |
| 3 | Network connection | 52 |
| 5 | Process termination | 217 |
| 16 | Configuration change | 1 |
| **Total** | | **485** |

![Final Sysmon Event Coverage](evidence/10-splunk-final-events.png)

**Observed:** All four event IDs were available in Splunk.

The counts describe only the observed search window. They are not permanent system totals or detection accuracy measurements.

---

## 7. Investigation Findings

### Finding D01-F01 — Missing Sysmon Network Events

**Initial observation:** Event ID 3 was absent from the retained log.

**Hypothesis:** Network event generation or filtering was not operating as expected.

**Investigation:** Inspected active configuration, driver state, signatures, error records and controlled network activity.

**Action:** Applied a validated XML configuration containing explicit event collection rules.

**Verification:** Event ID 3 appeared locally and in Splunk following the change.

**Disposition:** Resolved.

**Root cause:** Not conclusively established.

### Finding D01-F02 — Initial Sysmon Forwarding Access Denied

**Initial observation:** Splunk Universal Forwarder could not subscribe to the Sysmon Operational channel.

**Investigation:** Examined forwarding logs, service identity and event-channel access permissions.

**Action:** Added the service identity to Event Log Readers and restarted the forwarder.

**Verification:** Sysmon records appeared in Splunk.

**Disposition:** Resolved.

### Finding D01-F03 — Network Visibility Boundaries

**Observation:** Zeek recorded actual network and HTTP metadata.

**Limitation:** Full visibility into all VM traffic and encrypted HTTPS image contents was not demonstrated.

**Disposition:** Documented limitation for the later SOC correlation experiment.

---

## 8. Evidence Register

All evidence must be captured from the current laboratory.

| Evidence ID | Filename | Demonstrates |
|---|---|---|
| D01-E01 | `01-splunk-service.png` | Splunk management listener |
| D01-E02 | `02-splunk-connectivity.png` | Successful Ubuntu connectivity to Splunk |
| D01-E03 | `03-zeek-status.png` | Zeek running state |
| D01-E04 | `04-zeek-connections.png` | Actual Zeek connection records |
| D01-E05 | `05-zeek-http.png` | HTTP request and response metadata |
| D01-E06 | `06-sysmon-initial-config.png` | Original Sysmon network configuration |
| D01-E07 | `07-sysmon-config-update.png` | Validated configuration change |
| D01-E08 | `08-sysmon-event3-local.png` | Local network connection events |
| D01-E09 | `09-splunk-event3.png` | Network telemetry searchable in Splunk |
| D01-E10 | `10-splunk-final-events.png` | Final Event ID coverage |

**Evidence publication status:** Pending selection, sanitization and upload.

The filenames are the intended repository names for the genuine screenshots. They do not imply that the files have already been uploaded.

---

## 9. Observed vs Not Verified

### Verified

- Core laboratory network connectivity
- Splunk Enterprise service accessibility
- Windows event forwarding
- Sysmon process telemetry
- Sysmon Event ID 3 collection after remediation
- Sysmon Event ID 3 ingestion into Splunk
- Sysmon Event IDs 1, 3, 5 and 16 in the final Splunk search
- Zeek running state
- Zeek connection logging
- Zeek HTTP logging
- Docker runtime readiness

### Not Verified

- Definitive root cause of the original Sysmon Event ID 3 issue
- Visibility into every virtual-machine network path
- Ubuntu Zeek log forwarding into Splunk
- Ubuntu Splunk Universal Forwarder integration
- StegaShield deployment and API functionality
- HTTPS image-transfer inspection
- Steganography detection performance
- Detection accuracy or generalization

---

## 10. SOC Relevance

Day 01 demonstrated why SOC analysts must verify telemetry rather than rely on tool status.

Sysmon was installed, its driver was running, and network monitoring was reported as enabled.

Despite those indicators, Event ID 3 was initially missing.

The investigation required separating three questions:

**Was the network connection successful?**

The controlled TCP test confirmed connectivity.

**Did the endpoint generate a network event?**

The initial query found no matching Event ID 3 records. After the configuration change, local Event ID 3 records appeared.

**Could the SIEM search that event?**

The final Splunk searches confirmed ingestion and field availability.

These distinctions are important because a missing alert or event can represent a visibility gap rather than an absence of activity.

---

## 11. Final Outcome

The new laboratory's core infrastructure and telemetry baseline was verified.

The investigation identified a missing Sysmon Event ID 3 collection gap and an earlier event-channel access problem affecting Splunk forwarding.

Both issues were investigated separately.

Following the relevant remediation actions, the final evidence confirmed:

- Working Windows process telemetry
- Working Windows network connection telemetry
- Sysmon event forwarding into Splunk
- Searchable endpoint network evidence
- Operational Zeek connection logging
- Operational Zeek HTTP logging
- Docker runtime readiness

**Sysmon Event ID 3 disposition:** Resolved.

**Root cause confidence:** Inconclusive.

**Day 01 technical baseline:** Core checks verified.

**Day 01 documentation:** Evidence upload pending.

No StegaShield model predictions, image-transfer detections or detection accuracy results were established during this stage.

---

## 12. Lessons Learned

The most important lesson from Day 01 was that an enabled security feature does not guarantee usable telemetry.

At first, I could verify that Sysmon was running and network monitoring was enabled, but the expected network event was missing.

The investigation only became conclusive when I tested the complete evidence path, from actual network activity to locally generated events and then to Splunk.

This reinforced a practical SOC principle:

**A detection or investigation is only as reliable as the telemetry supporting it.**

---

## 13. Next Investigation

**Day 02/07 — Dataset and Ground Truth**

The next stage focuses on establishing independent ground truth for image steganography validation.

Planned work includes:

- Verify Python and Pillow availability
- Establish clean image provenance
- Create reproducible clean control samples
- Generate genuine LSB-modified carriers
- Record image dimensions, modes and hashes
- Document payloads and embedding methods
- Establish independent ground-truth labels
- Verify sample integrity and reproducibility

No model classifications will be claimed until StegaShield is deployed and tested.

---

**Day 01 — Core Technical Baseline Verified | Evidence Publication Pending**

[Return to Master Project README](../README.md)
