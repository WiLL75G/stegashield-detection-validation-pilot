
# Day 01/07 — Investigation Notes

**Project:** StegaShield Detection Validation Pilot  
**Investigation:** Environment Baseline and Telemetry Validation  
**Date:** November 5, 2026  
**Environment:** Authorized, isolated home laboratory  
**Status:** Core infrastructure verified; Sysmon network telemetry gap resolved  
**Evidence source:** Fresh observations from the current laboratory

---

## 1. Investigation Objective

The purpose of this investigation was to establish whether the laboratory could produce, collect, forward, and search the telemetry needed for subsequent StegaShield validation experiments.

The focus was not on detecting steganography.

Before evaluating a detection engine, I needed to confirm that the supporting infrastructure was functional and understand where visibility gaps existed.

The investigation addressed five questions:

1. Could the laboratory systems communicate over the required network paths?
2. Was Splunk operational and receiving endpoint telemetry?
3. Was Sysmon generating both process and network connection events?
4. Could Zeek record actual network connections and HTTP activity?
5. Were any telemetry gaps present that could affect later SOC investigations?

The investigation followed a baseline-first approach.

No StegaShield detection result or image-transfer outcome was expected during this stage.

---

## 2. Laboratory Context

The environment consisted of three systems.

| System | Role | Relevant Components |
|---|---|---|
| Mac host | Central SOC investigation workstation | Splunk Enterprise |
| Windows VM | Controlled endpoint | Windows 11 ARM64, Sysmon, Splunk Universal Forwarder |
| Ubuntu VM | Network monitoring and future image-analysis system | Ubuntu 24.04 ARM64, Zeek, Docker |

All systems belonged to the newly built laboratory.

The previous laboratory was retired. Its configurations, samples, and findings were not treated as evidence for the current investigation.

### Important Environmental Considerations

The laboratory runs on Apple Silicon hardware.

The Windows and Ubuntu virtual machines use ARM64 environments.

This matters because software compatibility cannot automatically be assumed from successful installations on x86-64 systems.

Docker was verified as operational, but compatibility with the StegaShield deployment image had not yet been established.

The laboratory also contained pre-existing services and configurations that could produce background network traffic.

These conditions were considered when interpreting network records.

---

## 3. Master Investigation Log

| ID | Investigation Area | Initial Observation | Action or Verification | Result | Disposition |
|---|---|---|---|---|---|
| D01-01 | Splunk service | Service availability required confirmation | Inspected listening service | Splunk management listener observed | Verified |
| D01-02 | Ubuntu connectivity | Cross-system access required validation | Performed HTTPS management request | HTTP 200 response from Splunkd | Verified |
| D01-03 | Windows connectivity | Endpoint communication required validation | Performed controlled network tests | Required tested paths reachable | Verified |
| D01-04 | Docker | Container runtime state unknown | Inspected version, service, and containers | Docker active; no containers observed | Verified |
| D01-05 | Zeek | Sensor initially stopped | Inspected configuration, started service, checked status | Standalone instance running | Verified |
| D01-06 | Zeek connections | Logging capability required validation | Inspected connection log | Actual TCP, UDP, and ICMP records | Verified |
| D01-07 | Zeek HTTP | HTTP parsing required validation | Inspected HTTP log | HTTP GET transaction with 204 response | Verified |
| D01-08 | Sysmon | Process events present; network events missing | Inspected service, driver, schema, and configuration | Event ID 3 absent despite enabled setting | Finding identified |
| D01-09 | Sysmon configuration | Existing configuration did not produce observed Event ID 3 | Applied minimal validated XML configuration | Configuration update successful | Remediated |
| D01-10 | Local network telemetry | Event ID 3 required verification | Generated controlled TCP connection and searched local log | Event ID 3 observed | Verified |
| D01-11 | Splunk forwarding | Sysmon channel subscription initially denied | Reviewed service access and event channel permissions | Sysmon ingestion restored | Resolved |
| D01-12 | End-to-end telemetry | Network events needed centralized verification | Searched Splunk for Event ID 3 | Network connection records found | Verified |
| D01-13 | Final coverage | Existing process events needed preservation | Aggregated Sysmon events in Splunk | Events 1, 3, 5, and 16 present | Verified |

The investigation log summarizes observed work. It does not imply that all checks occurred in the order shown above.

---

## 4. Investigation A — Infrastructure and Connectivity

### Initial Question

Could the existing laboratory systems communicate reliably enough to support endpoint logging, network monitoring, and later image-transfer experiments?

### Expected Behavior

The Windows endpoint and Ubuntu server should communicate over their configured virtual network.

The Splunk management service should be reachable from authorized laboratory systems.

A successful network test should demonstrate connectivity to the specified destination and service.

### Actions Performed

I inspected system networking and performed controlled connectivity tests.

The Windows endpoint successfully communicated with Ubuntu.

Ubuntu successfully communicated with the Mac's virtual network gateway.

I also performed an HTTPS request from Ubuntu to the Splunk management endpoint.

The response was:

```text
HTTP/1.1 200 OK
Server: Splunkd
```

A separate Windows TCP connectivity test confirmed successful access to the Splunk management port.

### Analysis

The results established that the tested paths were reachable.

The HTTP response also demonstrated that the destination service was responding at the application layer.

However, successful communication with the Splunk management endpoint does not establish event ingestion.

The management service and the event receiving pipeline serve different functions and require separate verification.

### Disposition

**Verified:** Basic tested network paths and Splunk management accessibility.

**Not established:** End-to-end image transfer, TLS certificate trust, or StegaShield integration.

---

## 5. Investigation B — Docker and Zeek Readiness

### Initial Question

Could Ubuntu support network monitoring and the later controlled image-analysis environment?

### Docker Findings

Docker was installed and its service was active.

The container inventory showed no running or stopped containers at the time checked.

This confirmed that the container runtime was available.

It did not establish that the StegaShield image could run on the ARM64 system.

### Zeek Initial State

Zeek version 9.0.1 was installed.

The executable was located under the Zeek installation directory rather than the shell's default command search path.

The standalone configuration specified:

```ini
[zeek]
type=standalone
host=localhost
interface=enp0s1
```

The configured interface was verified as the intended Ubuntu monitoring interface.

Zeek was initially stopped.

### Action Performed

The standalone Zeek service was started using ZeekControl.

The subsequent status check reported the Zeek node as running.

### Connection Log Verification

I inspected the current connection log.

The log contained actual connection records, including:

- TCP sessions with normal termination
- Rejected TCP attempts
- UDP traffic
- DHCP-related activity
- ICMP traffic

Some records involved ports commonly associated with existing security monitoring and log-forwarding services.

However, port numbers alone were insufficient to establish which application produced every connection.

The records were treated as network observations, not confirmed application attribution.

### HTTP Log Verification

I inspected the current HTTP log.

One observed record contained the following fields:

| Field | Value |
|---|---|
| Method | GET |
| Host | `connectivity-check.ubuntu.com` |
| URI | `/` |
| Destination port | 80 |
| Response code | 204 |
| Response message | No Content |

This was consistent with routine Ubuntu connectivity-check activity.

The event was not evidence of image transfer, steganography, or malicious behavior.

### Interpretation

Zeek was confirmed to be operating and generating useful network metadata.

The existence of network connection records demonstrated visibility into traffic observed on its configured interface.

It did not prove visibility into all communication between other laboratory machines.

Similarly, HTTP logging did not demonstrate an ability to inspect encrypted HTTPS image content.

### Disposition

**Verified:** Zeek standalone operation, connection logging, and HTTP metadata collection.

**Not verified:** Comprehensive virtual network visibility, TLS decryption, or Zeek event ingestion into Splunk.

---

## 6. Investigation C — Missing Sysmon Event ID 3

**Finding ID:** D01-F01  
**Severity:** Laboratory telemetry reliability issue  
**Affected System:** Windows endpoint  
**Relevant Event:** Sysmon Event ID 3 — Network connection detected

### Problem Statement

Sysmon was installed and running on the Windows endpoint.

The local Operational log contained process creation and process termination events.

However, network connection Event ID 3 was not observed.

This was a significant visibility gap for the pilot.

The later SOC investigation requires the ability to associate Windows processes with controlled network activity.

Without reliable network connection events, the endpoint evidence available for correlation would be reduced.

### Initial Evidence

The initial Sysmon configuration query reported:

```text
Network connection: enabled
HashingAlgorithms: SHA256
DNS lookup: enabled

No rules installed
```

Therefore, network monitoring was reported as enabled even though no Event ID 3 records were present.

### Hypothesis 1 — Network Monitoring Disabled

**Reasoning:**

Sysmon Event ID 3 collection may be disabled unless explicitly enabled.

**Test:**

Inspect the active Sysmon configuration.

**Result:**

Network monitoring was reported as enabled.

**Conclusion:**

The available configuration did not support the hypothesis that network monitoring was simply disabled.

### Hypothesis 2 — No Relevant Network Activity

**Reasoning:**

An event may be absent if no qualifying network connection occurs.

**Test:**

Generate a controlled TCP connection to an existing laboratory service.

**Result:**

The connection succeeded.

The subsequent local Event ID 3 search returned no matching records.

**Conclusion:**

There was successful network activity, but the expected corresponding Sysmon telemetry was not observed.

The test narrowed the problem to event generation or collection visibility rather than basic connectivity.

### Hypothesis 3 — Sysmon Service or Driver Failure

**Reasoning:**

A stopped or missing driver could explain missing telemetry.

**Tests:**

- Inspect Sysmon service state
- Inspect kernel driver state
- Inspect registered driver configuration
- Verify driver binary presence
- Compare executable and driver versions
- Verify driver digital signature

**Results:**

The Sysmon kernel driver was running.

The registered driver binary existed.

The driver and executable reported version 15.22.

Windows reported the driver signature as valid.

The service query did not report an exit error.

**Conclusion:**

The investigation found no evidence that a missing or stopped driver caused the telemetry gap.

These checks did not prove that every driver capability was functioning correctly.

### Hypothesis 4 — Sysmon Internal Error

**Reasoning:**

Sysmon can report internal problems through error events.

**Test:**

Search the retained Operational log for Event ID 255.

**Result:**

No matching Event ID 255 records were returned.

**Conclusion:**

No retained internal error record was found by that search.

The absence of Event ID 255 did not conclusively rule out an internal collection problem.

### Hypothesis 5 — Configuration Behavior

**Reasoning:**

The existing configuration reported network monitoring as enabled but contained no explicit event filtering rules.

A minimal configuration with explicit event collection rules could help determine whether network-event collection behavior changed.

This was a controlled remediation test rather than proof of a specific root cause.

### Original Configuration Preservation

Before applying the test configuration, I captured and verified the existing Sysmon configuration report.

The report preserved the original settings for comparison.

It was a configuration record, not a directly restorable XML backup.

### Remediation

I created a minimal XML configuration using Sysmon schema version 4.91.

The configuration specified SHA256 hashing and three event categories.

```xml
<Sysmon schemaversion="4.91">
  <HashAlgorithms>sha256</HashAlgorithms>
  <EventFiltering>
    <ProcessCreate onmatch="exclude"/>
    <ProcessTerminate onmatch="exclude"/>
    <NetworkConnect onmatch="exclude"/>
  </EventFiltering>
</Sysmon>
```

The empty exclusion rules were intended to collect the corresponding event categories without adding exclusion conditions.

The XML file was inspected before being applied.

Sysmon subsequently reported:

```text
Configuration file validated.
Configuration updated.
```

A separate active-configuration query confirmed that Sysmon was using the new XML configuration.

### Post-Change Test

After the configuration change, I generated another controlled TCP connection.

The TCP test succeeded.

I then searched the local Sysmon Operational log for recent Event ID 3 records.

The query returned network connection events.

The observed fields included:

- Process GUID
- Process ID
- Process image
- User
- Network protocol
- Source address
- Source port
- Destination address
- Destination port
- Connection initiation status

One observed event associated the Splunk Universal Forwarder process with a connection to the Splunk receiving service.

Other records included browser-related activity.

### Interpretation

Network telemetry became observable after the explicit XML configuration was applied.

The investigation therefore established a successful remediation outcome.

However, it did not isolate the precise internal cause of the original absence.

Potential configuration behavior, service state transitions, and event-generation conditions were not independently separated in a controlled comparison.

### Final Disposition

**Status:** Resolved

**Verified outcome:** Sysmon Event ID 3 generated locally after remediation.

**Root cause:** Not conclusively established.

**Operational lesson:** Enabled telemetry settings must be validated against actual records.

---

## 7. Investigation D — Splunk Forwarding and Event Correlation

**Finding ID:** D01-F02  
**Affected Component:** Windows Splunk Universal Forwarder  
**Issue:** Initial inability to subscribe to the Sysmon Operational event channel

### Initial Observation

The Windows Universal Forwarder was operational and configured to collect Windows event logs.

The Sysmon Operational channel was added to its input configuration.

However, the forwarder's log initially reported a failure to subscribe to that channel.

The reported Windows error code was:

```text
errorCode=5
```

This was consistent with an access-denied condition.

### Investigation

I inspected the service identity and event-channel access configuration.

The forwarder was running under its dedicated Windows service identity.

The local Event Log Readers group did not initially contain the service identity.

### Remediation

The Splunk Universal Forwarder service identity was added to Event Log Readers.

The forwarding service was restarted.

The service returned to the running state.

Subsequent Splunk searches successfully returned Sysmon events.

### Verification

The Sysmon source became searchable in Splunk.

Initial successful ingestion included process-related Sysmon records.

After the separate local Event ID 3 remediation, network connection events also became searchable.

### Important Distinction

Two separate collection layers had been affected:

**Layer 1 — Event Generation**

Sysmon initially did not produce observable Event ID 3 records in the retained local log.

**Layer 2 — Event Forwarding**

The Splunk Universal Forwarder initially lacked the required access to subscribe to the Sysmon channel.

Fixing the forwarding permissions did not automatically resolve local Event ID 3 generation.

Likewise, restoring Event ID 3 locally would not have guaranteed Splunk visibility without a functioning forwarding path.

### Final Ingestion Verification

I performed a Splunk search for Event ID 3.

The query returned network records from the Windows endpoint.

One record associated the Splunk Universal Forwarder with an outbound TCP connection to the Splunk receiving service.

This demonstrated the complete observable collection path:

```text
Windows Network Activity
         |
         v
Sysmon Event ID 3
         |
         v
Windows Event Log
         |
         v
Splunk Universal Forwarder
         |
         v
Splunk Enterprise
         |
         v
Searchable Investigation Evidence
```

### Final Event Coverage

A 30-minute Splunk search returned:

| Event ID | Description | Count |
|---|---|---:|
| 1 | Process creation | 215 |
| 3 | Network connection | 52 |
| 5 | Process termination | 217 |
| 16 | Configuration change | 1 |
| **Total** | | **485** |

The search window is important.

These values represent the records observed within that period, not permanent event counts.

### Disposition

**Forwarding access issue:** Resolved.

**Local Event ID 3 collection issue:** Resolved.

**End-to-end Sysmon telemetry:** Verified for the event types observed in the final Splunk search.

---

## 8. Investigation Conclusions and Evidence Gaps

### Confirmed Findings

The investigation established that:

1. The required tested laboratory network paths were reachable.
2. Splunk Enterprise was operational.
3. Windows event forwarding into Splunk was functioning after remediation.
4. Sysmon process telemetry was available.
5. Sysmon Event ID 3 was initially absent.
6. A validated XML configuration change was followed by successful local Event ID 3 collection.
7. Network events subsequently appeared in Splunk.
8. Zeek recorded actual connection and HTTP metadata.
9. Docker was operational.

### Remaining Evidence Gaps

The investigation did not establish:

- The definitive cause of the initial missing Event ID 3 records
- Complete Zeek visibility into all virtual network paths
- Zeek log forwarding into Splunk
- StegaShield platform compatibility
- StegaShield operational readiness
- StegaShield detection performance
- HTTPS image content visibility
- The ability to correlate actual StegaShield analysis results with endpoint and network evidence

These questions remain outside the verified Day 01 results.

### Final Analytical Assessment

The laboratory has demonstrated the core telemetry capabilities needed to proceed with controlled image dataset preparation.

The most significant Day 01 finding was the difference between an enabled monitoring configuration and observable telemetry.

The Sysmon investigation also showed why endpoint event generation and SIEM ingestion must be tested independently.

The final outcome is supported by actual event records, successful forwarding, and Splunk search results.

No conclusion about image-steganography detection performance can be drawn from this investigation.

---

## 9. Lessons for SOC Analysts

### Lesson 1 — A Running Tool Is Not Proof of Visibility

The Sysmon service and kernel driver were running.

Network monitoring was reported as enabled.

Nevertheless, Event ID 3 was initially absent.

An analyst must verify the events that a monitoring tool actually produces.

### Lesson 2 — Troubleshoot the Evidence Pipeline by Layer

When an event is missing from a SIEM, determine whether the issue is:

1. The underlying activity did not occur
2. The sensor did not generate an event
3. The local event channel did not retain the event
4. The forwarding service could not read the event
5. The event was not delivered or indexed
6. The search failed to find an existing event

The investigation found separate problems affecting event generation and forwarding access.

### Lesson 3 — Configuration Changes Require Before-and-After Evidence

The XML configuration update was not treated as successful simply because Sysmon accepted the file.

Success was established only after observing:

- A controlled connection
- A local network connection event
- The same category of telemetry searchable in Splunk
- Continued process-event ingestion

### Lesson 4 — Do Not Overstate Root Cause

The evidence supports a successful remediation.

It does not conclusively prove why the original configuration failed to produce observable Event ID 3 records.

A defensible investigation distinguishes the observed outcome from the suspected cause.

### Lesson 5 — Network Metadata and Content Inspection Are Different

Zeek demonstrated useful connection and HTTP metadata collection.

That does not mean it can automatically inspect encrypted HTTPS image contents.

The later pilot will require a separate authorized application inspection point for examining received images.

---

## 10. Final Investigation Disposition

**Day:** 01/07  
**Investigation:** Environment Baseline and Telemetry Validation

**Overall technical result:** Core baseline verified.

**Finding D01-F01:** Missing Sysmon Event ID 3 — Resolved.

**Finding D01-F02:** Splunk Universal Forwarder Sysmon channel access denied — Resolved.

**Network visibility limitation:** Documented for later investigation.

**StegaShield testing:** Not performed.

**Detection performance:** Not assessed.

### Readiness Decision

Proceed to Day 02 — Dataset and Ground Truth Preparation.

The current laboratory has verified core Windows endpoint telemetry, centralized Sysmon ingestion, Zeek network logging, and Docker readiness.

Additional product and integration checks remain necessary during their corresponding project stages.

---

## 11. Next Investigation

**Day 02/07 — Dataset and Ground Truth**

The next investigation will establish controlled image samples before submitting any files to StegaShield.

The immediate technical checkpoint is to verify that Python Pillow is available on Ubuntu.

After dependency verification, work will proceed through:

1. Reproducible clean-image generation
2. Original sample preservation
3. File hashing and metadata collection
4. Controlled LSB embedding
5. Independent ground-truth labeling
6. Reproduction and integrity testing
7. Manifest verification

Only evidence produced in the new laboratory will count toward the current pilot.

---

**Investigation status: Core baseline verified.**

**Evidence-first principle: Observed behavior must support every technical conclusion.**
