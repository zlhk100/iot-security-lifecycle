<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# Part 0 — Why This Matters

> **Series context:** This is the motivating frame for the IoT Security
> Lifecycle series. It explains why the engineering disciplines covered
> in Parts 1–7 are no longer optional for connected device products.
> Start here if you are new to the series. Skip to Part 1 if you
> already know why and want the how.

---

## The cost of getting this wrong is no longer abstract

For most of software engineering's history, a security failure meant
data loss, service disruption, or financial fraud. These are serious
harms. They are also recoverable — databases can be restored, accounts
can be reset, fraud can be disputed.

Connected devices operating in the physical world change this
calculus. A compromised brake controller, a manipulated insulin pump,
a falsified grid relay command — these produce consequences that
cannot be rolled back. The harm is physical. The victim may not be
the device owner.

In 2021, Gartner predicted that by 2025, attackers would have
weaponised operational technology environments to successfully harm
or kill humans. That prediction named the trajectory: as OT systems
connect to IP networks, the attack surface of industrial and
infrastructure systems converges with the attack surface of the
internet. We are past that date. The trajectory has not changed
direction.

This is not an argument for fear. It is an argument for engineering
discipline — and for understanding exactly which engineering decisions
create a defensible connected device and which ones merely look like
security while leaving the door open.

---

## The numbers behind the problem

Three statistics frame the scale:

**~70% of serious security vulnerabilities in large software projects
are memory safety bugs.** Microsoft, Google Chromium, and Mozilla
have all independently reported this figure from their own CVE
analysis. The root cause is the same in every case: C and C++ give
the programmer direct control over memory without enforcing bounds,
lifetime, or aliasing rules. The bugs this enables — buffer overflows,
use-after-free, out-of-bounds reads — have been understood for
decades and continue to dominate vulnerability counts regardless.
The embedded and IoT space runs predominantly on C. The implication
is not that C must be abandoned immediately, but that any security
architecture that relies solely on software correctness as its defence
is betting against a four-decade track record.

**The attack surface of a connected device grows with every interface
it exposes.** A device with a UART debug port, an OTA update channel,
a Bluetooth pairing interface, a cloud telemetry endpoint, and a
USB firmware recovery path has five independent attack surfaces —
each of which must be hardened, and any one of which, if neglected,
becomes the path of least resistance. The security of the system
is determined by its weakest interface, not its strongest one.

**A single compromised device is a template, not an endpoint.** When
an attacker breaks one device in a fleet of a million, the question
is not whether the attack affects one device. The question is whether
the secret material on that device — the hardware unique key, the
provisioned credentials, the class signing key — is unique to that
device or shared across the fleet. A class attack, where one device's
compromise is a template that applies to every device in the same
product line, is the defining threat model for IoT at scale. The
architectural decisions in Part 1 and Part 3 of this series exist
specifically to close that door.

---

## The regulatory frame has changed permanently

Security-by-design for connected products has moved from best
practice to legal obligation across major markets.

The **EU Cyber Resilience Act (CRA)** — Regulation 2024/2847, in
force from December 2024 — establishes binding security requirements
for products with digital elements sold in the EU market. It covers
vulnerability handling obligations, secure update mechanisms, and
security design requirements across the product lifecycle. Non-
compliance blocks market access, not just certification.

**UNECE R155 and R156** — in force for new vehicle type approvals
since July 2022 and applied to all vehicles produced from July 2024
— establish cybersecurity management (CSMS) and software update
management (SUMS) requirements for road vehicles globally. They apply
to every OEM selling vehicles in adopting markets, and by extension
to every Tier 1 and Tier 2 supplier whose components are in scope.

Sector-specific obligations layer on top: **IEC 62443** for industrial
automation and control systems, **HIPAA** for medical devices handling
patient data, **ISO/SAE 21434** for automotive cybersecurity
engineering, **FIPS 140-2/3** for cryptographic modules in US federal
procurement. In each case, the regulatory obligation is a floor, not
a ceiling — and meeting the floor requires understanding what the
floor actually requires at the engineering level, not just at the
compliance documentation level.

The practical consequence for an embedded systems engineer or
architect is this: the security decisions made during platform design
— which RoT architecture to use, how secrets are provisioned and
stored, how firmware updates are authenticated, how isolation is
enforced between components — are now subject to evaluation by a
third party. They must be defensible, documented, and correct.

---

## Zero Trust as the organising principle

Across all of these contexts — mobile, automotive, industrial, IoT,
AIoT — one architectural principle appears consistently as the
organising framework for how to reason about security decisions:
Zero Trust.

Zero Trust is not a product category. It is a design philosophy with
four core axioms:

**No implicit trust.** Every access request is evaluated explicitly,
regardless of whether the requestor is inside the network perimeter,
inside the device boundary, or inside the trusted execution
environment. Location is not evidence of legitimacy.

**Always verify.** Authentication is not a one-time gate at the
entry point. Every significant transaction — a firmware update, a
key derivation, a privileged API call, a cloud service request —
is authenticated at the point of the transaction, not assumed to be
authorised because an earlier step succeeded.

**Least privilege.** Every component — every process, every partition,
every bus master, every user — is granted only the access necessary
to perform its function and no more. The blast radius of a compromise
is bounded by the privilege granted to the compromised component.

**Assume breach.** Security design does not optimise for the case
where no component is compromised. It optimises for the case where
one component is compromised and asks: what can the attacker reach
from there? The answer to that question is the effective security
posture of the system.

These four axioms translate directly into the engineering decisions
this series covers. The RoT taxonomy in Part 1 is an answer to "no
implicit trust" applied to secret storage. The isolation mechanisms
in Part 2 are an answer to "least privilege" applied to hardware
access control. The attestation architecture in Part 2 is an answer
to "always verify" applied to device identity. The OTA security
model in Part 4 is an answer to "assume breach" applied to the
software update channel.

Zero Trust does not tell you which mechanisms to use. It tells you
what properties those mechanisms must provide, and why the properties
matter. The rest of this series provides the mechanisms.

---

## How this series is organised

The series follows a connected device through its full security
lifecycle — from the hardware decisions made at silicon design time
through to fleet management, decommission, and the emerging threat
model of edge AI deployments.

```
Part 1 — Hardware Root of Trust
         What must be unconditionally trusted, and how secrets are
         protected at different assurance levels.

Part 2 — Trust Extension Across the Stack
         From the RoT through the boot chain, TEE boundary,
         and operating system to remote attestation.

Part 3 — Secure Provisioning and Key Hierarchy
         How secrets reach the device securely, and how the key
         hierarchy is structured from device birth onward.

Part 4 — Secure OTA and Fleet Lifecycle
         Authenticated update delivery, anti-rollback, SBOM,
         and fail-safe recovery.

Part 5 — Threat Modelling Methodology
         How to reason systematically about what can go wrong,
         illustrated with worked case studies.

Part 6 — Case Studies
         Three real platform contexts — mobile, automotive, and
         industrial OT — examined through the lens of the series.

Part 7 — Extending Trust to the Edge AI Stack
         The emerging threat model for AIoT: model weight
         protection, inference result integrity, and SPDM-based
         attestation for accelerator-to-host trust.
```

**Companion repositories:**

This series is Layer 3 of a three-layer body of work:

- **Layer 1 — Isolation mechanisms** (the how):
  [Security and Safety by Spatial and Temporal Isolation][layer1-post]
  / [hw-security-safety-isolation][layer1-repo]

- **Layer 2 — Attestation and trust establishment** (the proof):
  [Attestation and TPM series][layer2-medium] (Medium, 2024–2025)

- **Layer 3 — Lifecycle and regulation** (the full picture):
  This series

Each layer stands alone. Together they form a complete architecture.

[layer1-post]: https://medium.com/@zlhk100/security-and-safety-by-spatial-and-temporal-isolation-design-pattern-behind-tz-smmu-mpu-and-mpam-f1be9b4da33d
[layer1-repo]: https://github.com/zlhk100/hw-security-safety-isolation
[layer2-medium]: https://medium.com/@zlhk100

---

*© 2026 Lei Zhou. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
*Part of the [IoT Security Lifecycle](https://github.com/zlhk100/iot-security-lifecycle) series.*
