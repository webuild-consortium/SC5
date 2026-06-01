# Peppol Binding — Oxalis-NG Reference Handler

**The Peppol transport binding for the Portable Trust Envelope (PTE), implemented as an extension to Oxalis-NG.**

| | |
|---|---|
| **Date** | 2026-06-01 |
| **Version** | 0.2 Draft |
| **Status** | For discussion |
| **Author** | Hans Boone — Banqup |
| **Layer** | TrustLayer Layer 3 — transport binding (see [../../README.md](../../README.md)) |
| **Companion documents** | [../../architecture/PTE-Architecture.md](../../architecture/PTE-Architecture.md), [../../protocol/APEIP-Specification.md](../../protocol/APEIP-Specification.md), [../README.md](../README.md) |

---

## 1. Scope and positioning

This document specifies the **Peppol transport binding** for the Portable Trust Envelope (PTE) and proposes a reference implementation built on Oxalis-NG. It is one of several transport bindings; see the [binding matrix](../README.md) for the others.

**What this binding specifies:**

- The UBL carrier element used to hold the STM, the qualified seal, and the TVE in a Peppol BIS Billing 3.0 invoice.
- The role of the Peppol Access Point (C2, C3) as the message handler that integrates with its companion EUBW provider via APEIP.
- The Oxalis-NG extension points used to implement that handler role.
- Discovery via the Peppol SMP.

**What this binding does not specify:**

- The PTE data model itself — see [PTE-Architecture.md](../../architecture/PTE-Architecture.md).
- The APEIP protocol between the AP and the EUBW — see [APEIP-Specification.md](../../protocol/APEIP-Specification.md).
- The AS4 transport, the Peppol PKI, the SMP/SML mechanics — these are unchanged from Peppol today.

### 1.1 Scope clarification: EUBW-targeted, not wallet-agnostic

An earlier draft of this proposal framed the integration as "wallet-agnostic" with the intent to support EUDI Wallet, sector-specific wallets, and future organisational wallet platforms in addition to the European Business Wallet. That framing has been narrowed.

**The integration target is the European Business Wallet (EUBW).**

The reasons for narrowing:

| Concern | Why this scoping decision |
|---|---|
| **EUDI Wallet is a natural-person wallet.** | Cross-organization invoice flows are legal-person events. A natural person does not book a company's invoices; integrating Peppol APs directly with EUDI is a category error for B2B e-invoicing. |
| **"Sector-specific wallets" are speculative.** | No concrete standards or deployed examples. Designing for them would mean designing for nothing in particular and producing an integration so generic that no actual integration ships. |
| **The APEIP boundary is already wallet-neutral.** | The AP integrates with one companion EUBW provider via APEIP. Whether that EUBW is operated by Sphereon, Banqup, Credenco, or any other provider on the EUBW Trusted List makes no difference to the AP. The wallet *implementation* is abstracted by APEIP; further abstraction adds no value. |
| **Wider integration is achievable via the EUBW Trusted List.** | EUBWs interoperate with each other via the EUBW Trusted List of Business Wallet Providers. The Peppol AP does not need integration code per wallet type; it integrates with one EUBW, and that EUBW reaches all the others. |

The integration *pattern* (Oxalis-NG handlers, event bus, audit storage) remains general and could in principle be repurposed for non-EUBW wallet types in the future, but the **specification** in this document is EUBW-targeted.

## 2. The Peppol binding choices

### 2.1 Carrier — `<ext:UBLExtensions>`

The PTE is carried in sibling `<ext:UBLExtension>` elements of the Peppol BIS Billing 3.0 UBL document. Three elements:

```xml
<Invoice xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"
         xmlns:ext="urn:oasis:names:specification:ubl:schema:xsd:CommonExtensionComponents-2">
  <ext:UBLExtensions>
    <ext:UBLExtension>
      <ext:ExtensionURI>urn:peppol:trust:pte:1.0:stm</ext:ExtensionURI>
      <ext:ExtensionContent>
        <!-- STM: written by Peppol C2 before sealing -->
      </ext:ExtensionContent>
    </ext:UBLExtension>
    <ext:UBLExtension>
      <ext:ExtensionURI>urn:oasis:names:specification:ubl:dsig:enveloped:xades</ext:ExtensionURI>
      <ext:ExtensionContent>
        <ds:Signature Id="PortableTrustSeal">
          <!-- XAdES qualified seal: covers UBL body + STM, excludes TVE -->
        </ds:Signature>
      </ext:ExtensionContent>
    </ext:UBLExtension>
    <ext:UBLExtension>
      <ext:ExtensionURI>urn:peppol:trust:pte:1.0:tve</ext:ExtensionURI>
      <ext:ExtensionContent>
        <!-- TVE: written by Peppol C3 after APEIP /verify, post-seal -->
      </ext:ExtensionContent>
    </ext:UBLExtension>
  </ext:UBLExtensions>
  <!-- Peppol BIS Billing 3.0 body, unchanged -->
</Invoice>
```

This realises the abstract STM/seal/TVE model from [PTE-Architecture §3.5](../../architecture/PTE-Architecture.md#35-container-split--resolving-the-seal-scope-problem) in the Peppol binding's specific carrier.

### 2.2 Seal — XAdES via UBL DSig profile

The qualified seal is XAdES (ETSI EN 319 132) using the OASIS UBL Digital Signature profile (`urn:oasis:names:specification:ubl:dsig:enveloped:xades`). The seal covers the UBL document body + STM and excludes itself (via the standard `enveloped-signature` transform) and the TVE (via a prospective XPath transform — the TVE does not exist at seal time but the transform anticipates it).

Detailed generation procedure in [PTE-Architecture §4.6](../../architecture/PTE-Architecture.md#46-the-qualified-seal--design-decisions).

### 2.3 Handler — the Peppol Access Point

The Peppol AP is the message handler. It plays both APEIP roles:

| AP role | APEIP endpoints called | Trigger |
|---|---|---|
| **C2 (sender AP)** | `/credentials/for-document`, `/seal`, optionally `/presentation-request` | C1's ERP submits an invoice via Oxalis SUBMIT. |
| **C3 (receiver AP)** | `/verify` | AS4 RECEIVE completes; document is unwrapped from SBDH; STM is detected in `<ext:UBLExtensions>`. |

The AS4 transport itself is unchanged. The AP becomes wallet-aware in two specific places (before SUBMIT, after RECEIVE); everywhere else it operates as a normal Peppol AP.

### 2.4 Discovery — SMP capability identifier

C4's SMP record advertises PTE support via a new capability identifier:

```
peppol-trust-envelope-v1
```

with optional sub-properties for receiver policy hints:

```
peppol-trust-envelope-v1;requires=approved-supplier:hard
peppol-trust-envelope-v1;requires=approved-supplier:soft
peppol-trust-envelope-v1;policy=urn:peppol:trust:scoring:v1
```

C2 reads these during the SMP lookup it already performs for AS4 routing. If `peppol-trust-envelope-v1` is absent, C2 falls back to sending a standard Peppol BIS UBL with no PTE — the invoice still flows through the regular Peppol rails. The PTE is graceful-degradable.

## 3. Oxalis-NG as the reference implementation platform

[Oxalis-NG](https://github.com/OxalisCommunity/oxalis) is selected as the reference implementation platform because:

| Criterion | Oxalis-NG fit |
|---|---|
| Open Source | Yes (Apache 2.0). |
| Peppol native | Yes (the canonical Java AP). |
| AS4 support | Yes (Holodeck B2B-based). |
| Extensible via PersisterHandler / PayloadPersister / ReceiptPersister | Yes — the same extension points are used by Belgium's CTC adapters and other national profiles. |
| Modular persistence | Yes. |
| Active community | Yes (OxalisCommunity GitHub org). |

The reference implementation is one possible implementation; other AP platforms (proprietary, .NET, Go, etc.) can implement the same binding by integrating APEIP at equivalent extension points.

## 4. Extension points on Oxalis-NG

### 4.1 Inbound (receiver side, C3)

Two existing Oxalis-NG extension points are used:

#### 4.1.1 `PayloadPersister`

Responsibilities for the PTE binding:

1. After AS4 decryption and Peppol BIS schematron validation, examine `<ext:UBLExtensions>` for an entry with URI `urn:peppol:trust:pte:1.0:stm`.
2. If present, canonicalise the document body and the STM extension content separately.
3. Call APEIP `/verify` (see [APEIP §8](../../protocol/APEIP-Specification.md#8-inbound-verify)) with: the canonicalised STM, the canonicalised document body, the sealed-document digest, and the receiver policy URI.
4. Construct a TVE extension from the response and insert it as a sibling of the STM in `<ext:UBLExtensions>`.
5. Persist the augmented UBL document.
6. Emit an internal event (see §5) for downstream observability — the event is *not* the trust artifact, only an operational signal.

If no STM is present, the persister behaves as a standard Oxalis PayloadPersister with no PTE involvement.

#### 4.1.2 `ReceiptPersister`

Responsibilities for the PTE binding:

1. Persist the AS4 receipt (existing behaviour).
2. Additionally persist the JWS-signed EUBWAssertion returned by APEIP `/verify`. This is the authoritative non-repudiation artifact for the trust verification — separate from, and complementary to, the AS4 receipt for transport non-repudiation.

### 4.2 Outbound (sender side, C2)

A sender wrapper module composes APEIP outbound calls with the existing Oxalis sender:

1. C1's ERP submits a UBL invoice via Oxalis SUBMIT.
2. The wrapper extracts the minimal fields needed by APEIP (sender EUID, receiver participant ID, IBANs, VAT numbers, total).
3. The wrapper calls APEIP `/credentials/for-document` to obtain the credential set.
4. The wrapper inspects the SMP record (already retrieved by Oxalis for AS4 routing) for `peppol-trust-envelope-v1;requires=approved-supplier:*`. If `hard` and not in the credential set, it calls APEIP `/presentation-request` to retrieve the Approved Supplier credential. If `soft` it does the same but tolerates `not_available`.
5. The wrapper assembles the STM extension and inserts it into `<ext:UBLExtensions>`.
6. The wrapper calls APEIP `/seal` with the canonicalised UBL + STM and assembles the resulting `<ds:Signature>` extension.
7. The wrapper hands the now-sealed UBL document back to Oxalis for AS4 SUBMIT (standard path, unchanged).

The wrapper is implemented as either a pre-processing filter or as a custom `Sender` decorator depending on the Oxalis-NG version.

### 4.3 What is NOT implemented

The earlier draft of this document proposed a per-invoice authorization check before send ("Validate authorization. Resolve wallet credentials. Generate audit event."). The per-invoice authorization step is **not** part of this design.

The reason: authorization is *standing*, not per-invoice. C1 issues an `IntermediationMandate` to its AP at onboarding time, scoped to `e-invoicing:seal+submit`. That mandate persists across all invoices C1 sends. Each invoice does not re-prove the authorization; the seal applied via APEIP `/seal` is itself the proof that the AP is authorised, because the EUBW would refuse the `/seal` call if the mandate were invalid (APEIP error `urn:apeip:error:mandate-invalid`).

Adding per-invoice authorization on top of standing authorization is theatre, not security.

## 5. Internal event model (operational, not trust)

Oxalis-NG can publish internal events for downstream observability — receipts, audit, ERP back-channel notifications. These events are **operational**, not trust artifacts. The trust artifacts are in the PTE inside the UBL document.

| Event | When emitted | Carries |
|---|---|---|
| `pte.outbound.stm-built` | After STM construction at C2, before seal | sealed-document digest, credential references |
| `pte.outbound.sealed` | After APEIP `/seal` returns | seal value, sealed-at timestamp |
| `pte.outbound.sent` | After AS4 SUBMIT completes | AS4 message ID, AS4 receipt reference |
| `pte.inbound.received` | After AS4 RECEIVE completes | AS4 message ID, document type, sender, receiver |
| `pte.inbound.verified` | After APEIP `/verify` returns | trust score, dimensional scores, EUBWAssertion ID |
| `pte.inbound.delivered` | After C4 delivery | delivery channel reference |

The events are useful for AP operations (SLA monitoring, fraud detection, regulator reporting) but they are not the trust signal C4 receives — C4 receives the TVE inside the UBL document.

Recommended bus: Kafka, RabbitMQ, or any message queue the AP operator already uses. The binding does not mandate a specific implementation.

## 6. Audit storage

The receiver-side AP must persist the JWS-signed EUBWAssertion for the regulator-relevant retention period (typically 7–10 years for tax compliance). Storage location is operator-defined:

- Object storage (Azure Blob, S3, on-prem).
- Database (relational or NoSQL).
- Existing AS4 evidence archive (extended to hold the EUBWAssertion alongside the AS4 receipts).

The EUBWAssertion is the artifact a regulator or auditor would request to verify a past transaction's trust. The TVE inside the UBL is the same artifact in its in-document form.

## 7. Reference implementation modules

The reference implementation is proposed as four open-source Java modules, hosted in a Banqup-led repository under the OxalisCommunity GitHub organisation (subject to OxalisCommunity acceptance):

| Module | Responsibility |
|---|---|
| `oxalis-pte-apeip-client` | APEIP client library (HTTP client, mTLS, request/response shapes). Reusable across the inbound and outbound modules. |
| `oxalis-pte-inbound` | `PayloadPersister` and `ReceiptPersister` implementations for the PTE inbound flow. Detects STM, calls APEIP `/verify`, writes TVE. |
| `oxalis-pte-outbound` | Sender wrapper for the PTE outbound flow. Builds STM, calls APEIP `/seal`. |
| `oxalis-pte-sample-deployment` | Docker Compose for a complete local environment: Oxalis-NG + sample APEIP-compliant EUBW + Kafka + object storage. |

The modules are designed so that an AP operator can add PTE support to an existing Oxalis-NG deployment by adding two dependencies and pointing them at the operator's APEIP-compliant EUBW endpoint.

## 8. Pilot deployment — SC5 Pilot 1

The first concrete deployment of this binding is SC5 Pilot 1 (Banqup ↔ Semansys/Sphereon, scenarios 1 + 2). See [../../sc5-binding/SC5-PTE-Technical-Binding.md §11](../../sc5-binding/SC5-PTE-Technical-Binding.md#11-pilot-proposal) for the staged plan.

Banqup acts as both Peppol C2 and EUBW C2 (same-provider APEIP authentication mode). Semansys acts as Peppol C3; Sphereon acts as EUBW C3. The bilateral APEIP mode is used between Semansys and Sphereon.

## 9. Standardisation path

| Body | Artifact |
|---|---|
| **OpenPeppol** | Peppol BIS Trust Profile: schemas for STM and TVE namespaces, schematron rules, SMP capability identifier definition. AP processing rules as an OpenPeppol Recommendation. |
| **OxalisCommunity** | Acceptance of the four reference implementation modules into the OxalisCommunity GitHub organisation. |

Bilateral pilots can run before either of these — the `<ext:UBLExtensions>` mechanism is open-by-design and the SMP can advertise unrecognised capability identifiers without rejection. OpenPeppol Recommendation status is the path to broad AP adoption, not a precondition for piloting.

## 10. Differences from the earlier "Peppol-to-EUBW Reference Handler" draft

For reviewers familiar with the earlier draft of this proposal, the substantive changes:

| Earlier draft | This document |
|---|---|
| "Wallet-agnostic" framing covering EUDI Wallet, business wallets, sector wallets | EUBW-targeted via APEIP. See §1.1. |
| Per-invoice authorization check before send | Removed. Authorization is standing via the `IntermediationMandate`. See §4.3. |
| `wallet-connector` module abstracting multiple wallet integrations | One `oxalis-pte-apeip-client` module integrating with the EUBW via APEIP. The wallet-implementation choice is the EUBW operator's, not the AP's. |
| Event model with `DocumentReceived`, `DocumentSent`, etc. as the primary integration surface | Event model demoted to operational observability; the PTE inside the UBL is the trust artifact. See §5. |
| Positioned as a generic Peppol-to-wallet bridge | Positioned as the **Peppol transport binding** for the Portable Trust Envelope, one of several bindings. See §1. |

These changes align the proposal with the three-layer TrustLayer architecture (envelope / protocol / bindings) and remove duplications and category errors that the earlier draft contained.
