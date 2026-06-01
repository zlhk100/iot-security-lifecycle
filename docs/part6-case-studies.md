<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# Part 6 — Case Studies

> **Series context:** The same principles produce different
> architectures. Each case study here applies the framework from
> Parts 1–5 to a specific platform context and surfaces the
> engineering decision that a generic guide would not have reached.
> Platforms are identified where the analysis is based on public
> information; others are presented with platform details generalised.

---

## The purpose of case studies in this series

Abstract principles are necessary. They are not sufficient. A threat
model that looks coherent on paper can have a gap that only becomes
visible when you apply it to a specific asset set with specific
operational constraints and a specific attacker.

Each case study here is chosen because it surfaces one non-obvious
engineering decision — a decision that the framework from Parts 1–5
motivates but does not resolve by itself. That decision is the
contribution. The platform description is context.

---

## Case Study 1 — Mobile platform: when TrustZone is not enough

**Platform:** Google Pixel 6 (publicly documented)
**Source:** Google security blog (public) and author's analysis

### The decision the threat model forced

Google Pixel 6 uses ARM TrustZone for logical isolation between the
rich execution environment (Android) and the trusted execution
environment (Trusty OS). This is standard for the class. What is not
standard is what Google added alongside it: the Tensor Security Core,
a physically separate security processor on the same die.

The reason for the addition is explicit in Google's design intent
and follows directly from the three-class attack taxonomy in Part 5:
TrustZone addresses remote attackers and provides logical isolation.
It does not address local attackers who can mount microarchitectural
side-channel attacks — Spectre, Meltdown, and cache timing attacks
that exploit shared execution pipelines between the Secure and
Non-Secure worlds.

The architectural conclusion Google reached: **TrustZone is
insufficient for the local attacker class.** Operations that must
resist local attackers — key storage, biometric matching, signing
operations — require physical pipeline separation, not logical world
separation.

### The non-obvious implication

This decision has a consequence that is rarely made explicit: the
TEE is not the highest-trust component on the device. The Titan M2
HSM is. The Tensor Security Core sits between them in terms of
physical isolation properties.

The trust hierarchy on Pixel 6 is therefore:

```
Titan M2 HSM (highest — physical attack resistance, FIPS 140-2)
  ↑ delegates specific operations to
Tensor Security Core (middle — physical pipeline isolation,
                      side-channel resistance)
  ↑ logical isolation from
TrustZone / Trusty OS (standard — logical world separation)
  ↑ boundary from
Android REE (untrusted for secret operations)
```

Most discussions of mobile security treat TrustZone as the security
boundary. The Pixel 6 architecture makes explicit that TrustZone is
one layer in a three-layer stack, and that the correct layer for any
given operation depends on which class of attacker that operation
must resist.

### The transferable principle

The Pixel 6 architecture is not unique to mobile. The same three-
layer pattern appears in automotive (ECU with dedicated HSM +
TrustZone application processor + REE), in industrial controllers
(secure element + TrustZone + Linux), and in AIoT edge platforms
(dedicated security processor + NPU isolation + application OS).

The transferable principle: **match the hardware tier to the
attacker class, not to the sensitivity of the data alone.** A key
that only remote attackers will attempt to extract can live in a
TEE. A key that local attackers can target needs a physically
separate processor. A key that physical attackers will target needs
a tamper-resistant HSM. Putting everything in the TEE is not
conservative — it is a mismatch between attacker capability and
hardware protection.

---

## Case Study 2 — Automotive V2X TCU: the forensic logging dilemma

**Platform:** Cortex-A55 based 5G-capable Telematics Control Unit
(TCU) with V2X communication, OP-TEE TEE stack
**Source:** Author's design experience, generalised

### The decision the threat model forced

UNECE R155 §7.3.7(c) requires forensic capability to analyse
attempted or successful cyber-attacks. The V2X TCU must log security-
relevant events: authentication failures, certificate validation
errors, anomalous message rates, diagnostic session initiations,
and TEE access events from REE clients.

For vehicles sold in the EU and other GDPR-jurisdiction markets,
a parallel and independent obligation applies: GDPR treats data
linked to a vehicle identification number (VIN) or registration
number as personal data — including technical data — because it is
linkable to the registered owner. Location data generated by a V2X
TCU is among the most sensitive categories. An event log that
records security-relevant events with timestamps on a V2X-connected
vehicle is, in practice, generating timestamped location records
tied to a device that is linkable to a person.

R155 and GDPR are separate legal instruments. R155 does not reference
GDPR, and GDPR does not reference R155. Both apply simultaneously
to the same product in EU-market vehicles. Research on automotive
cybersecurity standards confirms this structural gap: most automotive
cybersecurity standards focus on system integrity and operational
safety, and "cybersecurity compliance alone is insufficient in
jurisdictions with strong data protection laws." The OEM must satisfy
both independently.

These two obligations are structurally in tension. The engineering
question is not which to comply with — both are non-negotiable — but
how to satisfy both with one log architecture.

### The resolution: schema separation at design time

The resolution requires a design decision that must be made at log
schema definition time, not after:

**Security-relevant metadata** — event type, timestamp, ECU
identifier, outcome (success/failure), anomaly score — carries no
PII and has no GDPR retention restriction. It must be kept long
enough for forensic analysis, stored in a tamper-evident log
(append-only, MAC-protected, TEE-accessible for integrity), and
available to the OEM's CSMS for threat analysis per R155 §7.3.7.

**Privacy-sensitive content** — precise geolocation at event time,
V2X pseudonym certificate identifiers that could enable tracking,
user identity associated with biometric authentication events —
falls under GDPR Article 5 data minimisation and Article 17 right
to erasure. It requires explicit user consent for retention beyond
the minimum, and a provably effective erasure mechanism.

The two log streams have different storage locations, different
access controls, different retention periods, and different lifecycle
management requirements. They must be separated at the schema level —
not at the access control level — because an access-controlled
combined log still contains PII that cannot be erased without
destroying the forensic record.

### The non-obvious implication

The "TEE access logging" requirement in the R155 context creates a
data minimisation tension that arises specifically because two
independent regulatory regimes — R155 cybersecurity and GDPR privacy
— apply simultaneously to the same log data. A TEE access log that
records which REE client requested which TA service, at what time,
in conjunction with V2X pseudonym certificate activity, generates
data that is potentially linkable to a physical location and identity.

This is not a conflict that either regulation resolves. R155
§7.3.7(h) explicitly notes that the forensic logging capability
"shall respect the privacy rights of car owners or drivers" — but
provides no engineering guidance on how. GDPR provides the data
protection framework but was not written with V2X forensic logging
in mind. The resolution is an engineering design decision that sits
in the gap between the two instruments.

Designing the schema separation retrospectively — after the log
format is deployed to a fleet — requires a firmware update to change
the log schema, migration of existing logs, and a re-certification
event for R155 compliance documentation. The cost of getting it
wrong at design time is not a debugging session. It is a fleet-wide
OTA update and a regulatory disclosure.

---

## Case Study 3 — Industrial OT: what the Purdue model cannot see

**Platform:** Representative industrial SCADA system with RTU/PLC
field devices, industrial fieldbus (MODBUS, CAN bus, RS-485),
and IT/OT integration gateway
**Source:** Author's published analysis ([SCADA threat model
post][scada-post], April 2023) extended with additional context

[scada-post]: https://medium.com/@zlhk100/scada-systems-threat-model-and-zero-trust-architecture-as-mitigation-c4d2c0c8dd9d

### The Purdue model's structural blind spot

The Purdue model — the conventional SCADA security architecture —
organises industrial control systems into hierarchical zones
separated by firewalls, with an Industrial Demilitarised Zone (IDMZ)
creating a buffer between the OT network and the internet-connected
IT network. The implicit assumption is that attacks originate
outside the perimeter and are stopped at the zone boundary.

This assumption was reasonable when OT systems were air-gapped.
It is structurally wrong against the modern threat landscape. The
attack vectors that produce the most significant OT incidents operate
from inside the protected perimeter:

**USB removable media** (Stuxnet, multiple subsequent campaigns):
an attacker who cannot reach the OT network remotely delivers
malware on a USB drive through a trusted human. The IDMZ sees no
traffic to inspect. The perimeter is crossed by a person carrying
a device.

**Supply chain compromise**: malware embedded in a software update
from a trusted vendor arrives through the legitimate update channel.
The perimeter sees authenticated traffic from a known source. The
attack is indistinguishable from a legitimate update until execution.

**Insider threat and unauthorised maintenance access**: a
contractor with legitimate access to the OT network — authenticated
by the perimeter controls — performs unauthorised actions once
inside. The perimeter provides no protection after authentication.

The Purdue model's enforcement point is at the zone boundary. All
three attack vectors originate or authenticate at or before the
boundary and operate freely inside it.

### Zero Trust as augmentation, not replacement

Zero Trust does not replace the Purdue model's physical zone
separation — that separation still provides meaningful protection
against external attackers who cannot enter the perimeter. Zero
Trust augments it by adding enforcement at the transaction level
inside the perimeter.

The augmentation addresses exactly the three vectors the Purdue
model cannot see:

**For USB and removable media:** device identity verification at
every endpoint before any data is processed. A USB device that
cannot authenticate against a known-good identity policy is not
mounted, regardless of where it is physically inserted. The
enforcement point is at the endpoint, not the perimeter.

**For supply chain and software updates:** cryptographic
verification of every software component before execution,
continuous runtime integrity monitoring (IMA/EVM equivalent for
OT endpoints), and software bill of materials tracking to detect
unexpected component changes. The enforcement is at the code
execution level, not the network boundary.

**For insider and maintenance access:** continuous authentication
and authorisation on every transaction, micro-segmentation that
limits a compromised identity's lateral reach to only the specific
PLC or RTU required for the legitimate task, and full audit logging
of every privileged action with tamper-evident storage. The
enforcement is per-transaction, not per-session.

### The non-obvious engineering constraint

Implementing Zero Trust on legacy OT infrastructure faces a
constraint that IT deployments do not: most RTUs and PLCs running
in industrial facilities were designed for 20–30 year operational
lives, run proprietary real-time operating systems, and have no
cryptographic identity capability. They cannot participate in a
certificate-based authentication model.

The practical architecture is a two-tier approach: Zero Trust
enforcement runs on the gateway or edge device that interfaces with
the legacy field bus, rather than on the PLC itself. The gateway
authenticates to the Zero Trust infrastructure; the PLC trusts only
the gateway over the physical field bus. The security perimeter moves
from the network boundary to the edge gateway, which is a device
that can be updated and can run cryptographic software.

This means the PLC is still trusted by proximity (network segment
and physical access control), but the gateway that controls access
to the PLC is not trusted by default — it must continuously
authenticate. The threat model accepts that a compromised PLC is
reachable by a compromised gateway, and invests the security
engineering in making the gateway difficult to compromise rather
than in making the PLC authenticate.

---

## What the three cases have in common

Each case study reached the same structural conclusion through a
different path:

**The enforcement point must match the attack vector's entry
point.** Perimeter enforcement (Purdue model) fails against attacks
that enter through the perimeter. World-level enforcement
(TrustZone) fails against attacks that share microarchitecture
inside the world. Session-level enforcement (authenticate once,
trust throughout) fails against insider threats that operate
continuously after authentication.

The framework from Part 2 named this as the blind spot principle:
every enforcement mechanism has a gap defined by the transactions it
does not see. The three cases illustrate this principle in three
distinct operational contexts. In each case, the engineering decision
that mattered was not which mechanism to use — it was understanding
where the chosen mechanism's blind spot was, and what additional
layer was needed to address it.

---

*© 2026 Lei Zhou. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
*Part of the [IoT Security Lifecycle](https://github.com/zlhk100/iot-security-lifecycle) series.*
