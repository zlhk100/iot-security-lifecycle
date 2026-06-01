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

### [Part 2 — Trust Extension Across the Stack](docs/part2-trust-extension.md)
The blind spot principle: every isolation mechanism has a gap
defined by the transactions it never sees. DMA bypass, the
initiator/target distinction, the boot chain split at the world
boundary, TEE architecture and its limits, and attestation.

### [Part 3 — Secure Provisioning and Key Hierarchy](docs/part3-secure-provisioning.md)
The factory floor as the highest-risk moment in the device's life.
Three-phase provisioning threat model. HUK → SSK → TSK → FEK key
derivation hierarchy. PKI infrastructure. Zero-touch onboarding.
Non-securable NVM and cryptographic compensation.

### [Part 4 — Secure OTA and Fleet Lifecycle](docs/part4-secure-ota.md)
OTA as a controlled vulnerability. The atomicity problem and the
confirmation window design decision. Uptane key hierarchy and the
offline image key question resolved. Anti-rollback edge cases.
SBOM as a response capability.

### [Part 5 — Threat Modelling Methodology](docs/part5-threat-modelling.md)
Asset table methodology with the DIU (Data in Use) column.
Three-class attacker model mapped to assets. STRIDE applied to
trust boundary crossings, not components. Crypto agility as a
threat model dimension with PQC timeline.

### [Part 6 — Case Studies](docs/part6-case-studies.md)
Three platforms, three non-obvious engineering decisions: (1) Pixel
6 — why TrustZone alone was insufficient and what it forced; (2)
Automotive V2X TCU — the forensic logging / GDPR schema separation
problem; (3) Industrial SCADA — what the Purdue model cannot see
and Zero Trust as augmentation.

### [Part 7 — Extending Trust to the Edge AI Stack](docs/part7-edge-ai.md)
Model weights as a new principal the framework did not address.
Three properties model weights require. SPDM for accelerator-to-host
attestation. The open problem of inference result integrity, honestly
stated.

### [Appendix A — Regulatory Quick Reference](docs/appendix-a-regulatory-reference.md)
Navigation table: regulation → scope → key engineering obligation
→ relevant series section. Covers CRA, R155/R156, IEC 62443, FIPS,
NIST SP 800-193/218, PSA Certified, DICE, SPDM, RATS, CNSA 2.0,
NIST PQC standards.

### [Appendix B — UNECE R155 Annex 5 Practitioner Mapping](docs/appendix-b-r155-annex5.md)
All eight R155 Annex 5 Part B threat tables annotated with
engineering implementation notes. BSP/OEM obligation split.
Forensic logging / GDPR dual-obligation items marked. EVITA HSM
tier guidance. Standalone reference for automotive BSP engineers.

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
| Part 1 — Hardware Root of Trust | ✅ Complete | ~2,800 |
| Part 2 — Trust Extension Across the Stack | ✅ Complete | ~3,460 |
| Part 3 — Secure Provisioning and Key Hierarchy | ✅ Complete | ~2,700 |
| Part 4 — Secure OTA and Fleet Lifecycle | ✅ Complete | ~1,320 |
| Part 5 — Threat Modelling Methodology | ✅ Complete | ~2,900 |
| Part 6 — Case Studies | ✅ Complete | ~2,070 |
| Part 7 — Extending Trust to the Edge AI Stack | ✅ Complete | ~1,420 |
| Appendix A — Regulatory Quick Reference | ✅ Complete | ~900 |
| Appendix B — UNECE R155 Annex 5 Practitioner Mapping | ✅ Complete | ~2,780 |

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
