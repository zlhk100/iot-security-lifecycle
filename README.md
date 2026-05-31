<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# IoT Security Lifecycle

**From silicon root-of-trust to fleet decommission: a practitioner's
guide to securing connected devices.**

A structured technical series covering the full arc of connected
device security — hardware trust anchors, boot chain integrity,
TEE architecture, secure provisioning, OTA lifecycle, threat
modelling, and the emerging edge AI security stack.

---

## This is Layer 3 of a three-layer series

This repository completes a body of work built across three layers.
Each layer stands alone. Together they form a complete architecture.

| Layer | What it covers | Where |
|---|---|---|
| **Layer 1 — Mechanisms** | How isolation works: SAU, MPU, MPC, PPC, TrustZone, IO-SMMU, TZASC, Qualcomm VMIDMT/XPU, NXP RDC, MPAM — unified under one pattern | [Article][layer1-post] · [Repo][layer1-repo] |
| **Layer 2 — Attestation** | How trust is established and proven: TPM 2.0, DICE, remote attestation, device identity at scale | [Medium series][layer2-medium] |
| **Layer 3 — Lifecycle** | How security holds across the device's full life: provisioning, OTA, regulation, threat modelling | **This repository** |

> **The unifying insight across all three layers:**
> Every security property — isolation, attestation, lifecycle
> integrity — is an answer to the same question:
> *(initiator identity) × (target resource) → (access policy)*
>
> Layer 1 implements this at the hardware bus level.
> Layer 2 proves it holds at boot and runtime.
> Layer 3 maintains it across the device's operational life.

---

## Who this is for

Engineers and architects working on:

- **Embedded and IoT platforms** targeting PSA certification,
  CRA compliance, or sector-specific security standards
- **Automotive systems** under UNECE R155/R156 and ISO/SAE 21434
- **Industrial and robotics platforms** under IEC 62443
- **AIoT and edge AI deployments** where inference security and
  attestation are new engineering requirements
- **Anyone building a connected device** that will be evaluated
  by a customer, regulator, or certification lab

This series assumes you are comfortable reading C and ARM
architecture documents. It does not assume prior security
certification experience.

---

## Series contents

### [Part 0 — Why This Matters](docs/part0-why-this-matters.md)
The physical-world consequences of connected device compromise,
the regulatory obligations now in force, and Zero Trust as the
organising principle for everything that follows.
*Start here if you are new to the series.*

### [Part 1 — Hardware Root of Trust](docs/part1-hardware-root-of-trust.md)
What a hardware RoT must unconditionally guarantee. A three-level
taxonomy of secret protection (software-accessible → hardware-sealed
→ PUF-generated). Secure boot vs measured boot — definitions,
relationship, and two hardware implementations (TPM and DICE).

### Part 2 — Trust Extension Across the Stack *(in progress)*
From the RoT through the boot chain, TEE boundary (TrustZone-A
and TrustZone-M), defense-in-depth hierarchy, and remote
attestation. How the isolation mechanisms of Layer 1 are
orchestrated into a complete trust architecture.

### Part 3 — Secure Provisioning and Key Hierarchy *(planned)*
How secrets reach the device securely. Key derivation hierarchy
(HUK → SSK → TSK → FEK). PKI infrastructure and signing
certificate taxonomy. Discrete TPM threat model and bus security.
Decentralized identity at scale as a CA alternative.

### Part 4 — Secure OTA and Fleet Lifecycle *(planned)*
TUF design principles. Uptane Director/Image repository split
and the online/offline key rationale. Anti-rollback under UNECE
R156. SBOM and CVE pipeline (SPDX/CycloneDX/Syft). Fail-safe
A/B recovery.

### Part 5 — Threat Modelling Methodology *(planned)*
Three-class attack taxonomy (remote, local, physical) with
countermeasure mapping. STRIDE applied to embedded systems.
Asset table methodology. Crypto agility and algorithm lifecycle.

### Part 6 — Case Studies *(planned)*
Three platform contexts examined through the series framework:
mobile platform defense-in-depth, automotive V2X UNECE R155
compliance, and industrial OT Zero Trust augmentation.

### Part 7 — Extending Trust to the Edge AI Stack *(planned)*
The emerging threat model for AIoT: model weight protection,
inference result integrity, multi-tenant edge platforms. SPDM
for ECU-to-ECU and accelerator-to-host attestation. Where TEE
ends and CCA begins.

### [Appendix A — Regulatory Quick Reference](docs/appendix-a-regulatory-reference.md) *(planned)*
One-table summary: regulation → scope → key engineering
requirement → relevant series section.

### [Appendix B — UNECE R155 Annex 5 Practitioner Mapping](docs/appendix-b-r155-annex5.md) *(planned)*
The full R155 Annex 5 threat/mitigation table annotated with
engineering implementation notes. Standalone reference for
automotive BSP engineers.

---

## Relationship to companion repositories

**[hw-security-safety-isolation][layer1-repo]** (Layer 1) covers
the isolation mechanism layer in depth — the unified
Identity × Resource → Policy framework, the initiator/target
distinction, the lock pattern, and a unified lookup table across
TrustZone, SMMU, MPU, MPC, TZASC, Qualcomm VMIDMT/XPU, NXP RDC,
Xilinx XMPU, AMD TMR, and ARM MPAM. The present series references
those mechanisms but does not re-derive them. Read Layer 1 for
the mechanism detail; read this series for the threat model,
regulatory context, and lifecycle engineering that motivates
why those mechanisms must be correctly configured.

---

## Status

| Part | Status | Word count |
|---|---|---|
| Part 0 — Why This Matters | ✅ Complete | ~1,400 |
| Part 1 — Hardware Root of Trust | ✅ Complete | ~2,700 |
| Part 2 — Trust Extension | 🔄 In progress | — |
| Parts 3–7, Appendices | 📋 Planned | — |

---

## Contributing

Corrections, additions, and platform-specific notes are welcome.

Before opening a pull request:
- Technical claims require a cited primary source (standard,
  specification, or peer-reviewed publication)
- Platform-specific content should be clearly scoped to the
  platform it describes
- No vendor-confidential or NDA-covered material

See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

---

## Licence

Content: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
— © 2026 Lei Zhou

You are free to share and adapt the material for any purpose,
provided attribution is given.

---

*This repository is part of an open knowledge initiative on
embedded security architecture. Companion work:*
*[hw-security-safety-isolation][layer1-repo] ·*
*[Medium][layer2-medium]*

[layer1-post]: https://medium.com/@zlhk100/security-and-safety-by-spatial-and-temporal-isolation-design-pattern-behind-tz-smmu-mpu-and-mpam-f1be9b4da33d
[layer1-repo]: https://github.com/zlhk100/hw-security-safety-isolation
[layer2-medium]: https://medium.com/@zlhk100
