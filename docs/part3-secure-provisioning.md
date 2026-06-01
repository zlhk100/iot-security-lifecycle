<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# Part 3 — Secure Provisioning and Key Hierarchy

> **Series context:** Parts 1 and 2 established what must be trusted
> and how that trust extends upward through the software stack. Both
> depend on one upstream condition: that the secrets and keys required
> to make them work were correctly established at device birth. This
> part covers that establishment — the provisioning process and the
> key hierarchy it creates.

---

## The problem provisioning solves

A connected device's most vulnerable moment is not when it is deployed
in the field and exposed to the internet. It is on the factory floor,
during the seconds when the secret that will define its identity for
its entire operational life is being transferred from an HSM to
silicon.

Everything in Parts 1 and 2 depends on that transfer happening
correctly. The hardware Root of Trust is only trustworthy if the
secrets fused into it were generated with sufficient entropy, handled
by authorised personnel, and injected through an authenticated
channel. The attestation key is only meaningful if the certificate
that vouches for it was issued by a CA whose chain of custody was
protected during the issuance process. The firmware signing key is
only protective if it was never exposed in plaintext during the
signing infrastructure setup.

Most products treat provisioning as a manufacturing step. It is a
security protocol with a defined threat model and a defined attack
surface — and the attack surface is at its maximum exactly at the
moment of injection.

---

## §3.1 — What provisioning must establish

Every connected device requires three roots to be established at
provisioning time. None can be added or corrected after the device
leaves the factory without creating a new, separate trust establishment
event (which is itself a provisioning problem at one remove).

**Root 1 — The Hardware Unique Key (HUK)**

The HUK is the device's master secret from which all other keys are
derived. It must be:

- Generated with sufficient entropy (hardware TRNG output, not
  software-generated)
- Unique per device — a shared or class-wide HUK means a single
  compromise exposes the entire fleet
- Injected into OTP fuses or equivalent write-once storage through
  an authenticated channel
- Never stored in plaintext outside the secure provisioning facility
  after injection

The HUK is never used directly for cryptographic operations. It is
the root from which operational keys are derived. Direct use of the
HUK would create a side channel — every operation leaks information
about the raw key material. Derived keys compartmentalise the
exposure: a compromised derived key does not expose the HUK, and
therefore does not expose every other key in the hierarchy.

**Root 2 — The Root of Trust Public Key (ROTPK)**

The ROTPK is the hash of the public key used to verify the first
mutable boot stage. It is fused into OTP at provisioning time and
is consulted by the boot ROM on every boot to verify the first-stage
bootloader signature. Its integrity is therefore unconditional — an
attacker who can modify the ROTPK fuses can replace the entire
software stack.

The ROTPK hash is public — there is no confidentiality requirement
for a public key hash. The requirement is integrity and immutability.
OTP fusing satisfies both.

**Root 3 — Device identity credential**

The device identity credential — typically an X.509 certificate
containing the device's public attestation key, signed by the
manufacturer's CA — establishes the device's provenance in a way
that can be verified by any relying party with access to the
manufacturer's root certificate. This credential is provisioned at
the factory and stored in TEE-protected or hardware-sealed storage.

Unlike the HUK, the device certificate is exportable by design — it
is presented to relying parties as evidence of identity. Its private
key is not exportable. The certificate is the public claim; the
private key in the secure element is the proof.

---

## §3.2 — The provisioning threat model

The provisioning threat model has three distinct phases, each with
a different attacker and a different set of required controls.

### Phase 1 — Key generation (at the signing HSM)

**Threat:** Weak entropy produces a predictable key. An attacker who
can predict the HUK does not need to extract it from the device.

**Control:** Hardware TRNG output, verified entropy quality, HSM
with certified random number generation (FIPS 140-2 Level 3 or
equivalent). Do not use software PRNGs seeded from system state for
root key generation.

**Threat:** The HSM is compromised and the generated keys are
exfiltrated before injection.

**Control:** HSM with tamper-evident physical protection; role
separation for HSM access (no single operator has both generation
and export authority); audit log of all key operations.

### Phase 2 — Key transfer (from HSM to device)

**Threat:** The injection channel is intercepted. An attacker on the
production line network can observe the key in transit.

**Control:** Encrypted and authenticated transfer channel between
the HSM and the provisioning fixture. The device itself participates
in the key exchange — the key is encrypted to the device's ephemeral
public key, ensuring only the target device can decrypt it. This is
analogous to TLS — the key never appears in plaintext on the wire.

**Threat:** A counterfeit device is inserted into the production
line and receives a legitimate key injection.

**Control:** Device identity verification before key injection —
the provisioning fixture authenticates the device's hardware identity
(silicon serial number, die ID, or equivalent) before transmitting
the key. This authentication must be hardware-rooted, not software-
asserted.

**Threat:** The same key is injected into multiple devices (due to
production line error or deliberate manipulation).

**Control:** Monotonic injection counter at the HSM; key uniqueness
verified before each injection; HSM refuses to re-inject a key that
has already been issued.

### Phase 3 — Key storage on device (post-injection)

**Threat:** The key can be read back from OTP by privileged software.

**Control:** OTP read-back protection enabled immediately after
injection (the "close device" step in most SoC vendor flows). In
the NXP i.MX family this is the HAB close operation; in STM32 it is
the RDP level setting; in other platforms it is an equivalent OTP
lock bit. Once locked, the HUK is readable only to the hardware
crypto engine — not to any software path.

**Threat:** The RPMB authentication key (the key that authenticates
access to the Replay Protected Memory Block partition) is injected
through an unauthenticated channel.

**Control:** The RPMB key must be injected through the TEE using a
signed provisioning TA running in Secure World. An unsigned or
unsigned-firmware-based injection creates a window where the key
can be substituted. The typical derivation:

```
KEY_rpmb = KDF(HUK, "rpmb" || device_id)
```

Derived from the HUK, not independently generated — this binds the
RPMB key to the device identity and eliminates the injection channel
for this particular key.

---

## §3.3 — The key derivation hierarchy

Once the HUK is securely established, the key hierarchy provides
every operational key the platform needs without ever exposing the
root material. The structure is a tree: the HUK is the root; all
other keys are derived from it through a KDF (Key Derivation
Function) that takes the parent key and a diversification input
unique to the intended use.

A representative derivation hierarchy for a TEE-based platform with
encrypted storage:

```
HUK  (in OTP — never exposed to software)
  │
  ├── SSK (Secure Storage Key)
  │     = KDF(HUK, chip_id || "secure_storage")
  │     Scope: platform-wide secure storage
  │     Lives in: TEE Secure SRAM (runtime only, not persisted)
  │
  │     ├── TSK_n (TA Storage Key, one per Trusted Application)
  │     │     = KDF(SSK, TA_UUID)
  │     │     Scope: per-TA file encryption
  │     │     Lives in: TEE Secure SRAM (runtime only)
  │     │
  │     │     └── FEK (File Encryption Key, one per file)
  │     │           = random, wrapped by TSK_n
  │     │           Scope: single file confidentiality
  │     │           Lives in: per-file metadata in RPMB (ciphertext)
  │     │
  │     └── KEY_rpmb (RPMB Authentication Key)
  │           = KDF(HUK, "rpmb" || device_id)
  │           Scope: RPMB access authentication
  │           Lives in: RPMB controller OTP (injected once at factory)
  │
  └── Attestation key pair
        Private key: hardware-sealed (HSM or TEE non-extractable)
        Public key:  in device certificate (exportable)
        Scope: signing attestation tokens
```

**Properties of this hierarchy:**

Compromise is compartmentalised. A compromised FEK exposes one file.
A compromised TSK exposes one TA's files. A compromised SSK exposes
all secure storage — but not the HUK, and not the attestation key.
The HUK itself is never the direct input to any data operation, so
no side channel on any data operation leaks HUK material.

The SSK and TSK are runtime-derived and never persisted — they exist
only in TEE Secure SRAM during the operation that needs them, and
are cleared on power-down. An attacker who obtains a memory dump of
RPMB sees only ciphertext. Without the TSK (which requires the HUK
to derive), the ciphertext is opaque.

**When RPMB is not available: the non-securable NVM case**

The FEK storage row above assumes RPMB — hardware-authenticated NVM
where every write is MAC-verified by the eMMC controller, making
unauthenticated overwrites detectable. RPMB is the standard answer
to the NVM write-protection problem for TEE secure storage on
eMMC-based platforms.

On platforms where RPMB is not available — MRAM-based IoT SoCs, NOR
flash systems, or platforms where the NVM technology is not securable
by hardware design — the wrapped FEK cannot rely on hardware write
protection. An attacker with physical NVM access can overwrite the
stored ciphertext. They cannot read the plaintext without TSK, but
they can substitute an older ciphertext version — a replay attack
that restores an older file state without the TEE detecting the
substitution.

The cryptographic compensation for absent hardware write protection
is nonce-misuse-resistant AEAD at the storage layer. Standard
AES-GCM requires a unique nonce per encryption; under power-loss
conditions on IoT devices, the nonce counter may not be atomically
committed, creating a nonce-reuse risk that AES-GCM-SIV (RFC 8452)
is specifically designed to bound. AES-GCM-SIV limits the damage of
nonce reuse to per-message confidentiality loss rather than full key
compromise, and the authentication tag detects any substitution of
a stored ciphertext with a different ciphertext — including a replay
of an older version.

This is an architectural compensation for absent hardware — it raises
the attack cost significantly but does not provide the same guarantee
as hardware-enforced write protection. The residual risk (an attacker
who can both read and write NVM and has sufficient capability to
perform offline cryptanalysis) must be assessed against the
deployment threat model.

---

## §3.4 — PKI infrastructure: the signing certificate hierarchy

The device's runtime security depends on key derivation. Its ability
to prove its identity to the outside world depends on PKI — a
hierarchy of certificates that connects the device's attestation key
to a root of trust that relying parties recognise.

A typical signing certificate hierarchy for a connected device
platform:

```
Manufacturer Root CA  (offline, HSM-protected)
  │
  ├── Image Signing CA  (online, restricted access)
  │     Issues: firmware image signing certificates
  │     Used by: build system signing pipeline
  │     Validity: annual rotation
  │
  ├── TA Signing CA  (online, restricted access)
  │     Issues: Trusted Application signing certificates
  │     Used by: TA developer signing workflow
  │     Validity: annual rotation
  │
  └── Device Identity CA  (online, factory integration)
        Issues: per-device identity certificates
        Used by: factory provisioning system
        Validity: device operational lifetime
```

**Key design principles for the signing infrastructure:**

The **Root CA private key must never be online.** It lives on an
air-gapped HSM in a physically secured location. The only operation
performed with the Root CA key is signing sub-CA certificates during
rare key ceremony events. An online Root CA is a single point of
failure for the entire fleet's trust hierarchy.

**Role separation is mandatory.** The person who has access to the
signing system must not be the same person who approves what gets
signed. The audit log of all signing operations must be independently
reviewed. A compromised signing operator with no role separation can
sign any image — including malicious ones — with a legitimate key.

**Certificate revocation must be designed before it is needed.**
A signing key compromise is a matter of when, not if, over a
product's lifetime. The mechanism for revoking a compromised
sub-CA certificate and distributing the revocation to all deployed
devices must be part of the provisioning design, not a post-launch
addition. DBX-equivalent revocation lists in the boot chain and OCSP
or CRL distribution for attestation certificates are the standard
mechanisms — but both require OTA delivery infrastructure to be
effective in the field.

---

## §3.5 — Decentralised identity at scale

The PKI model described in §3.4 has a structural weakness at scale:
the Manufacturer Root CA is a single point of failure for the entire
fleet's identity. A Root CA compromise is recoverable — at enormous
cost: every device certificate must be reissued, every relying party
must update their trust anchor, and the window between compromise and
revocation is an exposure period for the entire fleet.

For deployments where the fleet size or the operational life makes
centralised PKI risk unacceptable, blockchain-based identity models
offer an alternative. The core property that makes blockchain relevant
here is immutability: a device identity record written to a
distributed ledger cannot be altered or deleted by any single party,
and does not depend on any single CA's continued trustworthiness.

**The architectural substitution:** Instead of a device certificate
signed by a manufacturer CA, the device's public attestation key is
written to a distributed ledger at provisioning time. Any relying
party can verify the device's public key by querying the ledger —
no central CA is in the verification path. Revocation is also
ledger-based: a compromised device's record is updated with a
revocation flag that all relying parties see simultaneously.

**The tradeoffs:** Blockchain-based identity introduces query latency
(ledger lookup versus local certificate verification), requires
network connectivity at the point of identity verification, and
inherits the governance and consensus risks of the chosen ledger
platform. For devices that operate in intermittently connected
environments, the offline verification capability of PKI is a
meaningful advantage that blockchain does not replicate without
a local cache.

The right choice depends on fleet size, operational life, connectivity
profile, and the organisation's tolerance for CA operational risk.
Both models are in production use. Neither is universally superior.

---

## §3.6 — Zero-touch onboarding and the provisioning lifecycle

Traditional provisioning requires a factory floor provisioning
fixture — a physical station that injects secrets into each device
as it passes through the production line. This model has three
operational constraints:

**It requires physical access at the production site.** For globally
distributed manufacturing, this means either shipping HSMs to
multiple sites (each with its own security requirements) or shipping
unprovisioned devices to a central provisioning facility (adding
logistics cost and a handling step where devices are unprovisioned
and therefore easier to substitute).

**It creates a hard dependency between manufacturing and security.**
A provisioning station failure stops the production line. Security
infrastructure becomes a manufacturing bottleneck.

**It does not scale well to field replacement.** A device that
fails in the field and is replaced must be provisioned — either
at the factory before shipping, or in the field with a mobile
provisioning tool, or via a cloud provisioning protocol.

**Zero-touch onboarding** addresses these constraints by splitting
provisioning into two phases: a minimal factory phase that injects
only the device's bootstrap credential (a certificate that proves
the device is genuine hardware), and a cloud provisioning phase
that delivers the operational credentials over the first authenticated
network connection.

The factory phase is simple and can be performed at the silicon
vendor level — the device identity certificate is provisioned at
wafer or package test, where the HSM infrastructure already exists
for test operations. The operational credentials are delivered by
the cloud platform on first boot, after the device authenticates
using its bootstrap certificate.

This model is used by Azure IoT Hub (Device Provisioning Service),
AWS IoT Core, and similar fleet management platforms. It does not
eliminate the provisioning threat model — it shifts part of the
exposure from the factory floor to the first cloud connection. That
first connection must be authenticated (the cloud platform verifies
the bootstrap certificate), encrypted (TLS 1.3), and protected
against replay (fresh challenge in the provisioning protocol).

---

## What comes next

Part 3 has established how the device's trust foundation is created
at birth. Part 4 covers how that foundation is maintained across the
device's operational life — the OTA update channel, the anti-rollback
mechanism, the SBOM pipeline, and the fail-safe recovery model that
determines whether a failed update leaves the device in a usable
state or a brick.

---

*© 2026 Lei Zhou. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
*Part of the [IoT Security Lifecycle](https://github.com/zlhk100/iot-security-lifecycle) series.*
