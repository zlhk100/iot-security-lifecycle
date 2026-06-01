<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# Part 2 — Trust Extension Across the Stack

> **Series context:** Part 1 established what must be unconditionally
> trusted at the hardware level. This part covers how that trust is
> extended upward — through the boot chain, across the TEE boundary,
> and into the operating system — and where that extension breaks under
> adversarial conditions.
>
> **Companion reading:** For the mechanism-level detail of every
> isolation primitive discussed here (SAU, MPU, MPC, TZASC, IO-SMMU,
> RDC), see [Security and Safety by Spatial and Temporal
> Isolation][layer1-post]. This part covers why those mechanisms must
> be correctly orchestrated; that article covers how each one works.

[layer1-post]: https://medium.com/@zlhk100/security-and-safety-by-spatial-and-temporal-isolation-design-pattern-behind-tz-smmu-mpu-and-mpam-f1be9b4da33d

---

## The problem Part 1 left open

Part 1 answered one question: is there anything on this device that
an attacker cannot forge, replace, or extract? The hardware Root of
Trust answers yes — and Part 1 described the three guarantees that
make that answer meaningful.

But the RoT is not the device. It is one component, or a small cluster
of components, at the bottom of a stack that grows significantly taller
before it becomes useful. The stack includes a bootloader, a secure
operating environment, a rich operating system, application code, and
network-facing services. The trust established at the hardware level
must propagate upward through every layer of that stack before it
reaches the software that actually handles the assets an attacker
wants.

**Trust does not propagate automatically.** It must be explicitly
extended at each boundary — and each boundary introduces a new
question: what can the attacker reach here, and what does this layer's
enforcement mechanism not see?

That second question is the subject of this part.

---

## §2.1 — The blind spot principle

Every hardware isolation mechanism has a blind spot: the transactions
it never sees. Understanding where that blind spot is determines
whether your security architecture holds under adversarial conditions
or only under test conditions.

This is not a theoretical observation. It is the most common class of
security architecture failure in modern connected devices — and it
follows from a structural property of every isolation mechanism ever
built.

### Two sides of every bus transaction

Every memory access and peripheral access in a modern SoC is a bus
transaction. Every bus transaction has two sides:

**The initiator side** is where the transaction originates. The
initiator — a CPU core, a DMA engine, a GPU, an NPU, a video codec —
generates the transaction and tags it with an identity. Initiator-side
enforcement mechanisms intercept this transaction at the source and
apply a policy based on the identity tag.

**The target side** is where the transaction lands. The target — a
memory range, a peripheral register, a cache partition — receives the
transaction. Target-side enforcement mechanisms check the identity tag
at the destination and enforce a policy before allowing access.

The same three-part structure — *initiator identity × target resource
→ access policy* — governs both sides. The mechanism is always
answering the same question. What differs is where in the transaction
flow the question is asked.

### The critical gap

The gap that causes most real-world failures is this: **CPU-side
isolation mechanisms only see transactions the CPU initiates.**

A CPU in TrustZone Secure state, protected by a correctly configured
SAU (on Cortex-M) or TZASC (on Cortex-A), cannot be reached by
Non-Secure software through the CPU path. But a DMA engine, a GPU, an
NPU, a video decoder, a network accelerator — these are independent
bus masters. They generate their own transactions. They do not have a
TrustZone state machine. They are invisible to every initiator-side
enforcement mechanism on the CPU path.

An attacker who controls the Non-Secure OS controls every DMA-capable
peripheral that the OS can programme. If a Secure memory region has
only initiator-side protection (SAU marking it as Secure, TZASC not
configured for target-side enforcement on that region), a compromised
NS OS can programme a DMA engine to read or write that region directly
— bypassing all CPU-side enforcement.

This gap is not specific to ARM TrustZone. It appears in every
architecture:

| Domain | CPU-side mechanism | What it misses | Target-side complement |
|---|---|---|---|
| Cortex-M SoC | SAU / IDAU | DMA, crypto accelerator | MPC, PPC, TGU |
| Cortex-A SoC | TZASC (NS/S attribution) | GPU, video codec, NPU | SMMU stage 2, RDC |
| Automotive HSM | HSM memory firewall | CAN controller, ETH DMA | Peripheral firewall |
| Edge AI platform | Hypervisor stage-2 tables | NPU, DSP DMA | SMMU, IOMMU |
| Industrial FPGA SoC | MPU (processor side) | FPGA fabric DMA masters | MPU target region |

The pattern is identical across all domains. The solution is also
identical: **any asset requiring protection against both CPU compromise
and DMA bypass must have target-side enforcement.** Initiator-side
alone is insufficient against an attacker who controls the OS.

This is the lens through which to read every isolation decision in the
sections that follow.

---

## §2.2 — The boot chain: extending trust from silicon to software

The hardware RoT establishes an unconditional foundation. The boot
chain extends that foundation upward to the software environment that
will eventually run on the device. Every link in the chain must be
verified before it executes — and each link must verify the next one
using a key or certificate whose validity is rooted in the hardware
RoT.

### The chain structure

The boot chain is not a single linear sequence. It splits at the
world boundary into two parallel paths — one for Secure World, one
for Non-Secure World — and each path has its own verifier for the
software it loads. This distinction matters: the Non-Secure OS has
no authority over, and no visibility into, Secure World assets.
Trusted Applications and Secure Partitions are verified within the
Secure World path, never by the NS kernel.

```
Boot ROM (immutable, in silicon)
  ↓ verifies signature of
First-stage bootloader
  ↓ verifies signature of
Second-stage bootloader
  │
  ├─── Secure World path ──────────────────────────────────────
  │      ↓ verifies and loads into Secure World
  │     TEE OS / Secure Firmware  (OP-TEE, TF-M, Trusty)
  │      ↓ verifies at TA load time        ↓ verified at boot
  │     [OP-TEE/Trusty model:         [TF-M/PSA model:
  │      TAs loaded dynamically        Secure Partitions baked
  │      from TA store, signed         into firmware image,
  │      with TA developer key,        verified by BL2 —
  │      rooted in TEE trust anchor]   no runtime TA loading]
  │
  └─── Non-Secure World path ──────────────────────────────────
         ↓ verifies and loads
        NS OS kernel + initial ramdisk
         ↓ (no verification role for any Secure World asset)
        NS applications / services
```

Each arrow represents a cryptographic verification step rooted in
the hardware RoT. The critical architectural property is the world
boundary separation: the NS OS kernel is never in the verification
path for Trusted Applications or Secure Partitions. It cannot be —
it runs at lower privilege in Non-Secure World and has no access to
Secure World memory where TEE assets are loaded and verified.

Breaking any link in either chain requires either forging a
signature (breaking the cryptographic primitive) or compromising
the previous link's key material — which is protected by the RoT
guarantees from Part 1.

**The two TA/SP verification models in practice:**

In the **OP-TEE / Trusty model** (Cortex-A, mobile and automotive
platforms), Trusted Applications are dynamically loaded at runtime
from a TA store. Each TA binary is signed with a TA developer key
whose certificate is rooted in the TEE OS's own trust anchor. The
TEE OS verifies each TA when a client requests it — the NS OS has
no role in this process.

In the **TF-M / PSA model** (Cortex-M, IoT and embedded platforms),
Secure Partitions are statically compiled into the secure firmware
image and verified by the bootloader (BL2, typically MCUboot) as
part of the boot sequence. There is no runtime TA loading in the
standard PSA configuration — the set of Secure Partitions is fixed
at build time and their integrity is established at boot, not on
demand.

### What the chain protects against

**Substitution:** An attacker cannot replace a bootloader or kernel
with a modified version, because any image not signed by the
authorised key will fail verification.

**Downgrade:** An anti-rollback counter in OTP prevents presenting an
old, signed image with a known vulnerability. Each firmware update
increments the counter; images with a lower version number are
rejected. This is a separate mechanism from signature verification and
must be explicitly implemented — signature verification alone does not
prevent downgrade.

**Tampering during storage:** The signature covers the entire image.
Any byte modification — whether accidental or deliberate — changes the
hash and invalidates the signature.

### What the chain does not protect against

**Runtime attacks.** Once a verified image is running, the boot chain
has no further role. A kernel vulnerability exploited after boot is
outside the chain's scope.

**A compromised signing infrastructure.** The chain is as strong as
the key used to sign the images. If the signing HSM is compromised,
valid signatures can be produced for malicious images. Key custody and
signing infrastructure security are outside the device itself but
are part of the overall security architecture.

**The boot chain does not prove anything to anyone else.** Secure
boot makes a local trust decision — it decides whether to run this
image. It does not report that decision to a remote party. A server
granting access to a licence, an update, or a sensitive service cannot
rely on secure boot alone — it cannot observe the outcome. This is
where measured boot and attestation enter, covered in §2.4.

### Boot chain implementation: two dominant patterns

**UEFI Secure Boot** (Linux-class and Windows platforms, automotive
infotainment, industrial gateways):

The UEFI firmware maintains four databases: PK (Platform Key), KEK
(Key Exchange Key), DB (Signature Database), and DBX (Forbidden
Signature Database). The PK is the root of trust for the database
hierarchy — only the PK holder can update KEK; only KEK holders can
update DB and DBX. DB contains keys authorised to sign boot images;
DBX contains explicitly revoked signatures and certificates.

At boot, UEFI verifies each executable against DB and checks it
against DBX before allowing execution. The entire database hierarchy
is signed by the PK, which is typically protected in hardware
(TPM PCR binding or secure element storage).

**Verified Boot / FIT signature** (embedded Linux, automotive ECUs,
industrial controllers):

U-Boot's Verified Boot model uses a device tree with embedded public
keys and hash values for each image component. The public key is baked
into the U-Boot binary itself, which is verified by the SoC's
hardware secure boot mechanism (HAB, TrustZone ROTPK, or equivalent).
This model is simpler than UEFI and better suited to constrained
platforms where the full UEFI stack is not appropriate.

Both models answer the same question — "was this image authorised by
the entity that provisioned this device?" — through different
mechanisms suited to different platform classes.

---

## §2.3 — The TEE boundary: what TrustZone provides and where it stops

The Trusted Execution Environment is the software-defined secure
domain that sits above the hardware RoT and below the rich operating
system. It provides a protected execution environment for
security-sensitive operations — key management, biometric
authentication, DRM content decryption, attestation token generation
— that must be isolated from the OS and its attack surface.

### What TrustZone provides

TrustZone partitions the processor into two worlds: Secure and
Non-Secure. The partition is enforced at the processor bus level —
every transaction is tagged with a security attribute (HNONSEC on the
AXI bus) that carries through the entire memory subsystem.

Secure World has access to all memory and peripherals. Non-Secure
World has access only to resources explicitly designated Non-Secure.
The TEE OS (OP-TEE, Trusty, a proprietary equivalent) runs in Secure
World. The rich OS (Linux, Android, an RTOS) runs in Non-Secure World.
Transitions between worlds happen only through a defined gate: the
Secure Monitor Call (SMC), handled at EL3 on Cortex-A or by the
Secure firmware on Cortex-M.

The enforcement mechanisms that give this partition meaning at the
hardware level are the target-side mechanisms described in §2.1:
TZASC (or TZC-400) for DRAM region protection, MPC/PPC for SRAM and
peripheral protection on Cortex-M platforms, and IO-SMMU stage 2 for
DMA isolation. TrustZone without these target-side mechanisms is
incomplete — the CPU-side world state is meaningless if DMA engines
can bypass it.

### Where TrustZone stops

**Microarchitectural side channels.** TrustZone shares CPU
microarchitecture pipelines — branch predictors, cache hierarchies,
execution units — between Secure and Non-Secure worlds. Spectre and
Meltdown class attacks exploit microarchitectural state that is shared
across the world boundary. A sufficiently capable attacker running
code in the Non-Secure world can use timing side channels to infer
information about Secure World execution.

This is not a theoretical concern. Published research has demonstrated
practical cross-world cache timing attacks against ARM TrustZone. The
threat is real and the mitigation is architectural: a physically
separate security processor eliminates the shared microarchitecture
pipeline.

**A concrete illustration:** Google Pixel 6 addresses this
architectural limitation explicitly. The Tensor Security Core is a
physically separate security processor — on the same die as the
application processor but with independent execution pipelines,
dedicated SRAM, and no shared cache hierarchy with the application
processor. Trusted operations that require resistance to
microarchitectural side channels are offloaded from TrustZone to the
Tensor Security Core. TrustZone handles the isolation boundary;
the Tensor Security Core handles the operations that require stronger
physical isolation.

This design pattern — TrustZone for logical isolation, discrete
security processor for side-channel-resistant operations — is not
unique to Google. It appears as: Qualcomm SPU (Secure Processing
Unit) in Snapdragon, Apple Secure Enclave in iPhone/iPad, Nordic
Semiconductor CryptoCell in IoT SoCs, STMicroelectronics STSAFE in
automotive platforms. The names differ; the architectural reason is
the same.

**Physical attacks.** TrustZone provides no physical attack
resistance. A device with physical access is vulnerable to fault
injection (glitching the power supply or clock to skip a verification
step), DPA (differential power analysis of the crypto engine), and
direct probing of the bus. The Titan M2 HSM in Pixel 6 addresses this
layer — TrustZone does not.

---

## §2.4 — Defense in depth: the complete hierarchy

The three-class attack taxonomy — remote, local, physical — maps
directly to the layered hardware architecture required to address each
class. No single mechanism addresses all three. The architecture works
because each layer handles the class of attack the previous layer
cannot.

```
Class 1 — Remote attacks
  Entry point: network stack, cloud telemetry, OTA channel,
               Bluetooth pairing, REST API
  Primary defense: OS hardening + TEE isolation
    ├── Privilege separation (DAC, SELinux MAC, seccomp)
    ├── Address space randomisation (KASLR, ASLR)
    ├── Stack canaries, heap hardening, NX/XN enforcement
    └── TEE for secrets (remote attacker cannot reach Secure World
        through network-facing code in REE)

Class 2 — Local attacks
  Entry point: local code execution (compromised app, malicious SDK,
               privilege escalation from REE)
  Primary defense: Physical security processor separation
    ├── Discrete security processor (Tensor Security Core, SPU,
        Secure Enclave, CryptoCell) for side-channel resistance
    ├── Microarchitecturally isolated pipelines (no shared cache
        with application processor)
    └── Hardware-enforced key operations (key never leaves
        security processor in plaintext)

Class 3 — Physical attacks
  Entry point: direct hardware access (chip decapping, fault
               injection, DPA, bus probing)
  Primary defense: Tamper-resistant HSM
    ├── Physically hardened secure element (Titan M2, STSAFE,
        SE050, Infineon SLB9670)
    ├── Active shield mesh, power supply monitoring
    ├── Side-channel resistant crypto implementation
    └── FIPS 140-2 Level 3 / Common Criteria EAL4+ certification
        as evidence of physical hardening
```

Each layer is necessary. None is sufficient alone.

**Remote attackers** are stopped by the TEE boundary — code running
in the REE, no matter how privileged, cannot directly access Secure
World memory or invoke Trusted Applications except through the defined
SMC gate. An attacker who achieves full Linux kernel compromise still
cannot extract a key stored in the TEE.

**Local attackers** who compromise the TEE itself (via TEE
vulnerability, side-channel, or fault injection against TrustZone)
are stopped by the physically separate security processor. Operations
delegated to the security processor execute on independent silicon with
no shared microarchitecture. A compromised TrustZone Secure World
cannot observe or influence what happens inside the security processor.

**Physical attackers** who target the application SoC are stopped by
the HSM's physical hardening. Titan M2, STSAFE, and equivalent devices
are built specifically to resist the laboratory-grade attack tools that
can compromise an application SoC. Key material in the HSM cannot be
extracted by decapping the application processor.

**The architectural consequence:** The security of the highest-value
assets (the root secret, the device identity credential, the biometric
template) depends on the entire chain being present. Removing any
layer reduces the class of attacker that is excluded.

### Memory safety as a baseline requirement

Underpinning the remote attack defense is a baseline property that
determines how hard the OS hardening layer must work: how many
exploitable vulnerabilities exist in the code that handles
network-facing input.

Approximately 70% of serious security vulnerabilities in large
software projects are memory safety bugs — buffer overflows,
use-after-free, out-of-bounds reads — arising from the absence of
memory safety enforcement in C and C++. This figure has been reported
independently by Microsoft, Google Chromium, and Mozilla from their
own CVE analysis, and is cited by CISA in its guidance on
memory-safe languages.

The embedded and IoT space runs predominantly on C. This does not mean
C must be abandoned — but it does mean that the OS hardening layer
(ASLR, stack canaries, NX/XN, SELinux, seccomp) is compensating for
a structural property of the codebase, not eliminating the underlying
source of vulnerability. Memory-safe languages (Rust, Ada/SPARK) at
the network-facing boundary reduce the compensating work required from
the OS hardening layer. This is an architectural design decision, not
a language preference.

---

## §2.5 — Attestation: proving the architecture is correctly assembled

Secure boot verifies locally that the correct software is running.
Measured boot records what ran. But neither answers the question a
remote party needs to answer before trusting a device with sensitive
material: **is this device currently running the expected, uncompromised
software stack, and can it prove it?**

Attestation closes this gap. It is the mechanism by which a device
produces cryptographically verifiable evidence about its own software
state, signed by a key that is anchored in the hardware RoT and
therefore cannot be forged by software.

### Two attestation models

**The passport model:** The device holds a pre-issued attestation
certificate — analogous to a passport issued by a trusted authority
before the journey. When a relying party asks for evidence, the device
presents the certificate. The relying party verifies the certificate
against the issuing CA's root. This model works offline and is fast,
but the certificate was issued at a point in time and may not reflect
current software state.

**The background-check model:** When a relying party requests access,
it initiates a fresh verification — analogous to a background check
run at the moment of the request. The device produces a fresh
attestation report containing current measurements, signed by its
attestation key. The relying party sends this report to a verification
service that compares it against reference values for the expected
software build. This model reflects current state but requires a live
connection to the verification service.

Both models appear in production systems. PSA attestation (the Initial
Attestation Token, IAT) uses a hybrid: a pre-provisioned attestation
key whose certificate is from a PSA-certified root, combined with a
fresh token containing current measurements signed by that key. The
relying party can verify the key's legitimacy offline (certificate
chain) and the current measurements online or against a cached
reference.

### What attestation enables for the lifecycle

Attestation is the mechanism that makes the trust established in Parts
1 and 2 visible to the rest of the world. Without it:

- A fleet management system cannot verify that a device is running
  uncompromised firmware before pushing a sensitive configuration
- A cloud service cannot verify that the requesting device is a
  genuine hardware platform rather than a software emulator
- A licence server cannot verify that the requesting device's TEE
  is correctly configured before issuing a content decryption key
- A vehicle OEM cannot verify that a Tier 1 ECU is running the
  approved firmware before enabling a safety-critical function

With attestation, each of these decisions becomes verifiable rather
than assumed. The device's claim — "I am a genuine device running
firmware version X, with TEE correctly configured" — carries
cryptographic weight because the claim is signed by a key that lives
in verified hardware.

This is the bridge from Part 2 into Part 3. The attestation key must
be provisioned — and provisioning is where the next challenge lives.

---

## What comes next

Part 2 has established how trust extends from the hardware RoT through
the boot chain, the TEE boundary, and into attestation. Every
mechanism described here depends on one upstream condition: that the
secrets and keys required to make it work were correctly established
at device birth.

Part 3 covers secure provisioning — the process by which a device
acquires its identity, and why the factory floor is the highest-risk
moment in the device's operational life.

---

*© 2026 Lei Zhou. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
*Part of the [IoT Security Lifecycle](https://github.com/zlhk100/iot-security-lifecycle) series.*
