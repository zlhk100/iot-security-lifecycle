<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# Part 1 — Hardware Root of Trust

> **Series context:** This is Part 1 of the IoT Security Lifecycle series.
> It covers what a hardware Root of Trust must provide and how secrets
> are protected at different assurance levels. Part 2 covers how that
> trust is extended across the software stack.
>
> **Companion reading:** For the isolation mechanisms that enforce the
> boundaries described here (SAU, MPU, MPC, TZASC, IO-SMMU), see
> [Security and Safety by Spatial and Temporal Isolation][isolation-post].

[isolation-post]: https://medium.com/@zlhk100/security-and-safety-by-spatial-and-temporal-isolation-design-pattern-behind-tz-smmu-mpu-and-mpam-f1be9b4da33d

---

## The problem every connected device has

A connected device is, by definition, reachable. It receives software
updates from a server it has never physically met. It stores credentials
that unlock services worth attacking. It may control physical systems —
a vehicle, a grid relay, a medical infusion pump — where a compromised
command has consequences beyond data loss.

Every security property that matters for such a device — authentic
software, protected secrets, verifiable identity — ultimately rests on
one question: **is there anything on this device that an attacker
cannot forge, replace, or extract?**

If the answer is no, there is no security. If yes, that thing is the
Root of Trust.

---

## §1.1 — What a hardware Root of Trust must guarantee

A Root of Trust (RoT) is not a single component. It is a set of
guarantees that some component — or combination of components — must
provide as the unconditional foundation beneath all other security
functions. Three guarantees are necessary. None is sufficient alone.

### Guarantee 1: Trusted key storage

The RoT must be able to hold secret material — cryptographic keys,
device-unique seeds, manufacturer-provisioned credentials — in a way
that survives the following adversary:

- Full control of the operating system and all mutable software
- Physical read access to external storage (eMMC, NOR flash, MRAM)
- The ability to run arbitrary code at the highest unprivileged level

Against this adversary, the secret material must remain
non-extractable and non-substitutable. "Non-extractable" means it
cannot be read out in plaintext through any software path. "Non-
substitutable" means an attacker cannot replace it with a key they
control without the device detecting the change.

This is why secrets live in OTP fuses, on-die SRAM protected by
hardware firewalls, or inside physically hardened secure elements —
not on a filesystem, not in external flash, and not in a memory region
that any software stack can reach at runtime.

### Guarantee 2: Trusted cryptographic operations

Knowing a secret is only useful if you can use it safely. The RoT
must implement cryptographic operations — signing, verification,
encryption, decryption, key derivation — in an environment where:

- The operation executes on the secret without exposing it to
  the caller, even a caller running at high privilege
- The implementation is resistant to side-channel observation
  (timing, power, electromagnetic) at the assurance level the
  threat model demands
- The result — a signature, a derived key, an authenticated
  ciphertext — can be trusted by the recipient to have originated
  from genuine hardware

This last point is the bridge to attestation. A cryptographic
operation is only as trustworthy as the environment that performed it.
Hardware-backed crypto, executed inside a protected boundary, produces
evidence that a pure-software implementation cannot.

### Guarantee 3: Trusted bootstrapping

Neither of the above matters if an attacker can replace the first
code that runs at power-on. The RoT must be part of the very first
boot stage — boot ROM, immutable by construction — and must verify
the integrity and authenticity of every subsequent code stage before
transferring control to it.

This is the chain of trust. It does not start at the bootloader. It
does not start at the operating system. It starts at silicon: an
immutable instruction store that contains the manufacturer's
verification key and the logic to use it before executing anything
else. Every link in the chain from that point forward is only as
strong as the link before it.

**If any one of these three guarantees is absent, the RoT is not a
root.** A system with trusted crypto and trusted bootstrapping but no
protected key storage will have its master key extracted from external
flash. A system with protected key storage and trusted bootstrapping
but no hardware-backed crypto will have its operations observed
through a power side-channel. A system with protected keys and
hardware crypto but no verified boot will have its RoT bypassed by
replacing the bootloader.

---

## §1.2 — How secret protection actually works: a three-level taxonomy

Not all hardware platforms provide the same level of secret
protection. This taxonomy describes what an engineer actually gets at
each level — and what the residual risk is.

The levels are not marketing tiers. They describe concrete architectural
properties that determine what class of attacker can succeed. This is
a practical engineering framework, not a normative standard
classification — different certification schemes (PSA, FIPS 140-2,
Common Criteria) use different tiering structures, and the boundaries
here are chosen for clarity, not compliance mapping.

---

### Level 1 — Software-accessible secrets

**What it looks like:** The device secret (hardware unique key, root
seed, manufacturer credential) is stored in an OTP fuse array or
on-die secure memory, but is directly readable by privileged software
at runtime. Alternatively, secrets are stored encrypted on external
storage using a key that software can access.

**What it protects against:** An unprivileged attacker — someone who
can run code as a normal application but cannot escalate to kernel or
secure firmware level. The secret is at least not in plaintext on a
filesystem.

**What it does not protect against:** Any attacker who achieves kernel
code execution, secure monitor compromise, or direct hardware access.
The secret is one privilege escalation away from exposure.

**Concrete illustration:** Before Android 9 (API level 28), Android
devices commonly stored private keys and biometric credentials
encrypted on external eMMC flash via RPMB (Replay Protected Memory
Block). The encryption key was TEE-derived, which provides meaningful
protection — but the encrypted secret lives on removable storage
outside the SoC boundary, accessible for offline cryptanalysis if
the wrapping key is ever compromised.

**Residual risk:** Class attacks. If an attacker breaks the wrapping
key on one device, the same technique applies to every device in the
same product class with the same key derivation scheme.

---

### Level 2 — Hardware-sealed secrets

**What it looks like:** Secrets are injected during secure
manufacturing into a hardware-protected boundary from which they
cannot be read in plaintext by any software, including privileged
firmware. The only operations available are: derive a key from this
secret, use it to encrypt or sign, check if a presented value matches.
The secret itself never appears on a bus or in a register readable
by software.

**What it protects against:** All software attackers regardless of
privilege level. Kernel compromise, TEE compromise, and hypervisor
compromise cannot extract the root secret — they can request
operations that use it, but cannot read it.

**What it does not protect against:** Sophisticated physical attacks:
chip decapping with microprobing, fault injection to bypass access
control gates, differential power analysis against the crypto
engine.

**Concrete illustration:** Android StrongBox (Android 9+) mandates
that the hardware-backed keystore implementation reside inside a
physically separate security element — in the case of Google Pixel 6,
the Titan M2 (a RISC-V based HSM). Private keys generated or
imported into StrongBox never leave the Titan M2 boundary in
plaintext. Even if the application processor is fully compromised,
the signing key inside Titan M2 cannot be extracted — only signing
operations can be requested through its defined API.

This property — secrets that cannot be extracted regardless of
software privilege level — appears under different names across
platforms and certification schemes: Android StrongBox, PSA
Certified Level 2, FIPS 140-2 Level 3, Common Criteria AVA_VAN,
EVITA Full for automotive HSMs. The names differ. The architectural
property is the same: a hardware boundary that no software path
crosses in plaintext.

In the Android ecosystem, this boundary is made explicit in the
API: `setIsStrongBoxBacked(true)` is not a preference flag. It is
a declaration that the key must live on the hardware-sealed side
of that boundary — and a rejection of any implementation that
cannot guarantee it.

**Residual risk:** Physical attacks on the secure element hardware
itself, which require laboratory equipment and per-device effort —
no longer a class attack.

---

### Level 3 — Device-generated secrets that never exist outside silicon

**What it looks like:** The device secret is never provisioned from
outside. It is generated on-device using a Physical Unclonable
Function (PUF) — a circuit that exploits nanoscale manufacturing
variation to produce a value that is unique to that specific die,
reproducible across power cycles, and unpredictable to anyone
(including the manufacturer) before the device is fabricated.

> **Note:** PUF is a family of technologies, not a single design.
> SRAM PUF, ring-oscillator PUF, and arbiter PUF have different noise
> characteristics, temperature sensitivity, and tamper response
> properties. The description below reflects general SRAM PUF
> properties, which are the most widely deployed in IoT security
> hardware. Consult your specific silicon vendor's documentation for
> the security claims applicable to their implementation.

**What it protects against:** The entire Level 2 threat surface, plus
manufacturing-time injection attacks. There is no provisioning step
at which a secret is transferred from an HSM to the device — and
therefore no provisioning channel to intercept. The secret was never
outside the silicon.

**What it does not protect against:** Sophisticated physical attacks
that can probe the PUF response directly, or side-channel analysis
of the PUF reconstruction circuit. At this level, the attacker must
attack each device individually with invasive hardware techniques.

**Key design property:** PUF-derived keys are used exclusively as a
source for key derivation — never stored, never transmitted, never
directly used for data operations. The PUF circuit reconstructs the
same value on each power cycle from the physical characteristics of
the silicon. If the chip is decapped or physically probed in an
attempt to extract the PUF response, the attempt necessarily disturbs
the nanoscale transistor characteristics from which the response is
derived — making successful extraction without altering the response
exceedingly difficult. This is tamper evidence by construction, not
by a dedicated detection circuit.

**Residual risk:** Advanced laboratory attacks against a specific
device. A successful attack on one device yields no information about
any other device in the fleet.

---

### What level does your system need?

The answer depends entirely on your threat model's attacker profile
and your certification target:

| Attacker capability | Minimum level required |
|---|---|
| Remote software only | Level 1 is a floor; Level 2 recommended |
| Local code execution on the device | Level 2 required |
| Physical access to the device hardware | Level 2 minimum; Level 3 for highest-value assets |
| Nation-state / advanced laboratory | Level 3 + side-channel resistant crypto engine |
| PSA Certified Level 2 target | Level 2 (hardware-backed isolation of RoT) |
| PSA Certified Level 3 target | Level 3 + physical attack resistance mandatory |
| FIPS 140-2 Level 3 / Common Criteria EAL4+ | Level 2 minimum with side-channel resistance |

---

## §1.3 — Secure boot and measured boot: definitions and the relationship

These two terms are frequently conflated. They solve adjacent but
distinct problems, and a system can have either without the other —
though a production-grade connected device needs both.

---

### Secure boot: integrity and authenticity before execution

**Definition:** Secure boot is the mechanism that verifies the
integrity and authenticity of every mutable code image before
transferring execution to it. Each stage of the boot sequence is
cryptographically checked against a key rooted in the hardware RoT
before the processor is allowed to run it.

**What it prevents:** An attacker cannot load a modified or
substituted firmware image — a rootkit, a backdoored bootloader, a
downgraded OS — because any image that was not signed by the
authorised key will fail verification and boot will halt.

**What it does not do:** Secure boot is a boot-time mechanism only.
Once a verified image is running, secure boot has no further role.
If a vulnerability is later exploited in the running OS, secure boot
provides no runtime protection. It also does not tell anyone else —
a remote server, a certificate authority, a fleet management
system — that the boot succeeded correctly.

**The chain:** The boot ROM verifies the first-stage bootloader.
The first-stage bootloader verifies the second-stage bootloader.
The second-stage bootloader verifies the OS kernel. Each link uses
the key or certificate provided by the previous link. The entire
chain is only as strong as the first link — the boot ROM — which
is why the boot ROM must be immutable and must contain or verify
the manufacturer's root signing key.

**Anti-rollback:** Secure boot alone does not prevent downgrade
attacks. An attacker who cannot forge a new image can still present
an old, signed image with a known vulnerability. Anti-rollback
counters — stored in write-once OTP storage, incremented on each
firmware update — are the complement that closes this gap. The UNECE
R156 regulation (software update requirements for road vehicles)
requires at §7.2.1.1 that updates be protected against "invalid
updates" — which normatively implies anti-rollback protection,
confirmed explicitly as a required property of the R156 Secure
Update Chain in implementation guidance.

---

### Measured boot: evidence collection for later verification

**Definition:** Measured boot is the mechanism that records — in a
tamper-resistant log — a cryptographic digest of every code and
configuration component loaded during boot, so that a remote verifier
can later determine exactly what software state the device was in
when it started.

**What it enables:** Measured boot does not make a trust decision
itself. It collects measurements (hash digests) and extends them into
a protected log — a PCR in a TPM, a Compound Device Identifier (CDI)
in DICE — that cannot be retrospectively falsified. The trust decision is made
later, by a remote verifier who receives the log and compares it
against a known-good reference.

**Why this matters for connected devices:** A server granting access
to a medical record, a licence key, a vehicle firmware update, or a
payment transaction wants evidence that the requesting device is
running the expected software stack — not a compromised version, not
a downgraded version, not a modified version. Measured boot provides
that evidence. Secure boot alone cannot, because secure boot makes
its decision locally and never reports the result.

---

### The relationship: complementary, not redundant

| Property | Secure boot | Measured boot |
|---|---|---|
| Prevents unauthorised code execution | ✅ Yes | ❌ No |
| Allows remote verification of software state | ❌ No | ✅ Yes |
| Protects against runtime compromise | ❌ No | ❌ No (detects, does not prevent) |
| Required for PSA attestation (F.ATTESTATION) | Prerequisite | Mechanism |
| Required for UNECE R155 §7.3.7 forensic capability | No | Yes |

They are not alternatives. A device with only secure boot is
trustworthy at boot time but opaque to the rest of the world. A
device with only measured boot collects faithful evidence of whatever
ran — including unauthorised code — but cannot prevent it from
running in the first place. Both are required for a production
connected device.

---

### Two hardware implementations of measured boot

**TPM-based measurement (PC and Linux-class devices):**
The Trusted Platform Module provides a set of Platform Configuration
Registers (PCRs) that can only be extended — never written directly.
Each boot stage hashes the next stage's image and calls `TPM_Extend`
to mix that hash into the relevant PCR. The PCR value after the full
boot sequence is a compact, unforgeable summary of everything that
ran. A remote verifier receives a TPM Quote — a signed attestation
token containing the PCR values — and checks it against a reference
measurement for the expected software build.

**DICE-based measurement (constrained IoT devices):**
The Device Identifier Composition Engine (DICE) provides an
equivalent capability without requiring a discrete TPM. Each boot
layer derives a Compound Device Identifier (CDI) by combining the
previous layer's CDI with a measurement of the current layer's code.
The CDI is therefore unique to the exact software combination that
ran — if any component changes, the CDI changes for that layer and
all subsequent layers, which changes the identity the device presents
to a remote verifier. DICE is architecturally well-suited to the
constrained Cortex-M class of devices where a discrete TPM is not
integrated.

Both models answer the same question: "Given a device presenting this
attestation token, what software was running when the token was
produced?" The implementation mechanism differs; the security
property is the same.

---

## What comes next

Part 1 has established what must be unconditionally trusted at
the hardware level. Part 2 extends that trust upward through the
software stack — from the boot chain through the TEE boundary to
the operating system — and examines how the isolation mechanisms
that enforce these boundaries actually work.

The threat model behind these mechanisms is the subject of Part 5.
Before the mechanisms, it helps to understand what the attacker is
trying to do.

---

*© 2026 Lei Zhou. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
*Part of the [IoT Security Lifecycle](https://github.com/zlhk100/iot-security-lifecycle) series.*
