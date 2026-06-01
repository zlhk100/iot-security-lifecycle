<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# Part 4 — Secure OTA and Fleet Lifecycle

> **Series context:** Parts 1–3 established the trust foundation at
> device birth. This part covers how that foundation is maintained
> across the device's operational life. OTA security requirements
> appear across multiple domains — UNECE R156 for road vehicles,
> IEC 62443 for industrial systems, NIST SP 800-193 for platform
> firmware resilience. The engineering principles are the same
> regardless of domain.

---

## The controlled vulnerability

Every update is a controlled vulnerability. To install new firmware,
the device must accept data from an external source, write it to
executable storage, and transfer control to it. These are precisely
the operations that every other security control on the device is
designed to prevent.

OTA security is the discipline of making this necessary exception
safe — not by removing the exception, but by making it auditable,
reversible, and resistant to substitution.

What makes OTA harder than provisioning is not the cryptography —
signature verification is a solved problem. What makes it hard is
the combination of three properties that must hold simultaneously
across an indefinite operational period: the update must be
authenticated (only authorised images install), atomic (power loss
at any point leaves the device in a recoverable state), and
reversible (a bad update can be undone). Each property is
individually achievable. All three together, at fleet scale, with
heterogeneous hardware and unpredictable field conditions, is the
real engineering problem.

---

## §4.1 — Atomicity: the problem documentation does not solve

Framework documentation covers signature verification thoroughly.
It covers transport security. It does not cover the hardest OTA
engineering problem: atomicity.

The question is simple to state: what is the minimal atomic unit
of an update, and what is the guaranteed device state at every
possible power-loss point within that unit?

A/B partitioning is the standard answer — maintain two complete
firmware copies, write the new image to the inactive partition while
the active partition continues running, switch only after the write
is complete and verified. Power loss during the write leaves the
active partition untouched. This is correct and well understood.

What documentation does not resolve is the **confirmation window** —
the interval between the bootloader switching to the new partition
and the application confirming the new firmware is healthy. This
window is where most production A/B failures occur.

The design decision has two options, each with a different failure
mode:

**Option A — Confirm before incrementing the anti-rollback counter.**
The new firmware runs, confirms health, then the anti-rollback
counter is incremented. If power is lost before confirmation, the
bootloader reverts to the previous partition on the next boot. Clean
recovery. Residual risk: the new firmware runs in an unconfirmed
state during the window — a power cycle at any point in the window
reverts the update, meaning the patch has not been applied.

**Option B — Increment counter before confirmation.**
Anti-rollback is immediate. Residual risk: if the new firmware is
faulty and cannot confirm, the device cannot revert — the previous
firmware is now below the rollback counter. The device is stuck on
a non-confirming image.

Neither option is universally correct. The choice depends on which
failure mode is less acceptable: a patched vulnerability temporarily
reverting (Option A) or a device stuck on a faulty image (Option B).
This is a threat model decision, not a technical one — and it must
be made explicitly before the OTA system is designed, not after
the first field failure.

The confirmation window duration is equally under-specified in
documentation. Too short and slow-initialising devices are bricked.
Too long and the unconfirmed exposure window grows. The correct
value is platform-specific and must be measured on the actual
hardware under worst-case initialisation conditions — not copied
from a reference implementation.

---

## §4.2 — The offline key question: security versus operational tempo

TUF and Uptane separate signing responsibilities across roles with
different online/offline status. The Root key — offline — authorises
all other keys. Image Targets keys — offline — sign metadata
describing admitted images. Director Targets keys — online — create
deployment assignments referencing already-admitted images. Timestamp
and Snapshot keys — online — provide freshness and consistency
guarantees.

The practical question this raises: if image keys are offline, how
does a security patch reach the fleet quickly when a zero-day is
published?

The answer requires a key ceremony — a controlled process where the
offline HSM is brought into a secured environment, new image metadata
is signed, and the HSM returns to offline storage. For routine
updates, ceremony scheduling adds hours to days between "build
complete" and "admitted to Image repository." The Director then
deploys from the admitted image using online keys — no further
ceremony required.

The insight that framework documentation does not surface: **the
emergency ceremony must be designed before it is needed.** Most teams
design the routine ceremony process and treat the emergency process
as a future problem. A zero-day that requires immediate fleet-wide
patching is the worst possible time to discover that no emergency
ceremony process exists, no quorum is available, and the offline HSM
is in a location that requires 48 hours of lead time to access.

Designing the emergency process means: defining a reduced quorum
for the offline key ceremony, identifying who is authorised to
convene it, pre-staging the HSM in a location accessible within the
required response window, and testing the process at least annually.
This is an organisational design problem as much as a technical one.

---

## §4.3 — Anti-rollback: the edge case that matters

The mechanism is straightforward: a monotonic counter in OTP,
checked by the bootloader before activating any image. An image
whose version number is below the counter is rejected.

The edge case that matters is the intersection of anti-rollback with
A/B recovery. If Option B from §4.1 is chosen — counter incremented
before confirmation — and the new image is faulty, the device cannot
revert. The previous image is now below the counter value. The device
is in a state that requires out-of-band recovery.

This is not hypothetical. It is the failure mode that makes OTA
engineers conservative about Option B. The consequence of getting
it wrong is not a security vulnerability — it is a fleet of devices
requiring physical servicing.

The engineering discipline: test the counter-increment/revert
interaction explicitly on the target hardware, under power-loss
simulation, before any production OTA deployment. This test is
almost never in reference implementations and almost always catches
something.

---

## §4.4 — SBOM: value is inversely proportional to latency

A Software Bill of Materials has operational security value only
when the time from "CVE published" to "affected devices identified"
is short enough to act before exploitation.

An SBOM reconstructed retrospectively after a CVE is published
provides almost no operational value — by the time the affected
components are identified across the fleet, the vulnerability has
been exploited. The discipline is not "produce an SBOM" but "make
SBOM generation a zero-overhead byproduct of the build, stored and
indexed at build time."

Yocto Project generates SPDX artifacts natively from the build.
Syft generates SPDX or CycloneDX from container images. The
incremental cost of SBOM generation in a CI pipeline is negligible.
The operational cost of not having it — reconstructing component
inventories under time pressure during an active incident — is
not.

The SBOM's value is realised in the pipeline that consumes it: SBOM
stored at build time, indexed by device and firmware version in the
fleet management system, queried automatically against a continuously
updated CVE feed. The bottleneck in this pipeline is not SBOM
generation — it is OTA deployment speed after the affected fleet
segment is identified.

---

## What this part does not cover

Secure transport for OTA delivery (TLS 1.3, mutual authentication)
is well-covered by existing documentation and is not repeated here.
The full TUF and Uptane specifications are at tuf.io and uptane.org
respectively — this part addresses the design decisions those
specifications leave to implementers, not the specifications
themselves.

---

*© 2026 Lei Zhou. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
*Part of the [IoT Security Lifecycle](https://github.com/zlhk100/iot-security-lifecycle) series.*
