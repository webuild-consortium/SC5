# QERDS Binding (eIDAS Art. 44) — Sketch

**A sketch of the QERDS-transport binding for the Portable Trust Envelope (PTE).**

| | |
|---|---|
| **Date** | 2026-06-01 |
| **Version** | 0.1 Sketch |
| **Status** | Demonstrates transport-independence; not pilot-ready |
| **Author** | Hans Boone — Banqup |
| **Layer** | TrustLayer Layer 3 — transport binding (see [../../README.md](../../README.md)) |

> **This is a sketch, not a specification.** It exists to demonstrate that the Portable Trust Envelope is genuinely transport-independent. A production-ready QERDS binding would require alignment with ETSI EN 319 522 and EUBW Implementing Acts.

---

## 1. Why a QERDS binding matters

QERDS — Qualified Electronic Registered Delivery Service, defined by eIDAS Art. 44 — is the right transport for document exchanges where:

- **Legal-evidentiary weight is required.** QERDS provides a qualified, eIDAS-recognised proof of sending, receipt, and integrity, including non-repudiation by both parties.
- **Formal notices and contracts** are exchanged (court communications, formal regulator submissions, contractual deliveries, statutory notifications).
- **Time-stamped delivery is required** (qualified timestamps as part of the QERDS evidence package).

Many cross-border B2B and B2G use cases in the EU sit in this space: cross-border legal proceedings (e-CODEX), public procurement notifications, regulator submissions, formal contractual notices.

**Today**, QERDS provides excellent transport-level evidence (who sent what to whom, when, integrity-protected), but it has **no end-to-end content-level trust layer** for the document being delivered. The QERDS evidence proves "this exact byte string was delivered"; it does not prove "the IBAN inside is verified", "the supplier is approved", or "the sender is authorised to seal on behalf of the named legal entity".

The PTE adds exactly that content-level trust layer on top of QERDS. The combination is powerful: QERDS provides regulator-grade delivery evidence; the PTE provides regulator-grade content trust. Both anchored on qualified trust services.

## 2. Relationship between QERDS evidence and the PTE

The two are **complementary**, not redundant.

| Layer | Provided by | What it proves |
|---|---|---|
| QERDS evidence | The QERDS provider (a QTSP under eIDAS) | The payload was delivered from sender to receiver at a specific qualified time, with integrity, non-repudiation of sending, and non-repudiation of receipt. |
| PTE | The legal-entity sender's EUBW (via APEIP) | The document content is sealed by the legal entity, the sealing intermediary is authorised, the IBAN/VAT/approved-supplier credentials are verified live. |

A document delivered via QERDS without a PTE has delivery-level trust but no content-level trust. A PTE-carrying document delivered via QERDS has both. Both layers can be verified independently and both contribute to a complete chain of evidence.

## 3. The binding choices

### 3.1 Carrier — PTE inside the QERDS payload

A QERDS message has two main components:

- **Payload** — the actual content being delivered (one or more files).
- **Evidence** — the QERDS provider's signed evidence package (sending evidence, integrity proof, receipt evidence).

The PTE rides **inside the payload**. The payload becomes a `multipart/related` structure (parallel to the mail binding's structure) containing:

```
payload/
├── document.xml            ← UBL invoice, CII invoice, or other document
├── trust-manifest.xml      ← STM
├── seal.xades              ← XAdES detached seal over document.xml + trust-manifest.xml
└── trust-validation.xml    ← TVE (written by receiver-side handler after APEIP /verify)
```

The QERDS evidence wraps the entire payload. The evidence's content-integrity hash covers all four parts above (or the entire multipart structure, depending on the QERDS provider's evidence format).

### 3.2 Two integrity layers — QERDS and the qualified seal

A common confusion: doesn't QERDS already sign the payload?

Yes — QERDS evidence proves the payload was delivered with integrity. But QERDS evidence is signed by the **QERDS provider**, not by the legal-entity sender of the document. The QERDS provider attests to delivery; it does not attest to content authorship or content authority.

The PTE's qualified seal is signed by the **legal-entity sender** (the company, via its EUBW or QTSP). This is the seal that gives the document its eIDAS Art. 35 presumption of integrity and origin from the legal entity itself.

Both layers are necessary:

| Without QERDS evidence | Without PTE seal |
|---|---|
| No qualified proof of delivery time, no qualified non-repudiation of receipt | No proof the legal entity authored or authorised the content |
| Disputes about whether the document was actually delivered, when, and to whom | Disputes about whether the document content is what the sender claims it is |

The two layers stack cleanly because they protect different facets of the exchange.

### 3.3 Handler — the QERDS provider plays both roles

In the Peppol binding, the AP (C2/C3) is the handler. In the mail binding, the mail gateway is the handler. In the QERDS binding, the **QERDS provider** is naturally positioned to be the handler:

- The QERDS provider is, by definition, a QTSP — it already operates qualified cryptographic infrastructure, holds eIDAS Trusted List status, and has the regulatory standing to integrate with EUBWs.
- The QERDS provider has access to the payload (it must, in order to produce the evidence) and the appropriate operational privileges to add/inspect MIME parts.
- The QERDS provider operates on both sides of the exchange (sender's QERDS provider produces the payload + evidence; receiver's QERDS provider verifies and forwards).

So:

| Side | Handler role | APEIP role |
|---|---|---|
| **Sender** | Sender's QERDS provider | Calls APEIP `/credentials/for-document`, `/seal`, `/presentation-request` |
| **Receiver** | Receiver's QERDS provider | Calls APEIP `/verify`, writes TVE part |

The QERDS provider's existing QTSP infrastructure can host or front the EUBW operations — there is significant operational overlap between "QTSP that runs an QERDS service" and "EUBW provider" in the EUBW Regulation's structure. A single legal entity may be both.

### 3.4 Seal format

The PTE seal inside the QERDS payload is XAdES (for XML documents), PAdES (for PDF documents), or CAdES (for binary documents) — same as the mail binding. The choice is driven by document type, not by QERDS.

### 3.5 Discovery — QERDS provider capability advertisement

QERDS providers advertise their capabilities via the standard ETSI mechanisms for QTSP capability discovery (e.g., LoTL / Trusted List service-type-identifier extensions). PTE support could be advertised:

- As a service-type-identifier extension on the QTSP's Trusted List entry.
- Via the QERDS provider's API capability advertisement (analogous to OpenAPI capability metadata).

**OPEN: standardised discovery format.** ETSI EN 319 522 (ERDS) and the QERDS Implementing Acts under eIDAS would be the natural homes for a capability declaration.

## 4. Flow

### 4.1 Send side

1. Sender's application submits a document to its QERDS provider.
2. QERDS provider extracts the document, looks up the receiver's PTE-over-QERDS capability.
3. If receiver supports PTE: QERDS provider calls APEIP `/credentials/for-document`, builds STM, calls APEIP `/seal`, assembles the multipart payload.
4. QERDS provider applies the qualified seal of the QERDS evidence over the payload.
5. QERDS provider routes to the receiver's QERDS provider.

### 4.2 Receive side

1. Receiver's QERDS provider receives the message.
2. QERDS provider verifies the QERDS evidence (integrity, sending qualification).
3. If a PTE STM part is present: QERDS provider calls APEIP `/verify`.
4. QERDS provider constructs the TVE part from the response and integrates it into the payload (the QERDS evidence is **not** re-signed — the TVE addition is post-delivery and lives outside the QERDS evidence scope, just as it lives outside the PTE seal scope).
5. QERDS provider delivers to the receiver's application + produces receipt evidence.

### 4.3 The two-evidence interaction

The receiver ends up with three signed artifacts:

1. **QERDS evidence (sending + receipt)** — signed by the QERDS provider(s). Proves delivery.
2. **PTE qualified seal** — signed by the sender's legal entity (via EUBW). Proves content authorship.
3. **EUBWAssertion in the TVE** — signed by the receiver's EUBW. Proves the verification result.

Together, these three provide a complete cryptographic chain of evidence: who sent it, what they sent, that the content was authentic, that delivery occurred at a qualified time, and that an independent verifier validated the content trust. This is significantly stronger than QERDS alone or PTE alone.

## 5. Open questions

| # | Question |
|---|---|
| O1 | **Alignment with ETSI EN 319 522.** ERDS specifies its own payload structures and evidence formats. The PTE inside the QERDS payload must align with ETSI's mandated payload constraints without conflict. |
| O2 | **Trusted List interaction.** A QERDS provider already has a Trusted List entry; adding "this QERDS provider also handles PTE" as a TL extension requires coordination with the EU LoTL governance. |
| O3 | **APEIP authentication for QERDS providers.** A QERDS provider is a QTSP. Should its APEIP authentication automatically inherit from its TL status, or does APEIP require a separate registration on the EUBW Trusted List? |
| O4 | **Cross-border QERDS-to-QERDS interoperability.** ETSI EN 319 521/522 specifies QERDS-to-QERDS interoperability protocols; PTE must not break this. |
| O5 | **Evidence storage.** Both QERDS evidence and EUBWAssertion need long-term storage for regulator/court access. Should they be stored separately or in a combined evidence bundle? |

## 6. Status

This document is a **sketch** to demonstrate feasibility. The architecture (PTE inside the QERDS payload, QERDS provider as APEIP handler, two-layer integrity model) is internally consistent and aligns with how QERDS already works under eIDAS.

Completing the binding to a pilot-ready state requires:

- ETSI EN 319 522 alignment review.
- Capability advertisement format agreed (Trusted List extension or API).
- A QERDS provider that is also an EUBW provider (or has APEIP integration with one) — both Banqup and several other QTSPs are positioned for this.
- A pilot scenario from a domain where QERDS is already deployed: cross-border legal communications (e-CODEX), regulator submissions, or formal procurement notifications.

A logical next step would be a tabletop exercise with one or two QTSPs currently operating QERDS services to validate the architecture before committing to a pilot.
