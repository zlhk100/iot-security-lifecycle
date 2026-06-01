<!--
SPDX-License-Identifier: CC-BY-4.0
Copyright © 2026 Lei Zhou
https://creativecommons.org/licenses/by/4.0/
-->

# Appendix A — Regulatory Quick Reference

> **Scope:** Security-relevant regulations and standards applicable
> to connected devices. Organised by domain. For each entry: the
> instrument, its geographic/sector scope, the key engineering
> obligation it creates, and the series section most relevant to it.
>
> This table is a navigation aid, not legal advice. Regulatory
> applicability depends on product category, market, and use case.
> Verify against the current version of each instrument.

---

## Cross-domain

| Instrument | Scope | Key engineering obligation | Series section |
|---|---|---|---|
| **EU Cyber Resilience Act (CRA)** Regulation 2024/2847 | All products with digital elements sold in EU market. In force Dec 2024; compliance deadline Dec 2027 | Secure by design, vulnerability handling lifecycle, security update mechanism, SBOM | Parts 0, 4 |
| **NIST SP 800-193** Platform Firmware Resiliency Guidelines | US federal procurement; widely adopted as de facto standard | Protect / Detect / Recover triad for platform firmware; RoT requirements | Parts 1, 2 |
| **NIST SP 800-218** Secure Software Development Framework (SSDF) | US federal procurement; CRA technical reference | Software supply chain transparency; SBOM as compliance artifact | Part 4 |
| **Common Criteria (ISO/IEC 15408)** | International; voluntary but required by many procurement frameworks | Evaluated assurance level (EAL) for security functions; Protection Profile compliance | Part 1 |
| **FIPS 140-2 / 140-3** | US federal procurement; cryptographic modules | Cryptographic module validation; physical security levels (L1–L4) | Parts 1, 3 |

---

## Automotive

| Instrument | Scope | Key engineering obligation | Series section |
|---|---|---|---|
| **UNECE R155** (UN Regulation No. 155) | Road vehicles in 60+ contracting parties (EU, UK, Japan, South Korea, Australia). Mandatory for all new vehicles from July 2024 | Cybersecurity Management System (CSMS); Annex 5 threat/mitigation implementation | Parts 0, 2, 4; Appendix B |
| **UNECE R156** (UN Regulation No. 156) | Same jurisdictions as R155 | Software Update Management System (SUMS); OTA security; anti-rollback; fail-safe recovery | Part 4; Appendix B |
| **ISO/SAE 21434** Road Vehicles — Cybersecurity Engineering | International standard; R155 references as suitable framework | TARA methodology; cybersecurity lifecycle; supplier obligations | Part 5 |
| **ISO 24089** Road Vehicles — Software Update Engineering | International standard | Software update process requirements; complements R156 | Part 4 |
| **EVITA** (EU project; de facto automotive HSM standard) | Automotive HSMs in EU/international OEM supply chains | HSM tier selection (Full/Medium/Light) based on TARA; key protection requirements | Parts 1, 3; Appendix B |
| **SAE J3101** Hardware Protected Security Environment | US/international automotive | Hardware security environment requirements for automotive ECUs | Part 1 |

---

## Industrial / OT

| Instrument | Scope | Key engineering obligation | Series section |
|---|---|---|---|
| **IEC 62443** Industrial Automation and Control Systems Security | Industrial control systems globally; widely adopted by sector regulators | Security levels (SL 1–4); zone and conduit model; component security requirements | Parts 0, 6 |
| **NERC CIP** North American Electric Reliability Corporation Critical Infrastructure Protection | US/Canada/Mexico bulk electric system | Physical and cyber security for bulk electric system assets | Part 6 |

---

## Medical devices

| Instrument | Scope | Key engineering obligation | Series section |
|---|---|---|---|
| **FDA Cybersecurity Guidance** (2023) | Medical devices for US market | Threat modelling; SBOM; OTA patching capability; disclosure obligations | Parts 4, 5 |
| **EU MDR / IVDR** with ENISA guidance | Medical devices in EU market | Cybersecurity by design; post-market surveillance including cybersecurity | Parts 0, 4 |
| **HIPAA Security Rule** | US healthcare — devices handling protected health information | Data encryption, access controls, audit logging for PHI | Parts 3, 5 |

---

## Cryptographic standards

| Instrument | Scope | Key engineering obligation | Series section |
|---|---|---|---|
| **NIST PQC standards** FIPS 203 (ML-KEM), FIPS 204 (ML-DSA), FIPS 205 (SLH-DSA) | Effective 2024; migration timeline varies by sector | Post-quantum algorithm migration; crypto agility design | Part 5 §5.4 |
| **CNSA 2.0** (NSA Commercial National Security Algorithm Suite 2.0) | US national security systems; influential beyond | PQC algorithm mandates from 2025; RSA/ECDSA phase-out by 2033 | Part 5 §5.4 |
| **ETSI TS 103 744** Quantum-Safe Cryptography | EU/international | Algorithm selection guidance for migration | Part 5 §5.4 |

---

## Attestation and identity

| Instrument | Scope | Key engineering obligation | Series section |
|---|---|---|---|
| **IETF RFC 9334** RATS (Remote ATtestation procedureS) | Internet standards; PSA attestation implements this model | Attester/Verifier/Relying Party model; Evidence format | Parts 2, 7 |
| **IETF EAT** (Entity Attestation Token, RFC 9528) | Internet standards; PSA IAT based on EAT | Attestation token format and claims | Part 2 |
| **TCG DICE** Layering Architecture | IoT/constrained device attestation | Compound Device Identifier (CDI) derivation; per-layer measurement | Parts 1, 2 |
| **DMTF SPDM** (DSP0274) Security Protocol and Data Model | PCIe/CXL devices; automotive (AUTOSAR AP 22-11); edge AI accelerators | Device-to-device authentication and attestation; measurement exchange | Parts 2, 7 |
| **PSA Certified** (JSADEN002/012/019) | ARM ecosystem; IoT/embedded | L1/L2/L3 security levels; RoT requirements; ITS/PS/Attestation APIs | Parts 1, 2, 3 |

---

*© 2026 Lei Zhou. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
*Part of the [IoT Security Lifecycle](https://github.com/zlhk100/iot-security-lifecycle) series.*
