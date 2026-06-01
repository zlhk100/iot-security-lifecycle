<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# Part 7 — Extending Trust to the Edge AI Stack

> **Series context:** This part extends the framework from Parts 1–5
> into the edge AI context. It is deliberately shorter than preceding
> parts — the engineering discipline for edge AI security is less
> settled than for the topics in Parts 1–6, and this series will not
> paper over that gap. What follows is a framework for thinking about
> the problem, not a solved solution set.

---

## A new principal the framework did not address

The principles in Parts 1–5 were developed for a world where the
software running on a device was authored by its manufacturer and
updated through a controlled channel. The asset table in Part 5
captures firmware binaries, cryptographic keys, and user data. The
boot chain in Part 2 verifies code signed by the device manufacturer.
The OTA channel in Part 4 delivers updates whose provenance is
controlled by the same signing infrastructure.

Edge AI introduces a new principal that none of these mechanisms
were designed to cover: **model weights**. Model weights are authored
by a third party (a model developer, a cloud training pipeline, or
an open source community). They are updated independently of the
firmware — a model can change without a firmware update, and a
firmware update does not imply a model change. They execute on
dedicated hardware (NPU, GPU, DSP) that the general-purpose OS
cannot fully observe. And their security properties — integrity,
provenance, confidentiality — are not covered by any of the existing
trust mechanisms.

The trust architecture must extend to cover this new principal.
The question is how, and what the correct abstraction is.

---

## §7.1 — The edge inference threat model

Applying the three-class taxonomy from Part 5 to the edge AI context
produces a threat model that is partially familiar and partially new.

**Remote attackers** targeting the inference pipeline can attempt
model poisoning (corrupting training data or the model update channel
to produce a model that behaves incorrectly on specific inputs),
adversarial inputs (crafting inputs that cause the model to produce
wrong outputs without the model itself being modified), and model
extraction (querying the model to reconstruct its weights through
API access). The first two are model-level attacks that the device's
network security controls do not prevent — they operate through the
legitimate inference API.

**Local attackers** with code execution on the device can attempt
weight extraction from NPU memory, cache timing side-channels on
the inference computation, and substitution of model weights with a
crafted version that passes integrity checks but behaves differently
on specific inputs. Weight extraction from NPU-local memory is the
embedded equivalent of key extraction from an unprotected keystore.

**Physical attackers** with hardware access can target model weights
in flash storage, probe the NPU-to-DRAM bus during inference to
observe weight values, and fault-inject the model loading sequence
to bypass integrity verification.

The novel element compared to the Parts 1–5 threat model: **the
model itself is an asset that has not previously appeared in embedded
security thinking.** It is not a key (not used for cryptographic
operations), not a certificate (not used for identity), and not
firmware (not executed as CPU instructions in the conventional
sense). It is a parameterisation of behaviour — and its integrity
determines what the device does, which in safety-critical contexts
determines what physical systems it controls.

---

## §7.2 — Three properties model weights require

Treating model weights as a first-class asset in the asset table
from Part 5 requires defining their security properties explicitly.
Three are necessary:

**Integrity:** The weights executing on the device must be the
weights that were trained, validated, and authorised for deployment.
A substitution attack that replaces authorised weights with crafted
weights — even weights that produce correct outputs on most inputs —
is a firmware tampering attack at the model layer. The control is
the same as for firmware integrity: cryptographic signature
verification at load time, rooted in the same signing infrastructure
as the firmware chain.

**Provenance:** The device must be able to demonstrate which version
of which model is currently loaded — to a remote verifier, to an
OTA management system, and potentially to a regulatory auditor.
For automotive ADAS models, knowing which model version made a
decision is a safety and liability requirement. For industrial
anomaly detection models, knowing which model version was active
during an incident is a forensic requirement. The mechanism is
measurement — extending the DICE/TPM measured boot concept to
include model version hashes alongside firmware measurements.

**Confidentiality (where required):** Not all edge AI models require
confidentiality — open-source models deployed on edge devices have
no confidentiality requirement. Proprietary models (computer vision
models trained on proprietary datasets, models that encode
competitive IP, models used in security functions) require
protection equivalent to what Part 3 describes for firmware images:
encrypted storage, decryption only inside a protected execution
environment, no plaintext exposure on external interfaces.

---

## §7.3 — SPDM and accelerator-to-host attestation

The attestation model from Part 2 covers the device's software
stack — what firmware and OS are running. It does not cover what is
happening inside an NPU or GPU accelerator that operates with its
own firmware, its own memory, and its own execution context.

SPDM (Security Protocol and Data Model, DMTF DSP0274) is the
protocol that fills this gap. Originally developed for datacenter
contexts (PCIe device attestation, CXL memory device identity),
SPDM is increasingly relevant at the edge:

- AUTOSAR AP release 22-11 mandates SPDM for secure ECU-to-ECU
  communication in zonal vehicle architectures — the same protocol
  that attests a PCIe GPU in a server attests an automotive ECU
  to its domain controller
- Edge AI accelerators (NPUs with their own firmware) can implement
  SPDM to attest their firmware state to the host processor —
  providing the host with evidence that the NPU is running trusted
  firmware before model weights are loaded
- The RATS framework (RFC 9334) extends naturally to cover composite
  attestation: a device that includes an NPU can produce an
  attestation token that covers both the host firmware state and
  the NPU firmware state

The engineering implication: model weight integrity verification
and NPU firmware attestation are two separate but complementary
controls. Weight integrity (is this the authorised model?) requires
a signing infrastructure. NPU attestation (is the execution
environment trustworthy?) requires SPDM or equivalent. Both are
needed for the full inference integrity claim.

---

## §7.4 — The open problem: inference result integrity

Weight integrity and NPU attestation answer the question "was the
right model loaded into a trustworthy environment?" They do not
answer "did the inference produce the correct result for this input?"

Inference result integrity — verifiable proof that a given output
was produced by a specific model from a specific input, without
tampering — is an open research and engineering problem. For
safety-critical applications (ADAS decision outputs, medical
diagnostic outputs, industrial control decisions), the inability
to verify inference results creates a gap in the trust chain: the
device can prove what model it loaded, but cannot prove what the
model concluded.

Approaches under active development include:
- TEE-based inference (running the model inside a TrustZone or
  equivalent Secure World, producing a signed result) — feasible
  for small models, not currently practical for large vision models
  on constrained hardware
- Cryptographic proof systems (zero-knowledge proofs of inference)
  — mathematically sound, computationally impractical at the scale
  of real edge inference workloads as of 2026
- Hardware-assisted audit trails (NPU execution logs signed by NPU
  firmware, attested via SPDM) — operationally feasible but
  requires NPU vendor support

This is an honest open problem. The framework from this series
provides the right vocabulary — asset identification, attacker
class, enforcement mechanism, blind spot — for reasoning about
solutions as they mature. It does not provide the solution.

---

## Forward pointers

**Confidential computing at the edge:** The principles behind server-
side confidential computing (TEE-protected inference, encrypted
memory, remote attestation of the compute environment) are being
applied to edge platforms. For the server-side trust extension of
the same principles developed in this series, see the author's OCP
confidential computing analysis:
[Confidential Computing and OCP (Parts 1 and 2)][ocp-posts].

**CCA (Confidential Compute Architecture):** ARM CCA introduces
Realms — isolated execution environments below the hypervisor — that
can host inference workloads on Cortex-A platforms with hardware-
enforced isolation from the OS and hypervisor. This extends the
isolation framework from Part 2 to cover a new execution domain.
CCA Realm attestation is a natural extension of the PSA attestation
model described in Part 2.

[ocp-posts]: https://medium.com/@zlhk100

---

*© 2026 Lei Zhou. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
*Part of the [IoT Security Lifecycle](https://github.com/zlhk100/iot-security-lifecycle) series.*
