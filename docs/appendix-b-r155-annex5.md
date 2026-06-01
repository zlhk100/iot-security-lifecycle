<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# Appendix B — UNECE R155 Annex 5 Practitioner Mapping

> **Scope:** This appendix is for automotive BSP engineers and Tier 1
> suppliers who need to translate R155 Annex 5 threat/mitigation
> language into concrete engineering implementation decisions. It
> covers Annex 5 Part B (vehicle-level mitigations) only — Part C
> (backend/IT mitigations) is outside BSP scope.
>
> **How to use this table:** The "Regulatory text" column contains the
> normative requirement. The "Engineering implementation" column
> contains practitioner annotations — what an embedded security
> engineer must actually build or verify to satisfy the requirement.
> Annotations in **bold** are the author's, not the regulation's.
>
> **Source:** UNECE Regulation No. 155, Annex 5, Part B.
> Annotations derived from BSP engineering experience on
> automotive Cortex-A platforms with OP-TEE TEE stack.

---

## Notation

| Symbol | Meaning |
|---|---|
| M-n | R155 mitigation reference number |
| ★ | High implementation complexity — warrants dedicated design session |
| ⚠ | Dual obligation — security requirement conflicts with or intersects a privacy/regulatory requirement |
| BSP | Directly implementable at BSP / firmware layer |
| OEM | Vehicle manufacturer obligation — BSP provides evidence or capability |

---

## Table B1 — Threats to Vehicle Communication Channels

| Ref | Threat | Mitigation ref | Regulatory text | Engineering implementation |
|---|---|---|---|---|
| 4.1 | Message spoofing (V2X, GNSS, 802.11p) | M10 | Vehicle shall verify authenticity and integrity of messages received | **BSP:** TLS mutual authentication for all network interfaces; V2X message signature verification using IEEE 1609.2 / ETSI ITS-G5 certificate chain; GNSS anti-spoofing (signal plausibility cross-check with IMU) |
| 4.2 | Sybil attack (spoofing multiple vehicles) | M11 | Security controls for storing cryptographic keys (e.g. HSM) | **BSP ★:** V2X pseudonym certificate management requires HSM-backed key storage (EVITA Full HSM for V2X ECUs); pseudonym rotation protocol must not leak long-term identity |
| 5.1 | Code injection via communication channel | M10, M6 | Authenticity/integrity of messages; security by design to minimise injection risk | **BSP:** Signed firmware images (boot chain); input validation at all network-facing interfaces; code/data separation (XN/NX enforcement) |
| 5.2 | Manipulation of vehicle data/code via channel | M7 | Access control to protect system data/code | **BSP:** SELinux MAC policy; signed data blobs; replay protection (nonce or monotonic counter on all command channels) |
| 5.3–5.5 | Overwrite / erasure / introduction of data via channel | M7 | Access control to protect system data/code | **BSP:** Flash write protection (RPMB or equivalent); TEE-enforced access control on secure storage; OTA update signature verification before any write |
| 6.1 | Accepting input from untrusted source | M10 | Verify authenticity and integrity of messages | **BSP:** Source authentication on all external data ingestion paths; reject unsigned or unverifiable input before processing |
| 6.2 | Man-in-the-middle / session hijacking | M10 | Mutual authentication | **BSP:** TLS 1.3 with mutual certificate authentication; certificate pinning for backend channels; session token binding |
| 6.3 | Replay attack (e.g. firmware downgrade via gateway) | — | (No mitigation ref assigned in regulation) | **BSP ★:** Anti-rollback counter in OTP; monotonic version counter verified before any firmware activation; replay window enforcement on command channels |
| 7.1 | Interception / monitoring of communications | M12 | Confidential data in transit shall be protected | **BSP:** TLS encryption for all backend channels; V2X payload encryption where confidentiality required; key material never in plaintext on external interfaces |
| 7.2 | Unauthorised access to files or data | M8 | System design and access control shall prevent unauthorised access to personal or system-critical data | **BSP ⚠:** Secure storage access control (TEE-enforced, SELinux policy); file-level encryption for PII; audit logging of all access to sensitive data partitions — *simultaneously required to respect GDPR consent obligations for any logged PII* |
| 8.1 | Denial of service via garbage data flood | M13 | Detect and recover from DoS | **BSP:** Rate limiting on all external-facing network interfaces; watchdog recovery; service isolation (compromised service cannot cascade) |
| 8.2 | Black hole attack (blocking V2X message relay) | M13 | Detect and recover from DoS | **OEM:** Network topology resilience; **BSP:** heartbeat monitoring, failover detection, alert generation |
| 9.1 | Privilege escalation (unprivileged → root) | M9 | Measures to prevent and detect unauthorised access | **BSP:** Minimal privilege design (no unnecessary SUID binaries, no world-writable paths); seccomp filtering; kernel hardening (CONFIG_SECURITY, CONFIG_AUDIT); authentication logging for privilege transitions |
| 10.1 | Virus / malware via communication media | M14 | Protect against embedded viruses/malware | **BSP:** Signed firmware and application images; measured boot; runtime integrity monitoring (IMA/EVM or equivalent) |
| 11.1 | Malicious internal CAN messages | M15 | Detect malicious internal messages | **BSP:** CAN message authentication (AUTOSAR SecOC or equivalent MAC on safety-critical CAN frames); anomaly detection on CAN bus message rates and IDs |
| 11.2 | Malicious V2X messages (CAM, DENM, etc.) | M10 | Verify authenticity and integrity | **BSP:** ETSI ITS certificate validation; plausibility checking on received position/speed data |
| 11.3–11.4 | Malicious diagnostic / proprietary messages | — | (No mitigation ref assigned) | **BSP:** Diagnostic session authentication (UDS security access); restrict diagnostic commands by session level and authentication state |

---

## Table B2 — Threats to the Update Process

| Ref | Threat | Mitigation ref | Regulatory text | Engineering implementation |
|---|---|---|---|---|
| 12.1 | Compromise of OTA update procedure | M16 | Secure software update procedures | **BSP ★:** TUF/Uptane framework (Director + Image repo separation); image signature verification before activation; delta update integrity; A/B partition scheme with rollback on failed update |
| 12.2 | Compromise of local/physical update procedure | M16 | Secure software update procedures | **BSP:** Local update port authentication (USB/OBD diagnostic session requires authenticated key exchange before accepting update payload) |
| 12.3 | Software manipulated before update process | M16 | Secure software update procedures | **BSP:** End-to-end image signing from build system to activation; hash verification at every transfer step; do not activate until signature verified on-device |
| 12.4 | Compromise of software provider signing keys | M11 | Security controls for cryptographic key storage | **OEM ★:** Signing key custody in offline HSM; key rotation policy; certificate revocation mechanism; **BSP:** DBX-equivalent revocation list checked at activation |
| 13.1 | DoS against update server (prevents critical update rollout) | M3 | Security controls on backend; recovery measures for system outage | **OEM:** Backend resilience; **BSP:** Update retry with backoff; local cache of critical security update if connectivity lost |

---

## Table B3 — Threats from Human Actions

| Ref | Threat | Mitigation ref | Regulatory text | Engineering implementation |
|---|---|---|---|---|
| 15.1 | Social engineering / phishing (user tricked into loading malware) | M18 | Define and control user roles and access privileges; least privilege | **BSP:** User-facing interfaces cannot load unsigned code regardless of user action; capability separation between user and system roles |
| 15.2 | Defined security procedures not followed | M19 | Security procedures defined and followed; logging of actions | **OEM:** Process obligation; **BSP:** Audit logging of security-relevant actions (key operations, configuration changes, diagnostic sessions) with tamper-evident log storage |

---

## Table B4 — Threats from External Connectivity

| Ref | Threat | Mitigation ref | Regulatory text | Engineering implementation |
|---|---|---|---|---|
| 16.1 | Manipulation of remote vehicle functions (remote key, immobiliser, charging) | M20 | Security controls on remote access systems | **BSP:** Remote command authentication and replay protection; challenge-response protocol for immobiliser; command authorisation tied to authenticated session |
| 16.2 | Manipulation of telematics | M20 | Security controls on remote access | **BSP:** Telematics data integrity (signed telemetry payloads); anomaly detection on out-of-range sensor values |
| 16.3 | Interference with short-range wireless / sensors | — | (No mitigation ref assigned) | **BSP:** Sensor plausibility cross-check; rate-of-change validation on safety-critical sensor inputs |
| 17.1 | Corrupted or poorly secured third-party applications | M21 | Software security assessed, authenticated, integrity protected | **BSP ★:** SBOM generation (SPDX/CycloneDX); CVE scanning pipeline on all third-party components; application signing and runtime verification; supply chain integrity |
| 18.1–18.2 | USB or external port used as attack vector / infected media | M22 | Security controls on external interfaces | **BSP:** Disable unused USB device classes in production image; USB gadget mode restricted; media mounted read-only and scanned before execution |
| 18.3 | Diagnostic port (OBD dongle) used for attack | M22 | Security controls on external interfaces | **BSP:** OBD diagnostic session requires authenticated key exchange; restrict privileged diagnostic functions to authenticated sessions only; log all diagnostic sessions |

---

## Table B5 — Threats Targeting Data and Vehicle Identity

| Ref | Threat | Mitigation ref | Regulatory text | Engineering implementation |
|---|---|---|---|---|
| 19.1 | IP / copyright software extraction | M7 | Access control to protect system data/code | **BSP:** Code confidentiality via encrypted firmware partitions; secure boot prevents running modified images |
| 19.2 | Unauthorised access to owner PII (identity, payment, location, address book) | M8 | System design and access control prevent unauthorised access to personal data | **BSP ⚠:** PII stored in TEE-protected or encrypted partition; access control enforced at OS and TEE layer; data minimisation — *GDPR obligation: PII collection requires explicit user consent; location data has heightened protection requirements* |
| 19.3 | Cryptographic key extraction | M11 | Security controls for storing cryptographic keys | **BSP ★:** Keys in EVITA HSM or TEE-backed keystore; keys non-extractable in plaintext; key usage limited by key policy (signing key cannot be used for encryption, etc.) |
| 20.1 | Unauthorised modification of vehicle electronic ID | M7 | Access control to protect system data/code | **BSP:** Vehicle ID stored in write-protected OTP or TEE-sealed storage; modification requires authenticated privileged access; ID integrity verified at boot |
| 20.2 | Identity fraud / impersonation in toll or V2X systems | — | (No mitigation ref assigned) | **BSP:** V2X pseudonym certificate management; long-term identity never exposed in V2X messages |
| 20.3 | Circumventing monitoring (ODR tracker, mileage tampering) | M7 | Access control to protect system data/code | **BSP:** Odometer and driving data stored in tamper-evident log; cross-correlation across data sources for anomaly detection |
| 20.4–20.5 | Falsification of driving data / diagnostic data | — | (No mitigation ref assigned) | **BSP:** Sensor data signed at source where safety-critical; diagnostic data write-protected |
| 21.1 | Unauthorised deletion/manipulation of system event logs | M7 | Access control to protect system data/code | **BSP ★ ⚠:** Security event logs stored in append-only, TEE-protected or RPMB-backed storage; log integrity protected by MAC; deletion requires authenticated privileged access — *simultaneous obligation: logs containing PII (IP addresses, location, biometric access events) are subject to GDPR right-to-erasure requests. Log retention policy must balance forensic requirement (R155 §7.3.7) against erasure right. Design log schema to separate security-relevant metadata (event type, timestamp, outcome) from PII (user identity) where possible* |
| 22.2 | Introduction of malicious software | M7 | Access control to protect system data/code | **BSP:** Signed software only; runtime integrity monitoring; no unsigned code execution path |
| 24.1 | Denial of service (CAN bus flooding, ECU fault injection via high message rate) | M13 | Detect and recover from DoS | **BSP:** CAN rate limiting; CAN message authentication (SecOC); ECU watchdog with safe-state recovery |
| 25.1–25.2 | Unauthorised modification of safety-critical parameters (brake, airbag, charging voltage) | M7 | Access control to protect system data/code | **BSP ★:** Safety-critical configuration parameters write-protected by hardware (OTP or secure storage); modification requires authenticated privileged session; parameter integrity verified at runtime |

---

## Table B6 — Software and Hardware Vulnerabilities

| Ref | Threat | Mitigation ref | Regulatory text | Engineering implementation |
|---|---|---|---|---|
| 26.1–26.3 | Weak / deprecated cryptographic algorithms; short key lengths | M23 | Cybersecurity best practices for software and hardware development | **BSP:** Algorithm policy: AES-256 (symmetric), RSA-3072 or ECC-256 minimum (asymmetric), SHA-256 minimum (hash); no MD5, SHA-1, DES, 3DES in new code; crypto agility design — algorithm upgradeable without full firmware rebuild |
| 27.1 | Hardware or software engineered to enable attack or fail to stop attack | M23 | Cybersecurity best practices | **BSP:** Supply chain integrity; hardware attestation; no undisclosed debug backdoors in production hardware |
| 28.1 | Software bugs as exploitable vulnerabilities | M23 | Cybersecurity testing with adequate coverage | **BSP:** Static analysis (SAST) in CI pipeline; fuzzing of all external-facing parsers and protocol handlers; dynamic analysis (DAST) on network services; penetration testing before release |
| 28.2 | Debug ports / JTAG / development certificates left enabled in production | — | (No mitigation ref assigned in regulation) | **BSP ★:** Production image: JTAG disabled in OTP; debug certificates revoked; developer passwords removed; all open ports audited — *verify via attestation token that running image is production build, not development build* |
| 29.1 | Superfluous internet ports open | — | (No mitigation ref assigned) | **BSP:** Minimal service footprint; firewall default-deny; port scan as part of release validation |
| 29.2 | Circumventing network separation via unprotected gateways | M23 | Cybersecurity best practices for system design | **BSP:** Zero Trust network model — no implicit trust across zone boundaries; authenticated and encrypted inter-ECU communication; gateway access control list enforcement |

---

## Table B7 — Data Loss / Data Breach

| Ref | Threat | Mitigation ref | Regulatory text | Engineering implementation |
|---|---|---|---|---|
| 31.1 ⚠ | PII breach on vehicle ownership transfer (sale or hire) | M24 | Best practices for data integrity and confidentiality; protection of personal data | **BSP ★ ⚠:** User data partition encrypted with device-binding key tied to current owner identity; ownership transfer procedure must include cryptographic key rotation (re-keying wipes previous owner's data without physical erase); factory reset must provably wipe all PII — *this is a hard GDPR right-to-erasure obligation; the engineering implementation must be verifiable and auditable. "Factory reset" that merely marks blocks as available but does not cryptographically invalidate the data key does not satisfy the erasure requirement* |

---

## Table B8 — Physical Manipulation

| Ref | Threat | Mitigation ref | Regulatory text | Engineering implementation |
|---|---|---|---|---|
| 32.1 | Unauthorised hardware added to enable MitM attack | M9 | Measures to prevent and detect unauthorised access | **BSP ★:** Hardware attestation (device identity certificate tied to silicon — PUF or OTP-fused key); bus encryption on external interfaces (SPI, I2C to external components) where practical; ECU boot attestation token verifiable by vehicle manufacturer backend — *counterfeited hardware and device cloning are the primary threat; attestation is the primary mitigation* |

---

## Cross-cutting implementation notes

**The forensic logging / GDPR tension (⚠ items):**

R155 §7.3.7(c) requires forensic capability to analyse cyber-attacks.
GDPR (and equivalent national privacy laws) constrains what data can
be logged and for how long without user consent. These two obligations
are simultaneously mandatory for vehicles sold in the EU. The
engineering resolution is schema separation: log security-relevant
event metadata (event type, timestamp, ECU ID, outcome) in a tamper-
evident log; keep PII (user identity, precise location, biometric
event details) in a separately controlled, consent-gated partition
with a defined retention and erasure policy. The two partitions have
different access controls and different lifecycle requirements. Design
this separation at schema definition time — retrofitting it after the
log format is established is significantly more expensive.

**Cryptographic module compliance (R155 §7.3.8):**

R155 requires cryptographic modules to be "in line with consensus
standards." For BSP engineers this means: crypto libraries used in
the TEE, in the kernel, and in user space must be from audited,
maintained implementations (mbedTLS, OpenSSL, BoringSSL — with
current CVE status verified). In-house or unmaintained crypto
implementations require explicit justification in the security target.
Document which crypto library serves each function: TEE crypto
operations, kernel IPsec/TLS, user-space application crypto, and
hardware accelerator integration.

**EVITA HSM tiers and R155 applicability:**

EVITA defines three HSM tiers for automotive ECUs:
- **EVITA Full:** V2X applications requiring highest assurance
  (pseudonym certificates, V2X message signing)
- **EVITA Medium:** Domain controllers (powertrain, chassis)
  requiring hardware key protection
- **EVITA Light:** Sensors and actuators requiring lightweight
  integrity protection

R155 does not mandate a specific EVITA tier. It requires that
cryptographic key protection be "in line with consensus standards"
(§7.3.8) and that mitigations be proportionate to identified risks
(§7.3.4). The EVITA tier selection should be driven by the TARA
(Threat Analysis and Risk Assessment) result for each ECU, not
applied uniformly across the vehicle.

---

*© 2026 Lei Zhou. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
*Part of the [IoT Security Lifecycle](https://github.com/zlhk100/iot-security-lifecycle) series.*
