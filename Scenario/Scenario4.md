# SC5 — Scenario 4: Direct eInvoicing between Business Wallets

**WE BUILD consortium | WP2 — UC SC5**

| | |
|---|---|
| **Date** | 2026-04-29 |
| **Version** | 0.7 |
| **Status** | Draft |
| **Author(s)** | Maarten Boender - Sphereon |

> **Part of the SC5 eInvoicing specification suite.** Read [Introduction.md](Introduction.md) for common concepts, roles, attestations and abbreviations.

---

## Index

1. [Introduction](#1-introduction)
2. [Pre-conditions](#2-pre-conditions)
3. [Main flow](#3-main-flow)
   - 3.1 [eInvoice preparation and sending by Supplier](#31-einvoice-preparation-and-sending-by-supplier)
   - 3.2 [eInvoice verification by Buyer](#32-einvoice-verification-by-buyer)
   - 3.3 [eInvoice with evidence](#33-einvoice-with-evidence)
4. [Detailed scenario flow](#4-detailed-scenario-flow)
5. [Additional flows](#5-additional-flows)
6. [Challenges and barriers](#6-challenges-and-barriers)
7. [Working assumptions](#7-working-assumptions)
   - 7.1 [Transfer protocol: OID4VP/DCQL](#71-transfer-protocol-oid4vpdcql)
   - 7.2 [(Q)ERDS — possible additional pilot track](#72-qerds--possible-additional-pilot-track)
- [Annex 1 — Requirements for scenario roles](#annex-1--requirements-for-scenario-roles)
- [Annex 2 — Open issues and decisions log](#annex-2--open-issues-and-decisions-log)

---

## 1. Introduction

Scenario 4 is a cross-border pilot of direct eInvoice exchange where a Supplier issues an eInvoice Attestation and delivers it using their Business Wallet directly to a Buyer Business Wallet, which verifies invoice integrity, supplier identity, and authorization.

Unlike Scenarios 1–3 and 5, this scenario does **not** use the Peppol 4-corner model. Instead, the transport layer is OID4VP/DCQL in a machine-to-machine context, with wallet-to-wallet direct delivery.

The eInvoice Attestation is a verifiable representation (Reference Attestation) of an invoice exchange event, binding invoice content (via a payload hash) to a cryptographic signature associated with the Supplier's legal-person identity. It is used by a Buyer to:

- verify invoice integrity,
- verify the authenticity and validity of referenced evidence (where applicable), and
- optionally approve a payment request (not piloted in this scenario; part of PA4).

This scenario is designated **MVP**.

The scenario is based on the following documentation:

- WE BUILD SC5 Stock Taking document (V1.0, 11/12/2025)
- ARF / EUDI Wallet specifications (exact version to be confirmed by WP2/4)
- WP2 Semantics outputs for eInvoicing attestations (to be referenced when available)
- Domain standards: EN 16931 (semantic model) and Peppol BIS 3.0 (syntax binding), and ViDA-relevant guidance (where applicable)
- eInvoice Attestation specification ([v0.5 final draft](https://portal.webuildconsortium.eu/group/11/files/6915/collabora-online/edit/3850))

---

## 2. Pre-conditions

- Buyer and Supplier each have an operational Business Wallet instance, capable of issuing, holding and verifying attestations.
- Each wallet instance SHALL be a valid We Build or European Business Wallet bound to a valid Legal Person identity credential/attestation e.g. EU Company Certificate (EUCC) or equivalent.
- Trust anchors are available for verifying legal entity identity binding and signing keys (registry/federation/EDD evidence to be defined by WP4 Trust Registry Infrastructure group).
- The Buyer has a policy decision on inbound acceptance based on a number of checks:
  - Verify signature validity on eInvoice attestation
  - Verify Supplier EUCC/LPID binding (identity valid)
  - Verify Buyer (buyer on invoice = receiving wallet EUCC/LPID)
  - Recompute and verify invoicePayloadHash
- The eInvoice attestation profile (semantics + integrity rules) is agreed and implementable by participating wallet providers.

To establish trust, either the Approved Supplier attestation (as defined in Scenario 1) must be used, or suppliers must be added to a Trust Registry (LoTE/DDE). This decision rests with the WP4 Architecture and/or Trust Registry Infrastructure working groups.

---

## 3. Main flow

The overview below shows the high-level trust setup between Buyer, Trust Registry and Supplier before direct delivery:

```mermaid
sequenceDiagram
    participant Buyer
    participant Trust_Registry
    participant Supplier

    Buyer ->> Trust_Registry: Create a Trusted List for preferred suppliers
    Supplier ->> Buyer: send eInvoice
    Buyer ->> Buyer: review eInvoice
    Supplier ->> Trust_Registry: List supplier
```

### 3.1 eInvoice preparation and sending by Supplier

```mermaid
sequenceDiagram
    participant Supplier_ERP as Supplier_ERP
    participant Supplier_Wallet as Supplier_Wallet
    participant Buyer_Wallet as Buyer_Wallet
    participant Trust_Registry as Trust_Registry
    participant Status_Service as Status_Service
    participant Evidence_Store as Evidence_Store

    rect rgb(220, 230, 245)
        Note over Supplier_ERP, Evidence_Store: Invoice preparation and attestation issuance (Supplier)
        Supplier_ERP ->> Supplier_ERP: Prepare eInvoice (EN16931)
        Note over Supplier_ERP: Canonicalize payload + compute invoicePayloadHash (rulebook-defined)
        Supplier_ERP ->> Supplier_Wallet: Submit payload + metadata
        Supplier_Wallet ->> Supplier_Wallet: Create eInvoice Attestation
        Supplier_Wallet ->> Supplier_Wallet: Sign eInvoice Attestation
    end

    rect rgb(220, 245, 220)
        Note over Supplier_ERP, Evidence_Store: Direct delivery/presentation (OID4VP/DCQL profile)
        Supplier_Wallet ->> Buyer_Wallet: Send/Present eInvoice Attestation + invoice payload + optional evidence refs
    end
```

### 3.2 eInvoice verification by Buyer

```mermaid
sequenceDiagram
    participant Supplier_ERP as Supplier_ERP
    participant Supplier_Wallet as Supplier_Wallet
    participant Buyer_Wallet as Buyer_Wallet
    participant Trust_Registry as Trust_Registry
    participant Status_Service as Status_Service
    participant Evidence_Store as Evidence_Store

    rect rgb(245, 235, 220)
        Note over Supplier_ERP, Evidence_Store: Verification (Buyer)
        Buyer_Wallet ->> Trust_Registry: Resolve Supplier issuer metadata (Trust registry)
        Trust_Registry -->> Buyer_Wallet: Trust metadata (issuer authorization)
        Buyer_Wallet ->> Status_Service: Check status of Supplier authorization (revocation/expiry/suspension)
        Status_Service -->> Buyer_Wallet: Status result
        Buyer_Wallet ->> Buyer_Wallet: Verify signature on eInvoice Attestation
        Buyer_Wallet ->> Buyer_Wallet: Verify Supplier EUCC/LPID binding (identity valid)
        Buyer_Wallet ->> Buyer_Wallet: Verify Buyer (invoice buyer = receiving wallet ID)
        Buyer_Wallet ->> Buyer_Wallet: Recompute invoicePayloadHash

        alt [All checks pass]
            Buyer_Wallet ->> Buyer_Wallet: ACCEPT (store for audit)
            Buyer_Wallet -->> Supplier_Wallet: Optional msg (accepted)
        else [Any check fails]
            Buyer_Wallet ->> Buyer_Wallet: REJECT or QUARANTINE (error code)
            Buyer_Wallet -->> Supplier_Wallet: Optional msg (rejected/quarantined)
        end
    end
```

### 3.3 eInvoice with evidence

```mermaid
sequenceDiagram
    participant Supplier_ERP as Supplier_ERP
    participant Supplier_Wallet as Supplier_Wallet
    participant Buyer_Wallet as Buyer_Wallet
    participant Trust_Registry as Trust_Registry
    participant Status_Service as Status_Service
    participant Evidence_Store as Evidence_Store

    alt [Evidence references present]
        Buyer_Wallet ->> Evidence_Store: Retrieve evidence by reference(s)
        Evidence_Store -->> Buyer_Wallet: Evidence artefacts
        Buyer_Wallet ->> Buyer_Wallet: Verify evidence integrity (hash/signature) + validity
    else [No evidence references]
        Note over Buyer_Wallet: Skip evidence verification
    end
```

---

## 4. Detailed scenario flow

*(To be completed during the specification phase, building on the main flows above.)*

---

## 5. Additional flows

*(To be completed during the specification phase based on partner input.)*

---

## 6. Challenges and barriers

**Onboarding or Discovery:** how do Supplier and Buyer find each other's Business Wallets?

Answer: The [ETSI TS 119 602](https://www.etsi.org/deliver/etsi_TS/119600_119699/119602/01.01.01_60/) specification describes the List of Trusted Entities (LoTE). It offers the necessary data and describes the mechanisms for Discovery and Verification.

For now the transport protocol will be OID4VP & DCQL. Depending on the upcoming Implementing Acts we may also need to pilot another transport protocol (QERDS?).

**Clarity of role:**

- Supplier issues eInvoice attestation
- Buyer holds eInvoice attestation
- Buyer Business Wallet technically verifies the Invoice attestation and Accepts (or not)
  - Integrity
  - Trust establishment
  - Third party may verify and accept eInvoice attestation (e.g. software provider checks integrity rules)
    - eInvoice xml integrity validation
  - Note: buyer is equipped to further process the eInvoice information / paves way for automation (out of scope)

---

## 7. Working assumptions

### 7.1 Transfer protocol: OID4VP/DCQL

**Transfer protocol: OID4VP/DCQL (pilot choice) in a Machine 2 Machine context**

OID4VP/DCQL is primarily about the semantics and security of verifiable presentations: what the verifier asks for (constraints), how the holder proves it (presentation), and how replay and audience binding are handled (nonce/audience/state). It does not require that the verifier (buyer) "starts the business process"; it requires that the verifier (buyer) controls the request object (or at least the parameters that make the response verifiable and non-replayable).

In our pilot, the Supplier initiates sending because it already has the Buyer's contact data (wallet address + API endpoint). That can still fit OID4VP/DCQL if we implement a short handshake where the Supplier fetches the Buyer's request object before submitting the presentation.

A workable supplier-initiated flow looks like this:

- Supplier wallet already has Buyer verifier endpoint (from contact data).
- Supplier wallet calls Buyer endpoint to obtain a fresh Presentation Request (OID4VP Request Object) that contains:
  - DCQL query constraints (what Buyer will accept),
  - nonce + audience + expiry window,
  - state/correlation identifier and the submission endpoint.
- Supplier wallet submits the invoice package as an OID4VP response (presentation) to the Buyer endpoint:
  - eInvoice Attestation,
  - invoice payload (or reference),
  - optional evidence references (or included evidence).
- Buyer verifies:
  - the presentation response integrity (signature/holder binding as defined),
  - the supplier's legal-person identity binding (EUCC/LPID or equivalent),
  - invoice payload hash binding and any evidence integrity/validity checks.

This is "Supplier-initiated delivery" operationally, while keeping "Verifier-defined requirements" cryptographically and semantically.

### 7.2 (Q)ERDS — possible additional pilot track

It may be that the upcoming Implementing Acts defined by the European Commission may require use of the (Q)ERDS protocol for exchange of data.

ERDS/QERDS is about the delivery channel and the evidentiary value of sending/receiving, not about "what claims are requested" in the same way. Conceptually:

- ERDS provides registered electronic delivery evidence (proof of sending and proof of receipt/delivery) with integrity protection.
- QERDS is the qualified eIDAS variant, delivered as a qualified trust service, with stronger and more harmonised legal effects for the evidence.

In eInvoicing, we should use ERDS/QERDS when the delivery evidence itself must be strong and dispute-proof ("who sent what to whom and when"), potentially because the Commission's upcoming Business Wallet implementing decisions may require (qualified) registered delivery capabilities for certain exchanges.

These are not mutually exclusive with OID4VP/DCQL. Two common positions are:

- Choose OID4VP/DCQL when you primarily need interoperable wallet/presentation semantics (constraints, selective disclosure, consistent verification), and you can rely on ordinary HTTPS/API transport plus application logs for delivery evidence.
- Add ERDS/QERDS when you also need legally stronger delivery evidence. ERDS/QERDS can wrap the same payload (eInvoice Attestation + invoice data) as the delivery channel, while OID4VP/DCQL governs the internal presentation semantics.

**Decision guidance**

| Decision factor | OID4VP/DCQL | ERDS | QERDS |
|----------------|-------------|------|-------|
| Primary purpose | Interoperable presentation semantics and verifier constraints | Registered delivery evidence (non-qualified) | Highest-assurance registered delivery evidence (qualified) |
| Who initiates operationally | Buyer provides request object. Supplier initiates. | Supplier initiates; delivery service transports | Supplier initiates; qualified delivery service transports |
| What you can prove best | The buyer received a verifiable presentation that satisfies buyer-defined constraints | Sent/received with integrity and evidence of delivery | Same as ERDS, but with qualified evidentiary strength |
| Dispute profile | Normal (as is today) | Elevated risk of dispute | High-value / High-risk / Regulated |
| Dependency on buyer endpoint | Yes (needs request object and submission endpoint) | Less strict (delivery channel can queue/retry) | Less strict (delivery channel can queue/retry) |
| Operational complexity | Low | Medium | Highest (qualified trust service onboarding/ops) |
| If implementing acts push registered delivery | Might still be needed for presentation semantics, but transport may need ERDS/QERDS | Possible | Likely if "qualified" is required |

The following diagram illustrates the (Q)ERDS direct send/receive flow:

```mermaid
sequenceDiagram
    participant Supplier_Wallet as Supplier_Wallet
    participant Supplier_QERDS as Supplier_QERDS
    participant Buyer_QERDS as Buyer_QERDS
    participant Buyer_Wallet as Buyer_Wallet

    rect rgb(220, 230, 245)
        Note over Supplier_Wallet, Buyer_Wallet: Submit for registered delivery
        Supplier_Wallet ->> Supplier_QERDS: Submit eDelivery package (eInvoice Attestation + invoice payload + metadata)
        Supplier_QERDS ->> Supplier_QERDS: Identify/authenticate Supplier, bind to legal entity (EUCC/LPID)
        Supplier_QERDS ->> Supplier_QERDS: Apply integrity protection (seal/sign + timestamp + package hash)
    end

    rect rgb(220, 245, 220)
        Note over Supplier_Wallet, Buyer_Wallet: Registered transmission
        Supplier_QERDS ->> Buyer_QERDS: Transmit registered delivery (QERDS CH) — package + routing + correlation id
        Buyer_QERDS ->> Buyer_QERDS: Identify/authenticate Recipient (Buyer), bind to Buyer legal entity (EUCC/LPID)
        Buyer_QERDS ->> Buyer_QERDS: Validate integrity + timestamps (check seal/signature, package hash)
    end

    rect rgb(245, 235, 220)
        Note over Supplier_Wallet, Buyer_Wallet: Delivery to recipient
        Buyer_QERDS ->> Buyer_Wallet: Deliver eDelivery package (eInvoice Attestation + payload + metadata)
    end

    rect rgb(240, 230, 255)
        Note over Supplier_Wallet, Buyer_Wallet: Evidence generation
        Supplier_QERDS -->> Supplier_Wallet: Proof of Sending — sealed/signed + timestamp + correlation id
        Buyer_QERDS -->> Buyer_Wallet: Proof of Receipt — sealed/signed + timestamp + correlation id
        Buyer_QERDS -->> Supplier_QERDS: Proof of Receipt — sealed/signed + timestamp + correlation id
        Supplier_QERDS -->> Supplier_Wallet: Forward Proof of Receipt/Delivery — sealed/signed + timestamp + correlation id
    end
```

---

## Annex 1 — Requirements for scenario roles

> ⚠️ *To be completed during specification phase based on partner input.*

| Primary role | Specific requirement |
|-------------|---------------------|
| User of Business Wallet | |
| Authentic sources | |
| Relying party | |
| Intermediary | |
| Business Wallet provider | |
| PID Provider | |
| Trusted list registrar | |
| QEAA provider | |
| Pub-EAA provider | |
| EAA provider | |
| QES provider | |

---

## Annex 2 — Open issues and decisions log

| # | Issue | Status | Responsible |
|---|-------|--------|-------------|
| 1 | Decide invoice payload canonicalization method(s) and hash strategy. | open | WP4 Architecture |
| 2 | Decide invoice semantic (e.g. EN16931 semantic model, Peppol BIS 3.0 UBL syntax, or both with one as primary) or align to ViDA. | open | WP4 Semantic |
| 3 | Decide trust anchor mechanism for LPID and signing keys (LoTE/DDE registry vs federation vs QTSP signals). | open | WP4 Trust Registry |
| 4 | Decide minimum disclosed invoice elements for validation vs privacy. | open | |
