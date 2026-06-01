<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# Part 5 — Threat Modelling Methodology

> **Series context:** Parts 1–4 established what to build — the
> hardware trust anchor, the trust extension stack, the provisioning
> architecture, and the OTA lifecycle. This part covers how to reason
> about whether what you have built is sufficient. The methodology
> here applies before a design is committed, during implementation
> review, and as a structured input to certification evidence.
>
> **Relationship to Appendix B:** Appendix B applies this methodology
> to the specific regulatory context of UNECE R155 Annex 5. Read this
> part first for the general framework; read Appendix B for the
> automotive regulatory instantiation.

---

## The fundamental question

A threat model is not a document. It is an answer to a specific
question: given everything an attacker might want from this system,
and everything they might try, what have we built that stops them —
and where have we left gaps?

The document is evidence that you have asked and answered the question
systematically. The question is the work. The document is the record.

Most security failures in connected devices are not failures of
cryptographic strength. They are failures of scope: the team built
strong protection for the assets they were thinking about, and missed
the ones they were not. A threat model is the process that makes
"the assets we were not thinking about" visible before an attacker
finds them.

This part presents a methodology in three steps: define the assets,
define the attacker, map the gaps. Each step has a concrete artifact
that can be reviewed, challenged, and handed to a certification
evaluator as evidence.

---

## §5.1 — Step 1: Asset identification using the asset table

The first and most important step is not to think about threats. It
is to think about what the attacker wants. An attacker who compromises
a device is not interested in the device — they are interested in what
the device holds, does, or can reach. Define that first.

The asset table is the artifact that makes this concrete. It has a
fixed schema that forces the right questions before any threat
analysis begins.

### The schema

| Column | What it captures | Why it matters |
|---|---|---|
| **Asset** | What is being protected (key material, credential, PII, firmware binary, command channel, sensor data, device identity) | Forces explicit enumeration — "security" is not an asset |
| **Asset class** | Secret / Identity / Code / Data / Channel / Capability | Groups assets by attacker motivation; different classes attract different attack techniques |
| **State** | Data at Rest (DAR) / Data in Transit (DIT) / Data in Use (DIU) | An asset's protection requirements change with state; a key in TEE SRAM (DIU) needs different protection than the same key in RPMB (DAR) |
| **Storage / location** | Where the asset lives in each state — OTP, TEE SRAM, RPMB, encrypted filesystem, external flash, DRAM, network channel | Determines which mechanisms are in the protection path |
| **Owner / accessor** | Which software component or hardware block is the legitimate accessor | Defines the access policy; anything outside this list is an unauthorised access |
| **Security properties required** | Confidentiality / Integrity / Availability / Authenticity / Non-repudiation | Determines which security controls are needed; not every asset needs all five |
| **Provisioning path** | How the asset reaches the device — OTP fusing, factory TA injection, cloud provisioning, user enrolment, derived at runtime | The provisioning path is a separate attack surface from the runtime storage path |
| **Residual risk** | What class of attacker can still reach this asset after all controls are applied | The honest conclusion of the analysis; drives escalation decisions |

### How to populate the table

Start with the most valuable asset — the one whose compromise would
be most damaging — and work outward. Ask:

**"Where does this asset exist in each state?"** A device certificate
exists as: a public/private keypair in HSM during provisioning (DIU),
a signed blob in RPMB (DAR), a TLS client certificate in a network
handshake (DIT), and an in-memory object in the TEE during a signing
operation (DIU). Each state is a separate row or a separate column
with distinct protection requirements.

**"Who is the legitimate accessor, and who is everyone else?"** The
TEE signing key should be accessible only to the attestation TA
running in Secure World. If the Linux kernel can read it directly, the
access policy is wrong. The table makes this explicit and reviewable.

**"What is the provisioning path, and who touches the asset during
that path?"** Provisioning is the highest-risk moment for any secret
asset (covered in Part 3). The asset table forces this question for
every secret — not just the HUK, but every credential injected during
manufacturing.

### A worked example: generic connected device asset table

The following table illustrates the schema applied to a representative
connected device with a TEE and hardware-backed keystore. Platform
names are deliberately omitted — the schema applies across mobile,
automotive, industrial, and AIoT platforms.

| Asset | Class | State | Storage / location | Legitimate accessor | Security properties | Provisioning path | Residual risk after controls |
|---|---|---|---|---|---|---|---|
| Hardware Unique Key (HUK) | Secret | DAR | OTP fuses (immutable) | TEE crypto engine only (never exported) | Confidentiality, Integrity | Silicon manufacturer — fused at wafer or package test | Physical attack on OTP cells (lab-grade) |
| Device attestation key | Secret / Identity | DAR / DIU | Hardware-sealed keystore (HSM or TEE non-extractable) | Attestation TA (Secure World only) | Confidentiality, Integrity, Authenticity | Factory TA provisioning, or DICE-derived at first boot | Physical attack on security element |
| Device certificate (public) | Identity | DAR / DIT | Filesystem (unencrypted acceptable) / TLS handshake | Any component (public) | Integrity, Authenticity | Factory PKI injection or cloud provisioning | Certificate spoofing if CA compromised |
| OTA signing root certificate | Identity / Code | DAR | Read-only partition or TEE-sealed | OTA verification component | Integrity, Authenticity | Factory or secure provisioning channel | Rollback to old cert if revocation not implemented |
| Firmware image | Code | DAR / DIT | Flash (mutable partition) / OTA channel | Bootloader (verifier), OS (executor) | Integrity, Authenticity | Build system → signing HSM → OTA delivery | Supply chain compromise of build system |
| User PII (location, identity, biometric template) | Data | DAR / DIU | Encrypted partition (key TEE-derived) / TEE SRAM during operation | User-authorised application, TEE biometric TA | Confidentiality, Integrity, Availability | User enrolment | Insider threat; OS compromise if encryption key derivable from software |
| Session key (TLS, DTLS) | Secret | DIU | TEE SRAM or hardware crypto engine | Network stack TA or TLS library | Confidentiality | Ephemeral — derived per session | Forward secrecy failure if long-term key compromised |
| V2X pseudonym certificate | Identity | DAR / DIT / DIU | EVITA HSM / V2X message payload | V2X communication stack | Authenticity, Privacy (non-linkability) | V2X PKI (SCMS or equivalent) | Long-term identity linkage if pseudonym rotation fails |
| Safety-critical parameter (brake threshold, airbag trigger) | Data / Capability | DAR / DIU | OTP or TEE-sealed config store | Safety-critical firmware (authenticated session only) | Integrity, Authenticity, Availability | Manufacturing calibration | Authenticated session compromise |
| Security event log | Data | DAR | Append-only, TEE-protected or RPMB-backed | Security monitoring component (read); authorised auditor (export) | Integrity, Availability | Generated at runtime | Physical extraction if storage not encrypted |

### What the table reveals that informal analysis misses

Two systematic gaps appear in almost every asset table that has not
been done before:

**Gap 1 — The provisioning path is a separate attack surface.** Most
teams think about protecting assets at runtime. The asset table forces
the provisioning path into the same frame. A HUK that is perfectly
protected at runtime but injected via an unauthenticated manufacturing
fixture interface has a gap at its most vulnerable moment.

**Gap 2 — DIU is consistently underprotected.** An asset in RPMB
(DAR) may have strong encryption. The same asset loaded into TEE SRAM
for an operation (DIU) may have no equivalent protection — it is in
plaintext in memory that a Spectre-class attack can observe. The state
column forces this distinction. Without it, teams tend to protect
storage and forget processing.

---

## §5.2 — Step 2: Three-class attacker model

Once the assets are defined, the attacker model gives the analysis
its boundaries. The most productive attacker model for connected
devices is the three-class taxonomy: remote, local, and physical.
Each class has a distinct capability set, a distinct cost profile, and
a distinct set of mechanisms that stops it.

### Class 1 — Remote attacker

**Capability:** Network access only. Can send packets, exploit
services, interact with public APIs. Cannot run code on the device
without first achieving remote code execution.

**Cost:** Low. Attacks are scalable, automated, and repeatable.
A single exploit script can be deployed against millions of devices
simultaneously.

**What stops them:** OS hardening (ASLR, stack canaries, NX/XN,
SELinux, seccomp), input validation at every network-facing interface,
rate limiting, mutual TLS authentication, and TEE isolation for
secrets. A remote attacker who achieves full REE kernel compromise
still cannot reach TEE-protected assets if the TEE boundary is
correctly enforced.

**What does not stop them:** Any secret that is accessible from
software running in the REE, regardless of privilege level.

### Class 2 — Local attacker

**Capability:** Code execution on the device, potentially at high
privilege (root, kernel). May have achieved this through remote
exploitation or through physical access to a debug port. Can observe
timing, interact with local interfaces, and in some cases run
side-channel attacks against shared microarchitecture resources.

**Cost:** Medium. Requires either a successful prior remote
exploitation or physical access. Not easily scalable.

**What stops them:** Physical separation of security-sensitive
operations onto a discrete security processor (Tensor Security Core,
SPU, CryptoCell, Secure Enclave) with independent execution pipelines.
A local attacker with full TrustZone Secure World compromise cannot
observe operations executing inside a physically separate processor.

**What does not stop them:** Any operation that shares
microarchitecture resources (cache, branch predictor) with the
attacker's code. Spectre/Meltdown class attacks exploit this sharing.

### Class 3 — Physical attacker

**Capability:** Direct hardware access. Can decap the chip, probe
buses, inject faults (power glitching, clock manipulation, laser
fault injection), and perform differential power analysis (DPA) or
electromagnetic analysis (EMA) on the crypto engine.

**Cost:** High. Requires laboratory equipment and specialist skills.
Not scalable — each device must be attacked individually.

**What stops them:** Tamper-resistant HSM with active shield mesh,
power supply monitoring, side-channel-resistant crypto implementation,
and physical hardening tested and certified to FIPS 140-2 Level 3
or Common Criteria EAL4+.

**What does not stop them:** Any secret that exists in the application
SoC, regardless of software protection. The SoC is not designed for
physical attack resistance.

### Mapping attackers to assets

For each row in the asset table, identify which class of attacker
can reach the asset given the stated controls. This produces the
"Residual risk after controls" column.

The critical check: **is the residual risk class consistent with the
asset's value?** A HUK reachable by a remote attacker is an
architecture failure. A session key reachable only by a physical
attacker with laboratory equipment may be an acceptable residual risk
depending on the deployment context.

---

## §5.3 — Step 3: STRIDE threat enumeration against the data flow

With assets defined and attacker classes established, STRIDE provides
a systematic lens for generating threats. Applied to the data flow
diagram of the system — showing how assets move between components
across trust boundaries — STRIDE prevents the most common omission:
threat categories that the team simply did not think to consider.

STRIDE maps to CIA (Confidentiality, Integrity, Availability)
as follows, with concrete embedded system examples:

| STRIDE category | Property violated | Embedded system example |
|---|---|---|
| **S**poofing | Authenticity | Device clone presenting valid credentials; malicious V2X message with forged certificate; counterfeit ECU on CAN bus |
| **T**ampering | Integrity | Bootloader replaced with modified version; sensor data falsified in transit; OTA package modified before verification |
| **R**epudiation | Non-repudiation | Security event log deleted or modified; audit trail of key operations erased; diagnostic session unlogged |
| **I**nformation disclosure | Confidentiality | Key material extracted from flash; PII leaked through debug port; plaintext content in unprotected DRAM buffer |
| **D**enial of service | Availability | CAN bus flooding; OTA update server DoS preventing critical patch; TEE service exhaustion causing safety function unavailability |
| **E**levation of privilege | Authorisation | Unprivileged process gaining kernel access; REE component accessing Secure World assets; diagnostic session bypassing authentication |

### Applying STRIDE to trust boundaries, not components

The most common mistake in applying STRIDE to embedded systems is
applying it to components ("what threats does the TEE face?") rather
than to trust boundary crossings ("what threats exist when data
crosses the REE → TEE boundary?"). The threats that matter most are
at the crossings, not inside the components.

For each trust boundary in the system:

1. **Identify all data that crosses this boundary** (both directions)
2. **Apply all six STRIDE categories to each crossing**
3. **For each identified threat, trace back to the asset table:** which asset is affected? What is the attacker class that could execute this threat?
4. **Identify the existing control** that addresses the threat, or flag it as an unmitigated gap

The output of this process — a threat per boundary crossing per STRIDE
category, linked to the asset table — is the evidence package that
both internal review and certification evaluators can verify.

---

## §5.4 — Crypto agility as a threat model dimension

One threat category that STRIDE does not natively surface — because
it is a lifecycle property rather than a point-in-time attack — is
algorithm obsolescence. An attacker who cannot break AES-256 today
may be able to break RSA-2048 within a decade using quantum
computing. A device with a ten-year operational life must be designed
for the threat model it faces at the end of that life, not only at
deployment.

Crypto agility is the property that allows a cryptographic algorithm
to be replaced without requiring a full firmware rebuild. It is not
a nice-to-have. For any connected device with an expected operational
life beyond five years, it is a design requirement.

**The engineering discipline for crypto agility:**

Every cryptographic algorithm selection in the system should be
treated as a parameter, not a constant. For each algorithm in use,
the design must answer:

- What is the migration path when this algorithm is deprecated?
- Can the algorithm be updated via OTA without requiring hardware
  replacement?
- Does the hardware crypto engine support the successor algorithm,
  or will the upgrade require software fallback?

**NIST PQC transition as a concrete near-term instance:**

NIST standardised the first post-quantum cryptographic algorithms in
2024 (FIPS 203 ML-KEM, FIPS 204 ML-DSA, FIPS 205 SLH-DSA). For
devices using RSA or ECDSA for firmware signing, attestation, or
key exchange, the migration question is not hypothetical — it is a
scheduled engineering task. CNSA 2.0 (the NSA's Commercial National
Security Algorithm Suite 2.0) mandates PQC algorithms for national
security systems from 2025, with a transition window through 2033.

The threat model dimension this introduces is not "quantum computer
breaks RSA tomorrow." It is "an adversary harvests ciphertext or
signed messages today and decrypts or forges them when a quantum
computer becomes available." For devices that sign long-lived
certificates or encrypt long-lived secrets, this is a present threat
requiring present design attention.

The asset table column "Security properties required" should include
an algorithm lifetime assessment for any asset whose protection
depends on asymmetric cryptography. If the asset's required protection
period extends beyond the estimated algorithm lifetime, the design
must account for migration.

---

## Putting the three steps together

The threat model for a connected device is complete when:

1. **The asset table is populated** — every asset enumerated, every
   state covered, every provisioning path traced, every residual risk
   honestly stated

2. **The attacker classes are mapped to assets** — for each asset,
   the highest class of attacker that can reach it (given current
   controls) is identified, and the design decision to accept or
   mitigate that residual risk is documented

3. **STRIDE has been applied to every trust boundary crossing** —
   every threat identified, every existing control linked, every
   gap flagged and assigned for resolution

4. **Crypto agility is addressed** — every algorithm has a documented
   migration path, and assets with long protection requirements are
   assessed against algorithm lifetime

This is not a one-time exercise. The threat model is a living artifact
that must be revisited when the attack surface changes — a new
interface is added, a new component is integrated, a new CVE is
published against a dependency. In the regulatory context, R155
requires continuous monitoring and analysis of emerging threats
throughout the vehicle's operational life. The threat model is the
structured baseline against which new threat intelligence is assessed.

---

## What comes next

Part 5 has provided the methodology. Part 6 applies it to three
concrete platform contexts — mobile, automotive, and industrial OT —
where the same methodology produces different results because the
asset set, the attacker model, and the deployment constraints differ.

---

*© 2026 Lei Zhou. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
*Part of the [IoT Security Lifecycle](https://github.com/zlhk100/iot-security-lifecycle) series.*
