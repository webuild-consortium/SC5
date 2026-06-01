# Portable Trust Envelope (PTE) — Architecture

**Scope:** Authentication, Authorization, Sealing, and document data validation, delivered as a transport-independent trust envelope anchored on the European Business Wallet (EUBW) ecosystem. The Peppol BIS UBL invoice case is used as the worked example throughout; other transport bindings are covered separately in [../bindings/](../bindings/).

**Status of this document:** Design proposal. The EUBW Regulation is a draft (November 2025). No EUBW–Peppol integration specification exists publicly as of May 2026; OpenPeppol has published nothing on EUBW integration. Everything below the "verified anchors" table is architecture extrapolation, not a description of a shipped standard.

**Naming note.** PTE stands for **Portable Trust Envelope**. The Peppol binding is the most worked-through example because it is the binding nearest to production deployment (Belgium B2B mandate, SC5 pilots); examples in this document use the Peppol binding's UBL realisation but the underlying envelope concept is transport-agnostic. See §3.7 for the binding model and [../bindings/README.md](../bindings/README.md) for the binding matrix.

---

## 1. Verified anchors

Each row confirmed by at least two independent sources.

| Element | Status | Basis |
|---|---|---|
| EUBW = "EUDI Wallet for Organizations," extends eIDAS 2.0 to legal entities | Confirmed, draft regulation Nov 2025 | Spherity / Partisia / iGrant executive summaries; reuses EUDI ARF + toolbox |
| EUBW explicitly targets invoicing + company representation | Confirmed | EUBW draft: "act/sign on behalf of a company for tax compliance, cross-border transactions, invoicing" |
| EUBW uses EUID + a Commission-operated directory + a Trusted List of Business Wallet Providers | Confirmed | EUBW draft regulation, Nov 2025 |
| EUBW interoperability with ViDA is a stated objective | Confirmed | EUBW draft objectives (OOTS, BRIS, BORIS, ViDA, DPP, IEA) |
| Peppol uses 4-corner, AS4, SMP/SML, Peppol PKI for signing/encryption | Confirmed | Peppol AS4 Profile; multiple network descriptions |
| Belgium B2B mandate live 1 Jan 2026 (Peppol BIS/UBL); continuous transaction reporting via 5-corner from 2028 | Confirmed | Belgian Royal Decree 14 Jul 2025; ViDA mandate trackers |
| EN 16931-1:2025 formally approved by CEN 13 Feb 2026 | Confirmed | CEN; ViDA mandate trackers |
| Peppol BIS Billing 3.0 is a CIUS of EN 16931 bound exclusively to UBL 2.1; EN 16931 also has a CII syntax binding used by ZUGFeRD/Factur-X/XRechnung-CII | Confirmed | OpenPeppol POACC documentation; TR 16931-5; multiple format mappers |

**The core gap this extension closes:** Peppol and the EUBW trust ecosystem run on two parallel, non-interoperable trust roots. Peppol identity is asserted by the *Access Point's* certificate from the Peppol PKI (hop-by-hop, C2↔C3, machine-to-machine). The *sender's* legal identity (corner 1) is asserted only by a participant identifier in the header plus the AP's out-of-band onboarding KYC. There is no cryptographic, portable link from the legal entity that issued the invoice — nor from the service provider authorized to seal it — to the bytes on the wire. A EUBW extension closes that gap.

---

## 2. The four functions, mapped to current gaps

### 2.1 Authentication — who is corner 1, cryptographically?

**Today:** The AS4 message is signed by corner 2's (the AP's) Peppol-PKI certificate. The sending business is identified only by a participant identifier (e.g. `9925:BE0123456789`, scheme 9925 = Belgian VAT). Nothing cryptographically binds the legal entity to that identifier; trust is inherited from the AP's onboarding KYC, which is out-of-band, non-portable, and unverifiable by corner 4.

**With the EUBW extension:** The EUBW holds a Legal Person Identification (LPID) attestation and an EUID. The extension carries a wallet-derived, verifiable **Legal Person Attestation** bound to the participant identifier. Corner 3/4 verify it against the EUBW Trusted List rather than trusting only the Peppol PKI. This converts Peppol from "trust the pipe operator" to "trust the originating legal entity."

### 2.2 Authorization — is this intermediate allowed to issue/sign on behalf of this company?

This reframes the question correctly for the real-world flow. In practice corner 1 is not a natural person logging in to sign each invoice. Corner 1 is a **legal entity** whose ERP system produces invoices in batch and forwards them to a **Peppol service provider (the intermediate / sender Access Point)** for delivery. The relevant authorization question is therefore:

> Is this Peppol service provider authorized to issue and apply the qualified seal on behalf of corner 1?

**Mechanism — an intermediation / sealing mandate.** The company's EUBW issues a scoped mandate attestation to the service provider's EUBW (legal-person to legal-person): *"Service provider X is authorized to apply the company's qualified e-seal and submit invoices on behalf of legal entity Y (EUID …), scope = e-invoicing, valid until …"*. The extension carries a reference to this mandate so corner 3 can verify, against the EUBW Trusted List, that the sealing intermediary was authorized by the named legal entity.

Peppol today has no representation-authority layer at all; this is genuinely new capability and is entirely a legal-person (EUBW) construct — no natural-person identity is involved.

### 2.3 Sealing — qualified electronic seal on the invoice, not just transport signing

Two distinct signatures must not be conflated:

- **Transport-level signing (exists today):** AS4 message signed by the AP's Peppol-PKI cert. Proves the hop, not the legal origin of the content.
- **Content-level qualified electronic seal (QESeal) over the invoice (largely absent in practice today):** a legal-entity seal under eIDAS, giving the EN 16931 invoice a presumption of integrity and origin.

The extension lets the Peppol service provider apply (or invoke a QTSP to remotely apply) a **qualified e-seal over the canonicalised UBL document**, with the sealing certificate chaining to the EUBW/QTSP trust root. The seal is end-to-end (C1→C4), not hop-by-hop. This is where the extension consumes QTSP seal services and the seal certificate becomes verifiable inside the Peppol envelope.

### 2.4 eInvoice data validation — semantic + trust validation at C3

**Today:** validation is two disconnected things — schematron / EN 16931 *format* validation at C2 and C3, and *no* trust validation of the asserting entity beyond the AP cert.

**With the proposed EUBW extension, at C3 the extension validates the invoice content against authoritative sources:**

- **IBAN** — verified against the payment account associated with the sealing legal entity / supplier credential.
- **VAT number** — verified against the legal entity's EUID and VAT registration.
- **Business registry data** — verified against the EUBW directory / authoritative registry (consistency of legal name, EUID, status).
- **Approved-supplier credential (optional, receiver-driven):** some receivers require the supplier to hold a buyer-issued approved-supplier credential before booking. Delivered in one of two ways (see §3.3.1):
  1. **Pushed in the STM** by Peppol C2 at send time, when it knows the receiver requires it.
  2. **Pulled by EUBW C3** via a presentation request to EUBW C2 at verify time, when the credential was not in the STM but the receiver's policy requires it.
  The receiver's policy decides whether absence/failure is a soft signal (score reduction) or a hard decline (§4.8).

The output is a combined "format-valid AND origin-trusted AND data-verified" validation report — exactly what a continuous-transaction-control (CTC) regime (e.g. Belgium's 5-corner from 2028) wants: corner 5 (the tax authority) gets cryptographically-attested, registry-consistent origin for free.

### 2.5 Relationship to EN 16931, Peppol BIS, CIUS, and CII

This design has so far used "Peppol BIS UBL" and "EN 16931 UBL" loosely. The precise relationship matters for standardization scope and for interoperability with non-Peppol e-invoicing rails. Three layered standards are at play:

| Standard | Role | What it defines |
|---|---|---|
| **EN 16931** (CEN) | Semantic data model | What business information an e-invoice must carry (business terms: BT-1 invoice number, BT-31 seller VAT, BT-84 IBAN, etc.). Independent of XML syntax. Mandated by EU Directive 2014/55/EU for B2G acceptance. |
| **EN 16931 syntax bindings** | Two official XML encodings | (a) **UBL 2.1** (OASIS) — verbose, self-documenting; (b) **UN/CEFACT CII** (Cross Industry Invoice) — compact, deeply nested. Both encode the same EN 16931 semantic model. |
| **Peppol BIS Billing 3.0** | CIUS + network profile | A **Core Invoice Usage Specification** of EN 16931 that tightens it (makes optional fields mandatory, restricts code lists, adds participant identifier schemes) and binds it **exclusively to UBL 2.1**. Adds Peppol-specific network requirements (SMP, AS4 transport). |

**Where this design currently sits:** the STM is defined as a Peppol BIS extension carried in `<ext:UBLExtensions>` of a Peppol BIS Billing 3.0 UBL document. That is the smallest viable scope and the cleanest first deployment target — Peppol BIS is what Belgian B2B (2026), Norwegian EHF, and most Northern/Western European e-invoicing actually uses on the wire.

**What this scope does NOT cover:**

1. **CII syntax.** EN 16931 has a second official syntax binding to UN/CEFACT CII, dominant in Franco-German trade through **ZUGFeRD** (Germany), **Factur-X** (France), and as the second permitted syntax for **XRechnung** (Germany). CII has its own extension mechanism (`ram:SpecifiedTradeAllowanceCharge`-style structures) but it is **not** UBL `<ext:UBLExtensions>`. The PTE as designed does not embed in CII. See §6 item 5 for what a CII binding would require.
2. **National CIUS profiles outside Peppol.** XRechnung (Germany), CIUS-PT (Portugal), Singapore BIS (which itself extends Peppol BIS), future French PPF profiles. These are EN 16931-conformant but each adds its own constraints; the PTE may need profile-specific schematron rules to coexist without false validation failures.
3. **Non-EN-16931 formats.** Italy's **FatturaPA** is not EN 16931-conformant and routes via the SDI platform rather than Peppol. The PTE design is irrelevant to FatturaPA in its current form.
4. **Non-invoice document types.** Peppol BIS covers credit notes, orders, despatch advices, and more. The PTE design is invoice-focused — credit notes share the same trust requirements with trivial schema reuse, orders need a different supporting-credential set (no IBAN, possibly delivery-address credentials).

**Positioning under TR 16931-5 — CIUS or Extension?**

CEN/TC 434's **TR 16931-5** ("Guidelines on the use of sector or country extensions") distinguishes two ways to deviate from the EN:

- **CIUS:** tightens the EN (mandatory subset of optional fields, restricted code lists). Cannot add new content. Conformant by construction.
- **Extension:** adds new business terms beyond the EN. May be ignored by recipients without bilateral agreement. Conformant only when the core EN-16931 fields are still independently processable.

The PTE adds new content (legal-person attestation, mandate, supporting credentials, trust score), so it is structurally an **Extension**, not a CIUS. The implication: receivers that don't understand the PTE namespace must still be able to process the underlying invoice as a normal EN 16931 / Peppol BIS document. This is achievable — the STM and TVE live in `<ext:UBLExtensions>`, which is precisely where EN 16931 says extension content goes; ignoring unknown UBL extensions is the conformant behaviour.

**Standardization-route consequence:** the cleanest path is to publish the PTE first as a **Peppol BIS extension** (governed by OpenPeppol's Coordination Committee, with schematron rules in the OpenPeppol GitHub), and separately as an **EN 16931 extension under TR 16931-5** (governed by CEN/TC 434), so that XRechnung, CIUS-PT, and any future EN 16931-conformant profile can adopt it without going through Peppol governance. This dual track maximises reach without forcing Peppol on national rails that don't use it.

---

## 3. Architecture overview

### 3.1 Design principle: the trust envelope is transport-independent

The PTE is a **data structure**, not a wire format. Its two containers (STM and TVE) and the qualified seal between them are properties of the document and its trust artifacts, not of any specific transport. The same PTE can ride:

- Inside a Peppol BIS UBL invoice (`<ext:UBLExtensions>`), transmitted over AS4 — the worked example in this document.
- As a MIME multipart attachment to an email, delivered over SMTP/IMAP.
- As an attachment in a QERDS (qualified electronic registered delivery) envelope under eIDAS Art. 44.
- As a request body in a direct REST API call between two organisations.
- As a verifiable presentation pulled by a buyer's wallet (OID4VP / DCQL).

In every binding the trust artifacts themselves are unchanged: the same credentials, the same seal, the same EUBW assertion. Only the wire-level packaging differs.

The transport layer is therefore left strictly untouched by the PTE:

| Layer | Trust scope | Touched by the PTE? |
|---|---|---|
| Transport (AS4, SMTP, QERDS-MIME, HTTP, etc.) | Hop-by-hop, transport-PKI-dependent | **No — unchanged** (backward compatibility preserved) |
| Document + PTE (carried by the transport) | End-to-end (C1 → C4), EUBW-anchored | **Yes — carries the trust artifacts** |

The PTE artifacts must survive each transport hop and be independently verifiable by the receiver, so they cannot live in the transport envelope. Where in the document they live, and how the binding's carrier element wraps them, is binding-specific (see §3.7 and [../bindings/](../bindings/)).

### 3.2 Component view — four corners, two service relationships per corner

Each legal entity (C1 sender, C4 receiver) uses **two operationally separate services**, possibly from different providers with a trust relationship between them:

- A **Peppol provider** (Peppol C2 on the send side, Peppol C3 on the receive side) — handles AS4 transport, Peppol BIS validation, UBL parsing.
- An **EUBW provider** (EUBW C2 on the send side, EUBW C3 on the receive side) — handles credential lifecycle, attestation issuance, mandate management, revocation, verification.

The Peppol provider and EUBW provider for the same legal entity **need not be the same party**, but a trust relationship between them exists (e.g. an `IntermediationCredential` issued from the legal entity's EUBW to its Peppol provider, authorising it to seal and submit on behalf of the entity). EUBW providers trust each other via the **EUBW Trusted List of Business Wallet Providers** (Commission-published), the same way QTSPs trust each other today via the eIDAS Trusted Lists.

```
              SENDER side                                    RECEIVER side
   ┌──────────────────────┐                       ┌──────────────────────┐
   │ C1 legal entity      │                       │ C4 legal entity      │
   │ ERP (Peppol BIS UBL) │                       │ ERP (Peppol BIS UBL) │
   └─────┬──────────┬─────┘                       └─────▲──────────▲─────┘
         │ uses     │ uses                              │ used by  │ used by
         ▼          ▼                                   │          │
   ┌─────────┐ ┌──────────┐                       ┌──────────┐ ┌─────────┐
   │ EUBW    │ │ Peppol   │                       │ Peppol   │ │ EUBW    │
   │ C2      ├▶│ C2 (AP)  │                       │ C3 (AP)  │▶│ C3      │
   │ sender  │ │          │                       │          │ │ receiver│
   │ wallet  │ │ builds   │                       │ receives │ │ wallet  │
   │ issues  │ │ STM +    │                       │ AS4,     │ │ verifies│
   │ creds,  │ │ seals,   │                       │ asks     │ │ STM     │
   │ holds   │ │ sends    │                       │ EUBW C3  │ │ creds   │
   │ mandate │ │ AS4      │                       │ to       │ │ against │
   │         │ │          │                       │ verify,  │ │ EUBW C2 │
   │         │ │          │                       │ writes   │ │ via TL  │
   │         │ │          │                       │ TVE ext  │ │         │
   └────┬────┘ └────┬─────┘                       └────┬─────┘ └────┬────┘
        │           │                                  ▲            │
        │           │      AS4 + sealed STM            │            │
        │           └──────────────────────────────────┘            │
        │                                                           │
        │           EUBW Trusted List (Commission-published)        │
        └───────────────────────────────────────────────────────────┘
                   EUBW C3 resolves sender credentials via EUBW C2
                          (presentation requests + lookups)
```

The diagram is a **sandwich**: the two Peppol Access Points (C2, C3) sit in the middle handling AS4 transport, and the two EUBW wallets (C2, C3) sit on the outside as wrappers handling credentials, mandates, and trust verification. C1 and C4 sit on top, talking only to their own pair (Peppol AP for transport, EUBW wallet for credential lifecycle). The wallet-to-wallet trust path (EUBW C2 ↔ EUBW C3) runs underneath via the Commission's Trusted List, completely outside the Peppol network.

**Three properties that fall out of this topology:**

1. **C1 and C4 keep their existing integrations.** C1 sends a normal Peppol BIS UBL to its Peppol C2 as it does today; C4 receives a Peppol BIS UBL with one extra `<ext:UBLExtension>` from its Peppol C3. Neither needs new protocols, new wallet calls, or new infrastructure. C4's ERP needs only to **parse one new extension** to surface the trust score in its booking UI.
2. **The integration work lives entirely at the Peppol providers (C2, C3).** They are the ones adding wallet-awareness to their existing AS4 pipelines. This is also where the commercial value lives — the Peppol providers sell the trust-enhanced service.
3. **Wallet-to-wallet trust crosses the EUBW Trusted List.** EUBW C3 does not need a bilateral relationship with the sender's EUBW C2; it just needs both to be on the Commission's Trusted List. Same pattern as eIDAS TLs for QTSPs today.

### 3.3 Processing flow

**Send side:**

1. **C1:** ERP produces a Peppol BIS UBL invoice in batch and forwards it to Peppol C2 — exactly as today. No change for C1.
2. **Peppol C2 ↔ EUBW C2 integration:** Peppol C2 calls EUBW C2 to:
   - obtain references to (or copies of) the sender's credentials needed for the STM — `LegalPersonAttestation`, `IntermediationMandate` (proving Peppol C2 is authorised to seal on C1's behalf), `LPIDCredential`, `VATCredential`(s), `IBANCredential`(s);
   - **optionally** obtain an `ApprovedSupplierCredential` — see §3.3.1;
   - invoke the QSeal (EUBW C2 either holds the seal key itself, or fronts the QTSP that does, see §4.6.3).
3. **Peppol C2:** runs today's schematron validation, builds the STM extension with the credentials it just obtained, applies the XAdES seal (§4.6), optionally writes a pre-send `SenderValidation` entry into the TVE, performs SMP/SML lookup, sends AS4. **AS4/Peppol-PKI hop signing unchanged.**

**Receive side:**

4. **Peppol C3:** receives the AS4 message, decrypts, performs today's Peppol BIS schematron validation. Unwraps SBDH, extracts the UBL document. Detects the STM extension.
5. **Peppol C3 ↔ EUBW C3 integration:** Peppol C3 hands the STM (and the sealed UBL digest) to EUBW C3 over an internal protocol (§4.9). EUBW C3:
   - verifies the XAdES seal against the eIDAS TL;
   - resolves each STM credential by calling **EUBW C2** (the sender's wallet endpoint, declared in `STM/Sender/EUBWEndpoint`), authenticated under the EUBW Trusted List;
   - performs live revocation checks against the sender's Bitstring Status Lists;
   - **applies receiver policy** — if the receiver's policy requires an `ApprovedSupplierCredential` and the STM does not contain one, EUBW C3 issues a **presentation request to EUBW C2** to retrieve it on demand (see §3.3.1);
   - computes the trust score (§4.8);
   - returns the score, dimensional findings, and a signed assertion to Peppol C3.
6. **Peppol C3:** writes the validation result as a new `<ext:UBLExtension>` (`urn:peppol:trust:pte:1.0:tve`) into the UBL document (§4.9.3), appended **after** the seal — the seal does not cover the TVE because the TVE did not exist at seal time (§3.5).
7. **C4:** receives a Peppol BIS UBL with two new extensions (STM, TVE) alongside the existing seal extension. Its ERP parses the TVE to read the score and decide booking action. **No new integration; just one new extension to parse.**

#### 3.3.1 Approved-supplier credential — push at send, pull at receive

The `ApprovedSupplierCredential` is the credential where buyers can express "I only book invoices from suppliers I've pre-approved." Unlike LPID/VAT/IBAN credentials (which apply to every sender regardless of who's receiving), this credential is **buyer-specific**: it is issued by the buyer (or by a qualifier acting on the buyer's behalf) to the supplier, and stored in the supplier's EUBW C2.

There are two delivery paths for this credential, used in combination:

**Push at send (Peppol C2 → STM, preferred when known):**

- If Peppol C2 knows from the recipient lookup (SMP / Peppol Directory metadata, or out-of-band buyer configuration) that C4 requires this credential, it asks EUBW C2 for it and includes it in the STM at send time.
- This is the happy path: the credential travels with the invoice, no extra round-trip at the receiver, EUBW C3 verifies it inline with the rest of the STM.

**Pull at receive (EUBW C3 → EUBW C2 presentation request, fallback):**

- If the credential is not in the STM but the receiver's policy requires it, EUBW C3 issues a **presentation request** to EUBW C2 (the sender's wallet, identified in `STM/Sender/EUBWEndpoint`).
- The presentation request is scoped: "for verifying invoice with digest `f7c2a1...`, please present the `ApprovedSupplierCredential` issued by `urn:eubw:euid:NLNL1234.567.890`" (the buyer's EUID).
- EUBW C2 either holds the credential and presents it (with the same cryptographic verification properties as if it had been in the STM), or doesn't, and returns `not_available`.
- This is the fill-in path: it uses the standard EUBW wallet-to-wallet presentation protocol, no new mechanism needed.

**Why both paths matter:**

| Scenario | Push at send works? | Pull at receive needed? |
|---|---|---|
| Buyer's approval requirement is well-known (e.g. large buyer with public supplier programme) | Yes — Peppol C2 includes it in the STM | No |
| Per-transaction buyer policy, or buyer's requirement is dynamic | Peppol C2 might not know | Yes — EUBW C3 fills the gap |
| Sender uses multiple Peppol providers, each with different buyer-awareness | Partial — one provider knows, another doesn't | Yes — receiver-side fill-in normalises this |
| Supplier was approved after the invoice was sealed (credential newer than invoice) | No — credential didn't exist at send | Yes — pull at receive picks up the new credential |

The pull-at-receive path is also what handles **credential rotation**: if the supplier's approval was renewed (new credential issued, old one revoked) between send and receive, EUBW C3 gets the current credential rather than a stale embedded copy.

**Policy and scoring (see §4.8):**

The receiver's scoring policy decides what to do with the `supplier-approval` dimension:

- **Hard requirement (weight 3, like seal-validity):** failure or absence forces the aggregate trust score to 0 → hard decline. Used by buyers operating strict supplier-approval programmes where unapproved suppliers must not be booked.
- **Soft dimension (weight 1 default):** failure or absence reduces the aggregate score; the receiver's threshold map (§4.8.4) decides whether to hold, review, or accept anyway. Used by buyers who treat approved-supplier as a preference rather than a hard rule.

The `unverified` status (§4.8.2) is distinct from `failed`: if the buyer's policy does not require approved-supplier, the dimension is `unverified` and contributes nothing to the score. If the policy does require it but EUBW C3 cannot obtain the credential via either path, the dimension is `failed` (score 0).

### 3.4 Discovery and graceful degradation

The SMP advertises a new capability identifier, e.g. `peppol-trust-envelope-v1`, as a supported document-type / transport attribute, so C2 knows whether C3/C4 can consume the extension. If the receiver does not support it, the PTE is ignorable metadata and the invoice still flows through the standard rails.

### 3.5 Container split — resolving the seal-scope problem

A single container holding both pre-seal credentials and post-seal validation artifacts cannot exist: anything covered by the qualified seal must be byte-stable at seal time, and anything produced after sealing (validation reports, transit timestamps, receiver verification results) must sit outside the digested scope.

The PTE is therefore **split into two containers** with different lifecycles, locations, and seal coverage:

| Container | Location | Lifecycle | Covered by seal? | Holds |
|---|---|---|---|---|
| **STM** — Sealed Trust Manifest | Inside the document, in the binding's carrier (Peppol binding: `<ext:UBLExtensions>`) | Created by the sender's handler **before** sealing; frozen at seal time | **Yes** (everything except the seal element itself and the prospectively-excluded TVE carrier) | Legal-person attestation (ref or inline), intermediation mandate (ref or inline), supporting credentials (IBAN, LPID, VAT, supplier — refs or inline), trust-anchor pointers, the document body |
| **TVE** — Trust Validation Envelope | Inside the document, in the binding's carrier, as a sibling of the STM | Optional pre-send entry by the sender's handler; mandatory post-receipt entry by the receiver's handler (carrying the signed assertion from receiver EUBW) | **No** (prospectively excluded by the seal's binding-specific transform) | Sealed-document digest, sender validation (optional), receiver validation with EUBW assertion, downstream receipts |

Both containers cross-reference each other: the TVE carries the digest of the sealed document so post-seal additions are anchored to the exact sealed bytes. The STM carries no references to the TVE — it cannot, because the TVE does not exist yet at seal time.

This split also removes the loop in §2.3: in the Peppol binding the qualified seal is enveloped inside `<ext:UBLExtensions>` using the standard XMLDSig `enveloped-signature` transform, with an XPath transform that prospectively excludes the TVE carrier. Other bindings apply analogous mechanisms (e.g. PAdES with content-excluding signed-attribute rules for PDF; CAdES with detached-from-TVE arrangement for mail-binding MIME parts).

The TVE is always placed adjacent to the STM in the binding's carrier — never in a transport envelope (Peppol SBDH, mail headers, QERDS envelope metadata), because transport envelopes typically do not survive to C4. The protocol that produces the TVE content is APEIP `/verify` (see §4.9 and [../protocol/APEIP-Specification.md](../protocol/APEIP-Specification.md)).

### 3.6 Extension-point governance — who is allowed to add what

The two physical extension points have very different governance regimes. This shapes which artifact can live where.

| Extension point | Governing body | Gatekeeping | Practical consequence |
|---|---|---|---|
| `<ext:UBLExtensions>` in the UBL document | OASIS UBL TC (defines the *mechanism*); content is "out of scope" of UBL itself | **No central authority for content.** UBL explicitly states `UBLExtensions` exists for "foreign content not defined by the business vocabulary." Anyone may declare a namespace and add content. | Peppol BIS Billing 3.0 schematron does not forbid arbitrary extensions, but extensions that conflict with EN 16931 or BIS cardinalities will fail validation. National Peppol Authorities (BE, FR, IT) routinely add country-specific UBLExtensions for their CTC regimes. **STM and TVE placement requires no OpenPeppol approval to deploy bilaterally.** |
| SBDH `BusinessScope/Scope` (UN/CEFACT layer) | UN/CEFACT / GS1 | Generic typed-scope element; anyone may add scope types in principle. | Out of scope for this design — SBDH does not reach C4 (it is stripped at C3), so there is no benefit to placing trust artifacts there. |
| Peppol Business Message Envelope (Peppol's profile of SBDH) | **OpenPeppol** | Gatekept. Specific scope types only. | Out of scope for this design (see row above). |

**Standardization route for this extension:**

1. **STM (UBL extension in the Peppol binding):** declare the `urn:peppol:trust:pte:1.0:stm` namespace; publish schema. Bilateral pilots can run immediately. A Peppol BIS profile recognising the namespace (so APs treat it as known-good) requires an OpenPeppol work item, but is not blocking for deployment.
2. **TVE (UBL extension in the Peppol binding):** declare `urn:peppol:trust:pte:1.0:tve` namespace; same governance posture as the STM — no OpenPeppol gatekeeper for the *mechanism*. Written by Peppol C3 after receipt; reaches C4 as part of the UBL document.
3. **APEIP — AP ↔ EUBW Integration Protocol:** the protocol between a message-handling intermediary (Peppol Access Point, mail server, QERDS provider, REST endpoint) and its companion EUBW provider. Transport-independent. Specified in [../protocol/APEIP-Specification.md](../protocol/APEIP-Specification.md). Profile defined under EUBW Implementing Acts; invisible to C1, C4, and to the transport layer.

**Out of scope for this document:**

- **SBDH-layer extensions** (Peppol binding): SBDH is a transport-layer construct that does not reach C4 (Peppol MLS spec confirms C3 forwards "at least the business document" — the SBDH envelope is typically dropped). Any artifact intended for C4 must live in the UBL document.
- **Transport-specific carrier details for non-Peppol bindings**: see the per-binding documents under [../bindings/](../bindings/).

### 3.7 Transport bindings — Peppol is one of N

The PTE is a transport-independent envelope. A **transport binding** specifies how the PTE is packaged on a specific wire, where the message-handling intermediary sits, and how APEIP integrates the intermediary with its companion EUBW. Each binding answers three questions:

| Question | What the binding specifies |
|---|---|
| **Carrier** | Which container element/structure carries the STM, the seal, and the TVE on the wire (e.g. `<ext:UBLExtensions>` in UBL; MIME multipart in mail; an attachment slot in the QERDS envelope; a JSON property in a REST request body). |
| **Handler** | Which software role acts as the "AP" — the message handler that constructs the STM at send, calls APEIP `/verify` at receive, and writes the TVE back into the document. |
| **Discovery** | How a sender determines whether a receiver supports the PTE (e.g. SMP capability identifier in Peppol; DNS TXT record or auto-discovery in mail; capability advertisement in QERDS). |

The binding does **not** redefine the PTE itself. The same STM bytes, the same XAdES (or PAdES, where appropriate) seal, the same APEIP `/verify` response live across all bindings. The transport just chooses a carrier.

**Current binding inventory:**

| Binding | Status | Use case | Notes |
|---|---|---|---|
| **Peppol BIS UBL over AS4** | Worked example throughout this document; reference implementation in [../bindings/peppol/](../bindings/peppol/) | B2B e-invoicing across the Peppol network (Belgium B2B 2026, Norway EHF, etc.) | UBL `<ext:UBLExtensions>` carrier; XAdES seal via UBL DSig profile; Oxalis-NG handler |
| **Mail (SMTP/IMAP)** | Sketch in [../bindings/mail/Mail-Binding.md](../bindings/mail/Mail-Binding.md) | B2B exchanges that still ride on email; B2C invoice delivery with trust signals | MIME multipart carrier; XAdES or CAdES seal over the document part; mail-server handler |
| **QERDS (eIDAS Art. 44)** | Sketch in [../bindings/qerds/QERDS-Binding.md](../bindings/qerds/QERDS-Binding.md) | Registered-delivery scenarios where a QTSP-provided evidence layer is required (legal contracts, formal notices) | QERDS evidence + PTE attachment; QERDS provider acts as the handler |
| **REST direct B2B** | Future | Bilateral B2B integrations that bypass any e-invoicing network | JSON property in request body; HTTP server as handler |
| **OID4VP / DCQL** | SC5 Scenario 4 territory; not modelled in this document | Buyer wallet pulls the invoice from supplier wallet via verifiable presentation | Different architecture (buyer-initiated, wallet-to-wallet); only mentioned here for completeness |

The worked example throughout this document uses the **Peppol binding** because it is the most concrete and the binding nearest to production deployment. Where examples show UBL elements, namespaces like `urn:peppol:trust:pte:1.0:stm`, or XAdES enveloped signatures via the UBL DSig profile, those details are **Peppol-binding-specific**. The abstract data model (STM contents, TVE contents, seal coverage, APEIP integration) is binding-agnostic. The per-binding documents under [../bindings/](../bindings/) explain how the same data model materialises in each other transport.

---

## 4. PTE schema

**Format note (Peppol binding).** In the Peppol binding the PTE schema is **XML**, because UBL (OASIS XML standard) is XML and `<ext:UBLExtensions>/<ext:UBLExtension>/<ext:ExtensionContent>` accepts XML element trees. JSON payloads can be carried inside the XML containers (e.g. an SD-JWT VC as a string-valued element), and the APEIP protocol uses JSON regardless of binding. Other transport bindings may use other carriers: a MIME multipart attachment for the mail binding, an attachment slot for QERDS, etc. The data model is the same; the wire format follows the binding.

**"PTE" is an umbrella term** covering the two logical containers defined in §3.5: the **STM** (Sealed Trust Manifest, written by the sender's message handler before sealing, covered by the qualified seal) and the **TVE** (Trust Validation Envelope, written by the receiver's message handler after verification, not covered by the seal). Where this document says "the PTE", read it as "the pair STM + TVE".

The examples below show the **Peppol binding's** realisation: STM and TVE as sibling `<ext:UBLExtension>` elements in a Peppol BIS Billing 3.0 UBL document, with the seal applied via the OASIS UBL DSig enveloped XAdES profile. Other bindings realise the same data model in their own carriers (see [../bindings/](../bindings/)).

### 4.1 Logical structure — two containers

The schema defines two containers with disjoint lifecycles. Both use the same XML namespace `urn:peppol:trust:pte:1.0` but live in different parts of the document.

#### 4.1.1 STM — Sealed Trust Manifest (in `<ext:UBLExtensions>`, covered by seal)

```
SealedTrustManifest (STM)
├─ @version, @profileID
├─ Sender
│   └─ EUBWEndpoint                    URI of sender's EUBW (for credential resolution by C3)
├─ LegalPersonAttestation         [1]  — authentication (§2.1)
├─ IntermediationMandate          [0..1] — authorization (§2.2)
├─ SupportingCredentials          [0..1] — invoice-data verification (§2.4, §4.7)
│   ├─ IBANCredential             [0..*]   one per IBAN cited in invoice
│   ├─ LPIDCredential             [0..1]   sender's qualified Legal Person ID attestation
│   ├─ VATCredential              [0..*]   sender VAT, plus per-jurisdiction VAT for cross-border
│   └─ ApprovedSupplierCredential [0..1]   receiver-required, where applicable
├─ TrustAnchorPointers            [1]    EUBW Trusted List(s), eIDAS TL, EUBW directory
└─ <ds:Signature>                 [1]    XAdES qualified seal — covers STM + invoice body, excludes itself
```

Every element under `Sender`, `Legal*`, `Intermediation*`, `Supporting*`, and `TrustAnchor*` is **inside the digested scope**. The `<ds:Signature>` is the only element excluded (via XMLDSig `enveloped-signature` transform).

#### 4.1.2 TVE — Trust Validation Envelope (in `<ext:UBLExtensions>`, post-seal, not covered by seal)

The TVE is a UBL extension that **Peppol C3 writes after verification**, containing the trust score returned by EUBW C3 and any pre-send score written by Peppol C2. It sits in `<ext:UBLExtensions>` as a sibling of the STM and the seal extension, **outside the digested scope** (because it doesn't exist at seal time).

```
TrustValidationEnvelope (TVE)
├─ @version, @profileID
├─ SealedDocumentDigest           [1]    digest of the sealed UBL document — anchors all post-seal content
├─ SenderValidation               [0..1] Peppol C2's pre-send validation (optional, sender-side preview)
│   ├─ TrustScore                          numeric, see §4.8
│   ├─ DimensionalScores
│   ├─ Findings                            structured per-credential results
│   └─ AssertionSeal                       optional QSeal by Peppol C2 over this assertion
├─ ReceiverValidation             [1]    Peppol C3's verification result, based on EUBW C3's assertion
│   ├─ TrustScore
│   ├─ DimensionalScores
│   ├─ Findings
│   ├─ EUBWAssertion                       signed assertion returned by EUBW C3 (JWS or JAdES)
│   └─ AssertionSeal                       optional QSeal by Peppol C3 wrapping the above
└─ DownstreamReceipts             [0..*] e.g. CTC corner-5 acknowledgement
```

**Who writes what:**

| Element | Written by | When |
|---|---|---|
| `SealedDocumentDigest` | Peppol C2 | After sealing, before send |
| `SenderValidation` | Peppol C2 | After sealing, before send (optional preview) |
| `ReceiverValidation` | Peppol C3 | After receiving AS4 and consulting EUBW C3 |
| `EUBWAssertion` inside `ReceiverValidation` | EUBW C3 (Peppol C3 inlines it) | Returned to Peppol C3 by EUBW C3 |
| `DownstreamReceipts` | C5 or downstream consumers | After delivery to C4 |

The `EUBWAssertion` is the authoritative artifact: it is signed by EUBW C3 (an EUBW Trusted List member) and represents the verification of the STM credentials against the sender's EUBW C2. Peppol C3 surfaces it in the TVE; Peppol C3's optional `AssertionSeal` wrap is a value-add (Peppol C3 attesting "I delivered this; I consulted EUBW C3; this is what came back"), not a replacement for the EUBW assertion itself.

Each credential-bearing element under the STM (`LegalPersonAttestation`, `IntermediationMandate`, every `SupportingCredentials` child) is a `CredentialBearingElement` per §4.5.2, which means it can be either **resolved by reference** to the sender's EUBW C2 (preferred — gives EUBW C3 live revocation status) or **inlined** (fallback when offline verification is required). See §4.7.

Cardinality notation: `[1]` mandatory single, `[0..1]` optional single, `[0..*]` optional multiple.

### 4.2 Illustrative XML — STM in `<ext:UBLExtensions>`

The Sealed Trust Manifest sits inside the invoice's UBL extensions. Two extension entries: one for the STM content, one for the XAdES signature (UBL DSig profile). The seal covers everything in the UBL document except the signature element itself.

```xml
<Invoice xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"
         xmlns:ext="urn:oasis:names:specification:ubl:schema:xsd:CommonExtensionComponents-2"
         xmlns:pte="urn:peppol:trust:pte:1.0">
  <ext:UBLExtensions>

    <!-- Extension 1: Sealed Trust Manifest (STM) -->
    <ext:UBLExtension>
      <ext:ExtensionURI>urn:peppol:trust:pte:1.0:stm</ext:ExtensionURI>
      <ext:ExtensionContent>
        <pte:SealedTrustManifest version="1.0" profileID="urn:peppol:trust:pte:1.0">

          <pte:Sender>
            <pte:EUBWEndpoint>https://eubw.example-trading.be</pte:EUBWEndpoint>
          </pte:Sender>

          <pte:LegalPersonAttestation>
            <pte:Format>w3c-vc</pte:Format>
            <pte:MediaType>application/vc+jwt</pte:MediaType>
            <pte:Resolution mode="byReference">
              <pte:CredentialURI>https://eubw.example-trading.be/vc/lpa/42</pte:CredentialURI>
              <pte:CredentialDigest algorithm="SHA-256">a3f2...</pte:CredentialDigest>
            </pte:Resolution>
            <pte:IssuerTrustListRef>https://ec.europa.eu/eubw/tl/BE-001</pte:IssuerTrustListRef>
            <pte:BindingMetadata>
              <pte:EUID>BEBE0123.456.789</pte:EUID>
              <pte:PeppolParticipantId scheme="9925">BE0123456789</pte:PeppolParticipantId>
            </pte:BindingMetadata>
          </pte:LegalPersonAttestation>

          <pte:IntermediationMandate>
            <pte:Format>w3c-vc</pte:Format>
            <pte:MediaType>application/vc+jwt</pte:MediaType>
            <pte:Resolution mode="byReference">
              <pte:CredentialURI>https://eubw.example-trading.be/vc/mandate/9f2c</pte:CredentialURI>
              <pte:CredentialDigest algorithm="SHA-256">b41d...</pte:CredentialDigest>
            </pte:Resolution>
            <pte:BindingMetadata>
              <pte:Mandator euid="BEBE0123.456.789"/>
              <pte:Mandatee euid="BEBE0987.654.321"/>
              <pte:Scope>e-invoicing:seal+submit</pte:Scope>
            </pte:BindingMetadata>
          </pte:IntermediationMandate>

          <pte:SupportingCredentials>
            <pte:LPIDCredential>
              <pte:Format>w3c-vc</pte:Format>
              <pte:Resolution mode="byReference">
                <pte:CredentialURI>https://eubw.example-trading.be/vc/lpid</pte:CredentialURI>
                <pte:CredentialDigest algorithm="SHA-256">c128...</pte:CredentialDigest>
              </pte:Resolution>
            </pte:LPIDCredential>

            <pte:VATCredential jurisdiction="BE">
              <pte:Resolution mode="byReference">
                <pte:CredentialURI>https://eubw.example-trading.be/vc/vat/BE</pte:CredentialURI>
                <pte:CredentialDigest algorithm="SHA-256">d937...</pte:CredentialDigest>
              </pte:Resolution>
              <pte:BindingMetadata>
                <pte:VATNumber>BE0123456789</pte:VATNumber>
                <pte:LinkedInvoiceField>cac:AccountingSupplierParty/cac:Party/cac:PartyTaxScheme</pte:LinkedInvoiceField>
              </pte:BindingMetadata>
            </pte:VATCredential>

            <pte:IBANCredential>
              <pte:Resolution mode="byReference">
                <pte:CredentialURI>https://eubw.example-trading.be/vc/iban/01</pte:CredentialURI>
                <pte:CredentialDigest algorithm="SHA-256">e4a1...</pte:CredentialDigest>
              </pte:Resolution>
              <pte:BindingMetadata>
                <pte:IBAN>BE68539007547034</pte:IBAN>
                <pte:LinkedInvoiceField>cac:PaymentMeans/cac:PayeeFinancialAccount/cbc:ID</pte:LinkedInvoiceField>
              </pte:BindingMetadata>
            </pte:IBANCredential>

            <pte:ApprovedSupplierCredential>
              <pte:Resolution mode="inline">
                <pte:CredentialValue><!-- VC bytes --></pte:CredentialValue>
              </pte:Resolution>
              <pte:IssuerTrustListRef>https://buyer.example.com/approved-suppliers/tl</pte:IssuerTrustListRef>
            </pte:ApprovedSupplierCredential>
          </pte:SupportingCredentials>

          <pte:TrustAnchorPointers>
            <pte:EUBWTrustList>https://ec.europa.eu/eubw/tl</pte:EUBWTrustList>
            <pte:eIDASTrustList>https://ec.europa.eu/tools/lotl/eu-lotl.xml</pte:eIDASTrustList>
            <pte:EUBWDirectory>https://ec.europa.eu/eubw/directory</pte:EUBWDirectory>
          </pte:TrustAnchorPointers>

        </pte:SealedTrustManifest>
      </ext:ExtensionContent>
    </ext:UBLExtension>

    <!-- Extension 2: XAdES enveloped seal (UBL DSig profile) -->
    <ext:UBLExtension>
      <ext:ExtensionURI>urn:oasis:names:specification:ubl:dsig:enveloped:xades</ext:ExtensionURI>
      <ext:ExtensionContent>
        <sig:UBLDocumentSignatures
            xmlns:sig="urn:oasis:names:specification:ubl:schema:xsd:CommonSignatureComponents-2"
            xmlns:sac="urn:oasis:names:specification:ubl:schema:xsd:SignatureAggregateComponents-2"
            xmlns:sbc="urn:oasis:names:specification:ubl:schema:xsd:SignatureBasicComponents-2">
          <sac:SignatureInformation>
            <cbc:ID>urn:oasis:names:specification:ubl:signature:1</cbc:ID>
            <ds:Signature Id="PeppolTrustSeal" xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
              <!-- XAdES content per §4.6.4 -->
            </ds:Signature>
          </sac:SignatureInformation>
        </sig:UBLDocumentSignatures>
      </ext:ExtensionContent>
    </ext:UBLExtension>

  </ext:UBLExtensions>
  <!-- standard EN 16931 / Peppol BIS invoice body (cbc:CustomizationID, cac:AccountingSupplierParty, etc.) -->
</Invoice>
```

The XPath transform on the seal's `<ds:Reference>` excludes the `<ext:UBLExtension>` whose `ExtensionURI` is `urn:oasis:names:specification:ubl:dsig:enveloped:xades` **and** the `<ext:UBLExtension>` whose `ExtensionURI` is `urn:peppol:trust:pte:1.0:tve` — so the STM extension **is** in the digest, the seal itself is **not**, and the TVE (which Peppol C3 adds after receipt) is **not**. See §4.6.4 step 3.

The TVE extension is **not present at seal time**. Peppol C3 inserts it as a third `<ext:UBLExtension>` in this same `<ext:UBLExtensions>` element after receiving and verifying. The TVE example is in §4.3.

### 4.3 Illustrative XML — TVE as UBL extension

The Trust Validation Envelope is appended by Peppol C3 as a new `<ext:UBLExtension>` in the same UBL document that carries the STM. It is **outside the seal's digested scope** because it didn't exist at seal time. The `<pte:EUBWAssertion>` inside `<pte:ReceiverValidation>` is the signed assertion returned by EUBW C3 — the authoritative artifact; Peppol C3 only embeds it.

```xml
<ext:UBLExtension>
  <ext:ExtensionURI>urn:peppol:trust:pte:1.0:tve</ext:ExtensionURI>
  <ext:ExtensionContent>
    <pte:TrustValidationEnvelope xmlns:pte="urn:peppol:trust:pte:1.0"
                                 version="1.0" profileID="urn:peppol:trust:pte:1.0">

      <pte:SealedDocumentDigest algorithm="SHA-256">
        f7c2a1...
      </pte:SealedDocumentDigest>

      <pte:SenderValidation timestamp="2026-05-27T10:24:01Z"
                            actor="urn:eubw:euid:BEBE0987.654.321">
        <pte:TrustScore value="94" scale="0-100" policy="urn:peppol:trust:scoring:v1"/>
        <pte:DimensionalScores>
          <pte:Score dimension="seal-validity" value="100"/>
          <pte:Score dimension="legal-person-identity" value="100"/>
          <pte:Score dimension="intermediation-authority" value="100"/>
          <pte:Score dimension="vat-consistency" value="100"/>
          <pte:Score dimension="iban-consistency" value="90"/>
          <pte:Score dimension="lpid-freshness" value="85"/>
          <pte:Score dimension="format-compliance" value="100"/>
          <pte:Score dimension="supplier-approval" value="80"/>
        </pte:DimensionalScores>
        <pte:Findings>
          <pte:Finding code="IBAN-OK" severity="info">
            IBAN BE68539007547034 matches IBANCredential, holder = sender EUID
          </pte:Finding>
        </pte:Findings>
        <pte:AssertionSeal>
          <!-- Peppol C2's QSeal over this SenderValidation element -->
        </pte:AssertionSeal>
      </pte:SenderValidation>

      <pte:ReceiverValidation timestamp="2026-05-27T10:24:08Z"
                              actor="urn:peppol:c3:provider:nl"
                              eubwActor="urn:eubw:euid:NLNL5555.666.777">
        <pte:TrustScore value="91" scale="0-100" policy="urn:peppol:trust:scoring:v1"/>
        <pte:DimensionalScores>
          <pte:Score dimension="seal-validity" value="100"/>
          <pte:Score dimension="legal-person-identity" value="100"/>
          <pte:Score dimension="intermediation-authority" value="100"/>
          <pte:Score dimension="vat-consistency" value="100"/>
          <pte:Score dimension="iban-consistency" value="90"/>
          <pte:Score dimension="lpid-freshness" value="85"/>
          <pte:Score dimension="format-compliance" value="100"/>
          <pte:Score dimension="supplier-approval" value="60"/>
        </pte:DimensionalScores>
        <pte:Findings>
          <pte:Finding code="SUPPLIER-APPROVAL-INLINE" severity="warning">
            ApprovedSupplierCredential delivered inline rather than via sender EUBW C2;
            EUBW C3 could not perform live revocation check, lower confidence per policy
          </pte:Finding>
        </pte:Findings>
        <pte:Decision action="accept-with-review" threshold="90"/>
        <pte:EUBWAssertion format="JWS">
          <!--
            JWS signed by EUBW C3 (NL5555.666.777, on EUBW Trusted List).
            Payload: anchored-to invoice digest, trust score, dimensional scores,
            findings, validation time, link to credentials resolved against EUBW C2.
            This is the authoritative artifact.
          -->
          eyJhbGciOiJFUzI1NiIsImtpZCI6IkVVQlctQzMtTkwtMDEifQ...
        </pte:EUBWAssertion>
        <pte:AssertionSeal>
          <!-- Optional: Peppol C3's QSeal wrapping the above (chain of custody) -->
        </pte:AssertionSeal>
      </pte:ReceiverValidation>

    </pte:TrustValidationEnvelope>
  </ext:ExtensionContent>
</ext:UBLExtension>
```

Four properties of this structure:

1. **`SealedDocumentDigest` anchors the TVE to the exact sealed bytes.** Any modification to the UBL document after sealing invalidates this digest and therefore the TVE — verifiers re-derive the digest by canonicalising the UBL document with all `<ext:UBLExtension>` elements whose URI is `urn:peppol:trust:pte:1.0:tve` excluded, and check it matches.
2. **`SenderValidation` and `ReceiverValidation` are independently sealed**. Peppol C3's `ReceiverValidation` does not overwrite Peppol C2's `SenderValidation` — both are present and verifiable.
3. **`EUBWAssertion` is the authoritative trust artifact.** The score and findings inside `ReceiverValidation` are surfaced for ERP convenience; the signed `EUBWAssertion` is what an auditor or regulator would verify against the EUBW Trusted List. Peppol C3 cannot mint trust assertions independently — it can only deliver what EUBW C3 says.
4. **Two distinct actors in `ReceiverValidation`:** `actor` is Peppol C3 (who wrote the TVE entry), `eubwActor` is EUBW C3 (who produced the underlying assertion). They may be the same legal entity but they are distinct service roles.

### 4.4 Element reference

**STM elements (in `<ext:UBLExtensions>`, covered by seal):**

| Element | Card. | Function | Verified at |
|---|---|---|---|
| `Sender/EUBWEndpoint` | 1 | URI of sender EUBW for credential resolution | C2, C3 |
| `LegalPersonAttestation` | 1 | Authentication — binds EUID/VAT to participant ID | C3, C4 |
| `IntermediationMandate` | 0..1 | Authorization — C1 authorized C2 to seal/submit | C3 |
| `SupportingCredentials/LPIDCredential` | 0..1 | Qualified Legal Person ID attestation | C3 |
| `SupportingCredentials/VATCredential` | 0..* | Per-jurisdiction VAT validation | C3 |
| `SupportingCredentials/IBANCredential` | 0..* | Payment account validation, one per IBAN in invoice | C3 |
| `SupportingCredentials/ApprovedSupplierCredential` | 0..1 | Receiver-required supplier approval | C3 |
| `TrustAnchorPointers` | 1 | EUBW TL, eIDAS TL, EUBW directory URIs | C2, C3 |
| `<ds:Signature Id="PeppolTrustSeal">` | 1 | XAdES qualified seal — covers STM + invoice | C3, C4 |

**TVE elements (in `<ext:UBLExtensions>`, post-seal, not covered by seal):**

| Element | Card. | Function | Produced by |
|---|---|---|---|
| `SealedDocumentDigest` | 1 | Anchors TVE to exact sealed bytes | Peppol C2 |
| `SenderValidation` | 0..1 | Pre-send validation with trust score (optional preview) | Peppol C2 |
| `ReceiverValidation` | 1 | Post-receipt validation with trust score | Peppol C3 |
| `ReceiverValidation/EUBWAssertion` | 1 | Signed assertion from EUBW C3 (authoritative trust artifact) | EUBW C3 |
| `DownstreamReceipts` | 0..* | CTC corner-5 or buyer-side acknowledgements | C5, C4 |

---

### 4.5 W3C Verifiable Credentials compliance

**Important scope note.** This question is about **EUBW** (European Business Wallet, legal-person, draft regulation Nov 2025), **not EUDI** (natural-person wallet, eIDAS 2.0, Regulation 2024/1183). EUDI's ARF mandates SD-JWT VC + mdoc and restricts W3C VCDM 2.0 to non-qualified EAAs — but that ruleset applies to EUDI, not to EUBW. The EUBW credential format rules are not yet locked in published Implementing Acts as of May 2026. The PTE therefore cannot assume any single credential format.

#### 4.5.1 Design choice: format-agnostic carrier + normative W3C VCDM 2.0 profile

The PTE is designed as a **format-agnostic carrier**. The base schema accepts any EUBW-issued credential format — W3C VC (VCDM 2.0), SD-JWT VC, mdoc, or a future format — and treats credential-format compliance as a **per-instance property of the carried credential, not of the PTE itself**.

In addition, a **normative W3C VCDM 2.0 profile** is defined: when an issuer chooses to issue a legal-person attestation or mandate as a W3C VC, this profile specifies exactly how that VC is embedded in the PTE, how its proof is verified, and which W3C-defined invariants must hold.

#### 4.5.2 Format-agnostic carrier (base schema)

Every credential-bearing element in §4.1 (`LegalPersonAttestation`, `IntermediationMandate`, `SupplierCredential`) is structured as a wrapper with three discriminators:

```
<CredentialBearingElement>
  <Format>            <!-- w3c-vc | sd-jwt-vc | mdoc | (extensible)        -->
  <MediaType>         <!-- application/vc+jwt | application/vc+sd-jwt | application/mdoc | application/vc+ld+json -->
  <CredentialValue>   <!-- the credential bytes, encoding determined by Format/MediaType -->
  <IssuerTrustListRef>
  <BindingMetadata>   <!-- format-independent fields PTE consumers need without parsing the credential -->
</CredentialBearingElement>
```

`Format` and `MediaType` together let C3 dispatch to the right verifier without parsing `CredentialValue`. `BindingMetadata` exposes the minimal fields (EUID, participant ID binding, validity period) needed for routing and pre-filter checks; the authoritative values live inside the signed credential.

Compliance posture: **the PTE itself is a Peppol/SBDH/UBL artifact and does not claim W3C VC conformance.** A specific credential instance carried in the PTE is W3C-VC-compliant if and only if it conforms to VCDM 2.0 and to one of the W3C-defined securing mechanisms (VC-JOSE-COSE or VC Data Integrity), independently of the PTE.

#### 4.5.3 Normative W3C VCDM 2.0 profile (when the carried credential is a W3C VC)

When `Format = w3c-vc`, the following constraints apply to the credential placed in `CredentialValue`. These are derived from the W3C Recommendations published 15 May 2025 (VCDM 2.0, VC-JOSE-COSE, VC Data Integrity 1.0, Bitstring Status List 1.0, Controlled Identifiers 1.0).

**Mandatory VCDM 2.0 invariants for the carried credential:**

| Requirement | Source |
|---|---|
| `@context` MUST include `https://www.w3.org/ns/credentials/v2` as the first value | VCDM 2.0 §4.2 |
| `type` MUST include `VerifiableCredential` plus a PTE-defined specific type (e.g. `EUBWLegalPersonAttestation`, `EUBWIntermediationMandate`, `ApprovedSupplierCredential`) | VCDM 2.0 §4.4 |
| `issuer` MUST be a URI resolvable to an entry in the EUBW Trusted List of Business Wallet Providers (Controlled Identifier) | VCDM 2.0 §4.5; Controlled Identifiers 1.0 |
| `validFrom` / `validUntil` MUST be present | VCDM 2.0 §4.7 |
| `credentialSubject` MUST contain the EUID of the legal person and a binding to the Peppol participant identifier | PTE profile |
| `credentialStatus` MUST use Bitstring Status List 1.0 with status purposes `revocation` and (where applicable) `suspension` | Bitstring Status List 1.0 |
| Proof MUST be one of: VC-JOSE-COSE (JWT, SD-JWT, or COSE) **or** VC Data Integrity (`DataIntegrityProof` with `ecdsa-rdfc-2019` or `ecdsa-jcs-2019` cryptosuite); cryptosuite MUST be on ENISA's agreed cryptographic mechanisms list | VC-JOSE-COSE; VC Data Integrity 1.0; Data Integrity ECDSA Cryptosuites 1.0 |
| Credential MUST be securable by either JOSE/COSE envelope **or** an embedded data-integrity proof; the chosen mechanism MUST be reflected in `MediaType` (`application/vc+jwt`, `application/vc+sd-jwt`, or `application/vc+ld+json`) | VC-JOSE-COSE; VC Data Integrity 1.0 |

**Specific credential types defined by this PTE profile:**

1. `EUBWLegalPersonAttestation` — populates `LegalPersonAttestation`. `credentialSubject` contains `euid`, `legalName`, `vatIdentifier`, `peppolParticipantId` (matching the SBDH sender participant ID).
2. `EUBWIntermediationMandate` — populates `IntermediationMandate`. `credentialSubject` contains `mandator` (EUID of C1), `mandatee` (EUID of C2 service provider), `scope` (e.g. `e-invoicing:seal+submit`), `validFrom`/`validUntil`.
3. `ApprovedSupplierCredential` — populates `SupplierCredential`. `credentialSubject` contains the supplier EUID and the issuer's qualification scheme reference.

**Verification at receipt:** EUBW C3 (invoked by Peppol C3 per §4.9) MUST (a) verify the credential's cryptographic proof per VC-JOSE-COSE or VC Data Integrity, (b) resolve the `issuer` against the EUBW Trusted List, (c) check status via the Bitstring Status List entry, (d) check `validFrom`/`validUntil`, (e) check that `credentialSubject.peppolParticipantId` matches the sender participant ID carried in the document, (f) for mandates, check `mandatee` matches the Peppol C2 service provider's EUID as carried in its Peppol AP certificate (cross-PKI binding — see §6 item 1).

#### 4.5.4 Illustrative W3C VC instance (Legal Person Attestation, JWT-secured)

```xml
<pte:LegalPersonAttestation>
  <pte:Format>w3c-vc</pte:Format>
  <pte:MediaType>application/vc+jwt</pte:MediaType>
  <pte:IssuerTrustListRef>https://ec.europa.eu/eubw/tl/BE-001</pte:IssuerTrustListRef>
  <pte:BindingMetadata>
    <pte:EUID>BEBE0123.456.789</pte:EUID>
    <pte:PeppolParticipantId scheme="9925">BE0123456789</pte:PeppolParticipantId>
    <pte:ValidFrom>2026-01-01T00:00:00Z</pte:ValidFrom>
    <pte:ValidUntil>2027-01-01T00:00:00Z</pte:ValidUntil>
  </pte:BindingMetadata>
  <pte:CredentialValue><![CDATA[
    eyJhbGciOiJFUzI1NiIsImtpZCI6IjEyMyIsInR5cCI6InZjK2p3dCJ9.
    eyJAY29udGV4dCI6WyJodHRwczovL3d3dy53My5vcmcvbnMvY3JlZGVudGlhbHMvdjIiXSwK
    InR5cGUiOlsiVmVyaWZpYWJsZUNyZWRlbnRpYWwiLCJFVUJXTGVnYWxQZXJzb25BdHRlc3Rh
    dGlvbiJdLAoiaXNzdWVyIjoiaHR0cHM6Ly9lYy5ldXJvcGEuZXUvZXVidy90bC9CRS0wMDEi
    LAoidmFsaWRGcm9tIjoiMjAyNi0wMS0wMVQwMDowMDowMFoiLAoidmFsaWRVbnRpbCI6IjIw
    MjctMDEtMDFUMDA6MDA6MDBaIiwKImNyZWRlbnRpYWxTdWJqZWN0Ijp7ImV1aWQiOiJCRUJF
    MDEyMy40NTYuNzg5IiwibGVnYWxOYW1lIjoiRXhhbXBsZSBUcmFkaW5nIE5WIiwidmF0SWRl
    bnRpZmllciI6IkJFMDEyMzQ1Njc4OSIsInBlcHBvbFBhcnRpY2lwYW50SWQiOiI5OTI1OkJF
    MDEyMzQ1Njc4OSJ9LAoiY3JlZGVudGlhbFN0YXR1cyI6eyJ0eXBlIjoiQml0c3RyaW5nU3Rh
    dHVzTGlzdEVudHJ5IiwiLi4uIjoiLi4uIn19.
    SIGNATURE_BYTES
  ]]></pte:CredentialValue>
</pte:LegalPersonAttestation>
```

The decoded JWT payload (the W3C VC proper):

```json
{
  "@context": ["https://www.w3.org/ns/credentials/v2"],
  "type": ["VerifiableCredential", "EUBWLegalPersonAttestation"],
  "issuer": "https://ec.europa.eu/eubw/tl/BE-001",
  "validFrom": "2026-01-01T00:00:00Z",
  "validUntil": "2027-01-01T00:00:00Z",
  "credentialSubject": {
    "euid": "BEBE0123.456.789",
    "legalName": "Example Trading NV",
    "vatIdentifier": "BE0123456789",
    "peppolParticipantId": "9925:BE0123456789"
  },
  "credentialStatus": {
    "type": "BitstringStatusListEntry",
    "statusPurpose": "revocation",
    "statusListIndex": "94567",
    "statusListCredential": "https://ec.europa.eu/eubw/status/BE-001/2026"
  }
}
```

When `Format = sd-jwt-vc` or `Format = mdoc`, the `CredentialValue` carries the respective format; verification follows IETF SD-JWT VC and ISO/IEC 18013-5 rules instead. The PTE schema is the same in all three cases.

#### 4.5.5 Compliance summary

| Question | Answer |
|---|---|
| Is the PTE container itself W3C-VC-compliant? | **No, by design.** The PTE is XML (forced by SBDH/UBL); W3C VCDM is a JSON / JSON-LD data model. The two are different layers. |
| Can a credential carried in the PTE be W3C-VC-compliant? | **Yes**, when `Format = w3c-vc` and the §4.5.3 normative profile is followed. |
| Will EUBW permit W3C VC for legal-person attestations? | **Not yet decided publicly** (no Implementing Act as of May 2026). EUDI's restriction of W3C VC to non-qualified EAAs applies to EUDI, not EUBW. The format-agnostic design protects against either outcome. |

---

### 4.6 Generating the XAdES qualified electronic seal

The `QualifiedSeal` element in the PTE carries a **XAdES qualified electronic seal** under eIDAS Article 35 (qualified electronic seals enjoy presumption of integrity and of correctness of the origin of the data to which they are linked). Standards basis:

- **ETSI EN 319 132-1 V1.3.1 (2024-07)** — XAdES digital signatures, Part 1: Building blocks and XAdES baseline signatures. Referenced by Commission Implementing Regulation (EU) 2024/2979.
- **ETSI EN 319 132-2** — Extended XAdES signatures (XAdES-E-XL, XAdES-E-A for long-term validation).
- **W3C XML Signature Syntax and Processing** (XMLDSig) — the underlying XML signature framework XAdES extends.
- **OASIS UBL Digital Signature Profiles 1.0** — defines the enveloped and detached profiles for signing UBL documents, including the XAdES profile (`urn:oasis:names:specification:ubl:dsig:enveloped:xades`).

#### 4.6.1 Packaging choice: enveloped XAdES inside UBL extensions

For the PTE, the recommended packaging is the **UBL enveloped XAdES profile**. Rationale:

| Option | Packaging | Pros | Cons |
|---|---|---|---|
| Enveloped (UBL DSig profile) | `<ds:Signature>` lives inside `<ext:UBLExtensions>/<sig:UBLDocumentSignatures>` | Single self-contained UBL document; OASIS-standardised; no separate file to lose; works for Peppol BIS today | Requires an XPath transform to exclude the signature itself from the digest |
| Detached | `<ds:Signature>` external to the invoice | Simple digest computation | Two artifacts to keep together; doesn't survive Peppol's single-document transport cleanly |
| ASiC-E container | ZIP container with invoice + detached signature | Strong for archival / long-term | Breaks Peppol's UBL-document-on-the-wire assumption |

The enveloped profile aligns with how UBL invoices are already signed today (Belgium B2B mandate, Malaysian MyInvois, Saudi ZATCA all use this pattern). The PTE merely standardises which `QualifyingProperties` and which trust anchors apply.

#### 4.6.2 Where the seal sits in the document

Per the container split (§3.5), the seal lives **inside the UBL document** in its own `<ext:UBLExtension>`, alongside (not inside) the STM extension. The seal covers the STM and the invoice body. It does **not** cover the TVE, which Peppol C3 inserts post-seal. The structural layout matches the example in §4.2:

```
<Invoice>
  <ext:UBLExtensions>
    <ext:UBLExtension URI="urn:peppol:trust:pte:1.0:stm">
      ← STM (legal-person attestation, mandate, supporting credentials, anchors)
         Written by Peppol C2 before sealing.
    </ext:UBLExtension>
    <ext:UBLExtension URI="urn:oasis:names:specification:ubl:dsig:enveloped:xades">
      ← <ds:Signature Id="PeppolTrustSeal"> — XAdES enveloped seal
         Written by Peppol C2 after building the STM.
    </ext:UBLExtension>
    <ext:UBLExtension URI="urn:peppol:trust:pte:1.0:tve">
      ← TVE (sender + receiver trust scores, EUBW C3 assertion)
         Inserted by Peppol C3 post-receipt; NOT covered by the seal.
    </ext:UBLExtension>
  </ext:UBLExtensions>
  ← invoice body (cac:AccountingSupplierParty, cac:InvoiceLine, etc.)
</Invoice>
```

The seal's `<ds:Reference>` covers (a) the invoice body and (b) the STM extension. It excludes both the UBL DSig extension (containing the seal itself) and the TVE extension (which doesn't exist at seal time but will be added by Peppol C3), via the XPath transform in §4.6.4 step 3. The TVE is anchored to the sealed bytes via `SealedDocumentDigest` rather than being signed by this seal.

#### 4.6.3 Where the seal is generated

Three actors, two physical locations for the key:

1. **The sealing entity** is the legal entity C1 — the qualified seal certificate is issued in C1's name (eIDAS Art. 38). This is independent of who *holds* the key.
2. **Key custody** is one of:
   - **(a) Local QSCD at C1** — rare in practice for invoice sealing; relevant only if C1 operates its own QSCD.
   - **(b) Remote QSCD at the QTSP** — the standard pattern. The QTSP holds the seal key in a SCAL2-architected QSCD; C2 (the Peppol service provider, authorised by the `IntermediationMandate`) triggers remote sealing via the QTSP's signing API (e.g. CSC API v2 / ETSI TS 119 432).
3. **The orchestrator** is C2: it canonicalises the document, computes the digest, calls the QTSP for the signature value, and assembles the `<ds:Signature>`. C2 never sees the seal key.

This split is what makes the `IntermediationMandate` essential — it is the EUBW-anchored proof that C2 was authorised by C1 to trigger remote sealing on C1's behalf.

#### 4.6.4 Generation procedure — step by step

**Inputs:**
- The full UBL invoice document `D` (with PTE already populated but **without** the `<ds:Signature>` element).
- The legal entity C1's qualified seal certificate `cert_C1` (X.509, eIDAS QcStatement extensions per ETSI EN 319 412-3 for legal persons).
- Access to the QTSP remote sealing endpoint (CSC API v2 / ETSI TS 119 432).
- The `IntermediationMandate` already obtained from C1's EUBW.

**Step 1 — Build the unsigned `<ds:Signature>` skeleton.**

```xml
<ds:Signature Id="PeppolTrustSeal" xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
  <ds:SignedInfo>
    <ds:CanonicalizationMethod Algorithm="http://www.w3.org/2006/12/xml-c14n11"/>
    <ds:SignatureMethod Algorithm="http://www.w3.org/2001/04/xmldsig-more#ecdsa-sha256"/>

    <!-- Reference 1: the invoice itself, with enveloped-signature + XPath transforms -->
    <ds:Reference Id="ref-invoice" URI="">
      <ds:Transforms>
        <ds:Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature"/>
        <ds:Transform Algorithm="http://www.w3.org/TR/1999/REC-xpath-19991116">
          <ds:XPath xmlns:ext="urn:oasis:names:specification:ubl:schema:xsd:CommonExtensionComponents-2">
            not(ancestor-or-self::ext:UBLExtension[
              ext:ExtensionURI='urn:oasis:names:specification:ubl:dsig:enveloped:xades'
              or ext:ExtensionURI='urn:peppol:trust:pte:1.0:tve'])
          </ds:XPath>
        </ds:Transform>
        <ds:Transform Algorithm="http://www.w3.org/2006/12/xml-c14n11"/>
      </ds:Transforms>
      <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
      <ds:DigestValue><!-- filled in step 3 --></ds:DigestValue>
    </ds:Reference>

    <!-- Reference 2: the XAdES SignedProperties -->
    <ds:Reference Id="ref-signed-props"
                  Type="http://uri.etsi.org/01903#SignedProperties"
                  URI="#xades-signed-props">
      <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
      <ds:DigestValue><!-- filled in step 4 --></ds:DigestValue>
    </ds:Reference>
  </ds:SignedInfo>

  <ds:SignatureValue><!-- filled in step 6 --></ds:SignatureValue>

  <ds:KeyInfo>
    <ds:X509Data>
      <ds:X509Certificate><!-- base64 of cert_C1 --></ds:X509Certificate>
    </ds:X509Data>
  </ds:KeyInfo>

  <ds:Object>
    <xades:QualifyingProperties Target="#PeppolTrustSeal"
        xmlns:xades="http://uri.etsi.org/01903/v1.3.2#">
      <xades:SignedProperties Id="xades-signed-props">
        <xades:SignedSignatureProperties>
          <xades:SigningTime><!-- ISO-8601, set at signing --></xades:SigningTime>
          <xades:SigningCertificateV2>
            <xades:Cert>
              <xades:CertDigest>
                <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
                <ds:DigestValue><!-- SHA-256 of cert_C1 (DER) --></ds:DigestValue>
              </xades:CertDigest>
            </xades:Cert>
          </xades:SigningCertificateV2>
          <xades:SignaturePolicyIdentifier>
            <xades:SignaturePolicyImplied/>
          </xades:SignaturePolicyIdentifier>
        </xades:SignedSignatureProperties>
        <xades:SignedDataObjectProperties>
          <xades:CommitmentTypeIndication>
            <xades:CommitmentTypeId>
              <xades:Identifier>
                http://uri.etsi.org/01903/v1.2.2#ProofOfOrigin
              </xades:Identifier>
            </xades:CommitmentTypeId>
            <xades:AllSignedDataObjects/>
          </xades:CommitmentTypeIndication>
        </xades:SignedDataObjectProperties>
      </xades:SignedProperties>
    </xades:QualifyingProperties>
  </ds:Object>
</ds:Signature>
```

The two key choices baked in here:

- **Canonicalisation = Exclusive XML Canonicalization 1.1** (`xml-c14n11`). This is the canonicalisation method most commonly required by national e-invoicing schemes built on UBL+XAdES (Saudi ZATCA, Malaysian MyInvois, Belgian e-invoicing reference). It is stable under namespace re-declarations that occur during transport.
- **Signature algorithm = ECDSA-SHA256** (`xmldsig-more#ecdsa-sha256`). On ENISA's agreed cryptographic mechanisms list. RSA-SHA256 is equally acceptable; the QTSP's seal certificate determines which is used.

**Step 2 — Insert C1's certificate into `<ds:X509Certificate>`** (base64-encoded DER). The certificate is fixed before digest computation because the XAdES `SigningCertificateV2` digest covers it.

**Step 3 — Compute the digest for Reference 1 (the invoice).**

1. Start from the full UBL document `D` including the (empty) `<ds:Signature>` skeleton.
2. Apply transforms in order:
   - `enveloped-signature` — removes the `<ds:Signature>` element from the node-set so the signature does not cover itself.
   - The XPath transform — additionally excludes `<ext:UBLExtension>` elements whose `ExtensionURI` is either `urn:oasis:names:specification:ubl:dsig:enveloped:xades` (the seal itself) or `urn:peppol:trust:pte:1.0:tve` (the TVE, which Peppol C3 will insert post-seal). Excluding the TVE prospectively means the digest the verifier re-computes after C3 has added the TVE matches the digest computed at seal time. Without this XPath, Peppol C3's addition of the TVE would invalidate the seal.
   - `xml-c14n11` — canonicalise the resulting node-set.
3. Compute SHA-256 over the canonicalised bytes.
4. Base64-encode and place into `<ds:DigestValue>` of `ref-invoice`.

**Step 4 — Compute the digest for Reference 2 (the XAdES SignedProperties).**

1. Extract the `<xades:SignedProperties Id="xades-signed-props">` element subtree.
2. Canonicalise it (`xml-c14n11`).
3. SHA-256, base64, into `<ds:DigestValue>` of `ref-signed-props`.

This second reference is what makes the signature **XAdES** rather than plain XMLDSig — the signing time, certificate digest, and commitment type are part of the signed payload.

**Step 5 — Canonicalise `<ds:SignedInfo>` and produce the data-to-be-signed (DTBS).**

1. Take the now-complete `<ds:SignedInfo>` (both `DigestValue`s populated).
2. Apply the canonicalisation method declared in `<ds:CanonicalizationMethod>` (`xml-c14n11`).
3. The canonicalised bytes are the DTBS.
4. SHA-256 over DTBS produces the DTBS Representation (DTBSR) — what the QSCD actually signs.

**Step 6 — Invoke the QTSP's remote sealing endpoint.**

C2 calls the QTSP (CSC API v2 / ETSI TS 119 432). The call carries: (a) DTBSR, (b) the certificate identifier (C1's seal cert), (c) C2's authentication to the QTSP, and (d) the `IntermediationMandate` reference as the authorisation artifact proving C2 may trigger this seal on behalf of C1. The QTSP:

1. Verifies C2's authentication and the mandate against the EUBW Trusted List.
2. Activates C1's seal key inside the QSCD (SCAL2 — key never leaves the cryptographic boundary).
3. Computes the ECDSA signature over DTBSR.
4. Returns the signature value to C2.

C2 places the value (base64) into `<ds:SignatureValue>`.

**Step 7 — XAdES level augmentation (optional but normally required).**

The seal so far is **XAdES-B-B** (baseline). For long-term validity, augment to higher levels per ETSI EN 319 132-1:

| Level | Adds | Provides |
|---|---|---|
| **XAdES-B-B** | Basic signature + SignedProperties | Minimum eIDAS qualified seal |
| **XAdES-B-T** | `SignatureTimeStamp` from a qualified TSA | Proof signature existed before time T |
| **XAdES-B-LT** | `CertificateValues` + `RevocationValues` | Self-contained verification material (long-term verifiability) |
| **XAdES-B-LTA** | Archive timestamp(s) | Long-term archival (re-timestampable as crypto ages) |

For e-invoicing the practical target is **XAdES-B-LT** (sometimes B-LTA for fiscal archival). The TSA timestamp request is issued by C2 (or by the QTSP itself, integrated) after Step 6 and before serialisation.

**Step 8 — Compute the sealed-document digest for the TVE.**

After serialisation, compute SHA-256 of the canonicalised sealed UBL document and place it into the TVE's `<pte:SealedDocumentDigest>` (§4.3). This anchors all post-seal validation entries to the exact sealed bytes. C2 then populates `<pte:SenderValidation>` with the trust score (§4.8) and seals the assertion separately if `AssertionSeal` is used.

#### 4.6.5 Seal verification at receipt (by EUBW C3)

EUBW C3 reverses the procedure:

1. Locate `<ds:Signature Id="PeppolTrustSeal">` in the UBL document.
2. Re-compute the digest of Reference 1 (apply the same transforms in order; verify they match the declared ones — including the XPath that excludes both the seal extension and the TVE extension).
3. Re-compute the digest of Reference 2.
4. Canonicalise `<ds:SignedInfo>` and verify the ECDSA signature using the public key from `<ds:X509Certificate>`.
5. Validate the certificate chain to a trust anchor on the **EU Trusted Lists for QTSPs (eIDAS TL)** — note this is the eIDAS TL for the seal certificate, separate from the **EUBW Trusted List of Business Wallet Providers** used to verify the legal-person attestation and mandate. Both must be checked. (See §6 item 1 — this two-PKI reconciliation is the hard part.)
6. For XAdES-B-T and above, verify the timestamp(s) against the qualified TSA's certificate chain.
7. Check `xades:SigningCertificateV2/CertDigest` matches the certificate presented in `<ds:KeyInfo>`.
8. Confirm `xades:CommitmentTypeId` is `ProofOfOrigin` (or another type acceptable to the verifier's policy).

A pass on all eight points yields **qualified electronic seal** status under eIDAS Art. 35 — invoice integrity and origin are legally presumed. The pass result feeds the `seal-validity` dimension in the trust score (§4.8) that EUBW C3 returns to Peppol C3.

---

### 4.7 Resolvable credentials — the sender's EUBW as the authoritative store

A core property of linking the PTE to the EUBW is that **credentials do not need to live inside the invoice**. The sender's EUBW is the authoritative store; the STM references credentials by URI, and the receiver resolves them at verification time.

#### 4.7.1 Two resolution modes

Every credential-bearing element in the STM uses the same `Resolution` wrapper with one of two modes:

| Mode | Element shape | Used when |
|---|---|---|
| **byReference** (default) | `<Resolution mode="byReference"><CredentialURI>https://eubw.{sender}/vc/{id}</CredentialURI><CredentialDigest>...</CredentialDigest></Resolution>` | Sender's EUBW reachable from receiver; live revocation desired; smaller invoice payload |
| **inline** | `<Resolution mode="inline"><CredentialValue>{credential bytes}</CredentialValue></Resolution>` | Offline / air-gapped receivers; archive scenarios; receiver's policy forbids outbound lookups |

`CredentialDigest` is **mandatory in both modes**. In `byReference` it lets the receiver detect a swap between seal time and resolution time (the digest is sealed, the resolved credential must match). In `inline` it lets a downstream archive verify that the bytes were not altered after the seal.

#### 4.7.2 Resolution and freshness — EUBW C3 calls EUBW C2

Credential resolution happens at the **EUBW** layer, not the Peppol layer. Peppol C3 hands the STM to EUBW C3 (§4.9); EUBW C3 then performs the steps below by calling **EUBW C2** (the sender's wallet endpoint).

For `byReference`:

1. **Resolve.** EUBW C3 calls EUBW C2 at `STM/Sender/EUBWEndpoint`, GET `CredentialURI`. Authentication is via the EUBW Trusted List of Business Wallet Providers (no bilateral pre-arrangement; both EUBWs authenticate via their TL entries). Caching is keyed by URI + digest.
2. **Digest-match.** SHA-256 of the resolved bytes must equal `CredentialDigest`. Mismatch ⇒ score this credential as failed; do not use it for content validation.
3. **Verify.** EUBW C3 runs the format-appropriate verification (W3C VC: VC-JOSE-COSE or Data Integrity proof; SD-JWT VC: SD-JWT verification; mdoc: ISO/IEC 18013-5 verification).
4. **Status-check (live).** EUBW C3 resolves the credential's status mechanism — for W3C VC, the Bitstring Status List entry. This is the live revocation check the sender's EUBW C2 makes possible: a credential valid at seal time may be revoked by verification time.
5. **Trust-anchor check.** Issuer resolves to an entry in the EUBW Trusted List declared in `TrustAnchorPointers`.

If EUBW C2 is unreachable, EUBW C3 falls back to inline credentials if present, otherwise scores the affected dimension as `unverified` (not `failed` — see §4.8). Network-unreachable is a different signal from cryptographically-failed.

#### 4.7.3 Why this matters

The reference model means the EUBW does the heavy lifting it was designed for: credential lifecycle, revocation, freshness, holder binding. The PTE just **points** to the right wallet entries. A revoked IBAN credential (e.g. the seller's bank account is no longer valid) shows up at the receiver in real time, *without* the seller having to re-issue the invoice or the seal becoming invalid — the seal remains valid for the bytes it covers, but the receiver's trust score for the `iban-consistency` dimension drops.

This separates two things that today are tangled: **document integrity** (seal, immutable) from **assertion currency** (credentials, time-varying).

---

### 4.8 Trust Score — weighted, dimensional, non-boolean

Validation produces a **numeric trust score**, not a pass/fail boolean. The score is per-actor (sender and receiver each produce their own) and is decomposed into weighted dimensions so receivers can apply policy rather than reject on a single failure.

#### 4.8.1 Dimensions

Eight default dimensions, each scored 0–100:

| Dimension | Question it answers | Inputs |
|---|---|---|
| `seal-validity` | Is the XAdES qualified seal cryptographically valid and chained to an eIDAS-qualified CA? | XAdES verification (§4.6.5); eIDAS TL |
| `legal-person-identity` | Is the sender's `LegalPersonAttestation` valid, current, and matched to the participant ID? | EUBW TL; credential status; binding check |
| `intermediation-authority` | Did C1 authorize C2 (the AP that sealed) — mandate valid, in scope, not revoked? | `IntermediationMandate` resolution + verification |
| `vat-consistency` | Do VAT credentials match the VAT numbers in the invoice body, per jurisdiction? | `VATCredential` resolution + cross-check vs. UBL fields |
| `iban-consistency` | Do IBAN credentials cover every IBAN in the invoice, and are the holders consistent with the sender? | `IBANCredential` resolution + cross-check vs. `cac:PaymentMeans` |
| `lpid-freshness` | Is the LPID attestation present and recent enough per receiver policy? | `LPIDCredential` issuance date vs. policy threshold |
| `format-compliance` | Does the invoice pass EN 16931 / Peppol BIS schematron? | Standard Peppol validation |
| `supplier-approval` | Where required: does the supplier hold a valid approved-supplier credential? | `ApprovedSupplierCredential` resolution + receiver's approval list |

The list is extensible — additional dimensions can be added by profile (e.g. `eb2b-mandate-coverage` for specific national CTC mandates) without breaking existing consumers.

#### 4.8.2 Per-dimension score values

| Score | Meaning |
|---|---|
| 100 | Verified — cryptographically valid, not revoked, matches invoice content |
| 80–99 | Verified with minor caveats — e.g. inline rather than EUBW-resolved, slightly aged but within policy |
| 60–79 | Partial — some checks pass, some unverifiable (e.g. EUBW unreachable) |
| 1–59 | Inconsistent — credential resolves and verifies, but does not match invoice content |
| 0 | Failed — cryptographic verification failed, or credential revoked |
| `unverified` (no numeric) | Not checked — credential not provided, or check not applicable |

`unverified` is distinct from 0. A missing optional credential is not a failure; it just removes that dimension's contribution from the score.

#### 4.8.3 Aggregate score

The aggregate `TrustScore` is a weighted mean of provided dimensions:

```
TrustScore = round( Σ (w_i × s_i) / Σ w_i )    over dimensions i where s_i ≠ unverified
```

Default weights — published as scoring policy `urn:peppol:trust:scoring:v1`:

| Dimension | Default weight | Notes |
|---|---|---|
| `seal-validity` | 3 (hard requirement) | Always a hard requirement |
| `legal-person-identity` | 3 (hard requirement) | Always a hard requirement |
| `intermediation-authority` | 2 | |
| `vat-consistency` | 2 | |
| `iban-consistency` | 2 | |
| `format-compliance` | 2 | |
| `lpid-freshness` | 1 | |
| `supplier-approval` | **receiver-configurable: 0 (not required), 1 (soft), or 3 (hard requirement)** | Default 1 when present in receiver policy, 0 otherwise. Receivers operating strict supplier-approval programmes set this to 3, making absence or failure a hard decline. See §3.3.1. |

**Hard requirements:** any dimension with weight 3 scoring 0 forces the aggregate to 0 regardless of other scores. A revoked legal-person attestation, a failed seal, or — when configured as a hard requirement — a missing/invalid approved-supplier credential cannot be averaged away. This is the "boolean inside the score" needed for legal soundness and for strict commercial policies.

#### 4.8.4 Receiver policy and decision

The receiver maps the score to an action via a configurable policy:

| Score | Default action |
|---|---|
| 0 (hard-requirement failure) | **Hard decline** — booking blocked regardless of other dimensions |
| ≥ 90 | Accept and book |
| 75 – 89 | Accept with review (flag for AP team) |
| 60 – 74 | Hold pending sender clarification |
| 1 – 59 | Reject |

Receivers publish their thresholds and weighting policy via their SMP or via a separate trust-policy endpoint, so senders can predict acceptance before submitting. This is especially relevant for B2B onboarding workflows where the receiver may require `supplier-approval = hard requirement` — senders that aren't pre-approved learn this before submitting, not after their invoice is hard-declined.

#### 4.8.5 Why a score, not a boolean

Operationally the score model handles the realistic mixed-state cases that a boolean cannot:

- IBAN credential exists and is valid, but the holder name on the credential differs by punctuation from the invoice — not a fraud signal, not a clean pass; deserves a 90, not a reject.
- EUBW is reachable, mandate verifies, but the LPID is 14 months old when receiver policy prefers 12 months — score the freshness dimension at 85, leave the aggregate above accept threshold, log the finding.
- All cryptographic checks pass, but the supplier is not on the receiver's approved list — supplier-approval dimension = 0, but if it has weight 1 and other dimensions are 100, aggregate sits around 90; receiver policy decides.

A boolean validator would reject all three. A scoring validator gives the receiver's AP team a single number to act on plus dimensional detail to investigate when needed.

---

### 4.9 The verification call — APEIP `/verify`

The TVE is always carried inside the document (binding-specific carrier). The question this section answers is **how the message handler obtains the trust score it writes into the TVE**. The answer: by calling its companion EUBW provider via the **AP ↔ EUBW Integration Protocol (APEIP)**, specifically the `/verify` endpoint.

**This protocol is specified in a separate document** to keep it transport-independent and to allow it to be standardised on its own track (EUBW Implementing Acts). See:

> **[../protocol/APEIP-Specification.md](../protocol/APEIP-Specification.md)** — the complete AP ↔ EUBW Integration Protocol, including:
> - Outbound endpoints (`/credentials/for-document`, `/seal`, `/presentation-request`) used by message handlers at send time.
> - Inbound endpoint (`/verify`) used by message handlers at receive time.
> - Authentication modes (same-provider, bilateral, TL-anchored).
> - Error model, versioning, transport-independence.

**Why this is an internal protocol, not an external endpoint:**

1. **C4 already gets the result inline in the document.** The TVE rides in the binding's PTE carrier, parsed by C4's ERP. There is nothing for C4 to fetch.
2. **C4 doesn't speak EUBW protocols.** Forcing C4's ERP to integrate with the wallet ecosystem would defeat the architecture; the message handler ↔ EUBW integration absorbs that complexity.
3. **The verification authority is EUBW C3, not the handler.** The handler surfaces what EUBW C3 says; it has no independent trust authority over EUBW credentials. A C4-facing endpoint would imply the handler *is* the verifier, which it is not.

**The integration boundary stops at the message handlers and their EUBW providers:**

| Actor | Change required for the PTE |
|---|---|
| C1 (sender legal entity) | **None.** ERP sends the same document it sends today. |
| C4 (receiver legal entity) | **None for transport.** ERP receives the same document type with one additional carrier element (TVE) to parse. Surfacing the score in the booking UI is the only ERP-side change. |
| Sender handler (Peppol C2, mail server, QERDS provider, REST sender) | **Integrate with sender EUBW** via APEIP (outbound endpoints). Build the STM in the binding's carrier. Apply seal. |
| Receiver handler (Peppol C3, mail receiver, QERDS receiver, REST receiver) | **Integrate with receiver EUBW** via APEIP `/verify`. Insert the TVE into the document before delivery. |
| Sender EUBW | **Expose APEIP outbound endpoints** to its authorised message handlers. |
| Receiver EUBW | **Expose APEIP `/verify`** to its authorised message handlers. Talks to sender EUBW via the EUBW Trusted List. |

The clean property: **C1 and C4 are insulated from the EUBW ecosystem entirely**. Trust services are absorbed by the message-handler layer, and the handler ↔ EUBW protocol is specified in APEIP.

## 5. Standardization path

Standardizable — the constituent pieces are already standards; the work is *profiling their combination*.

| Body | Artifact needed | Builds on |
|---|---|---|
| OpenPeppol | **Peppol Trust Profile**: UBL extension schemas for STM and TVE + Peppol BIS Billing 3.0 schematron rules + SMP capability identifier + AP processing rules. Primary track, fastest to pilot. | Existing Peppol AS4 / SMP / BIS Billing 3.0 specs |
| CEN/TC 434 | **EN 16931 Extension** for trust attestation under TR 16931-5, so XRechnung, CIUS-PT, and future EN 16931-conformant profiles can adopt the PTE without going through Peppol governance. Includes both UBL and CII syntax bindings. | EN 16931-1:2025 (approved 13 Feb 2026 — favourable timing); TR 16931-5 extension guidelines |
| ETSI | Profile binding EUBW legal-person attestations + mandates to the XAdES seal (UBL) and PAdES seal (PDF/A-3 hybrid CII formats like ZUGFeRD / Factur-X). | EN 319 132/182 (XAdES/JAdES); EN 319 142 (PAdES); EN 319 411-2 |
| European Commission / EUBW Implementing Acts | Trusted List format; Peppol C3 ↔ EUBW C3 verification protocol (§4.9); EUBW C3 ↔ EUBW C2 presentation protocol (§3.3.1); EUID binding rules; directory API | EUBW draft: Commission Trusted List + EU directory |

**Why plausible:** the EUBW draft names ViDA interoperability as an explicit objective, and ViDA's reporting regime is the policy forcing-function. Peppol is the de-facto rail for that reporting in most member states.

---

## 6. Hard parts (open issues)

1. **Two PKIs, one envelope.** Peppol PKI (transport) and EUBW/QTSP trust lists (content) are governed by different authorities with different revocation, lifecycle, and liability regimes. Reconciling them — especially revocation timing and cross-recognition — is the hardest engineering and legal problem, not the XML schema.
2. **Performance.** Peppol's selling point is sub-second exchange. Per-message verification of attestations, mandates, seals, and registry data against external Trusted Lists at C2 and C3 adds latency and external dependencies. Requires aggressive caching of trust lists and OCSP/CRL, and likely a verify-async / flag-on-failure mode for high volume.
3. **Adoption asymmetry.** The EUBW draft makes acceptance mandatory for authorities but voluntary for companies. So a B2B Peppol extension cannot be mandated on the corner-1 side via the EUBW regulation alone — adoption is driven by national CTC mandates (Belgium 2028, France, Poland), implying mixed adoption for years.
4. **No existing OpenPeppol work item.** The 5-corner CTC work and the EUBW work are advancing on separate tracks; someone has to convene them.
5. **CII syntax binding.** EN 16931 has two syntax bindings (UBL and CII); the PTE is currently UBL-only. CII coverage matters for Franco-German trade corridors (ZUGFeRD, Factur-X, XRechnung-CII) and for the French B2B mandate (which accepts both syntaxes). A CII binding would require a parallel PTE schema using UN/CEFACT CII's extension mechanism rather than `<ext:UBLExtensions>`, but the abstract data model (STM, TVE, supporting credentials) and the XAdES seal mechanism transfer directly. Hybrid PDF formats (ZUGFeRD, Factur-X) add a second complication: the seal must cover the PDF/A-3 container and the embedded CII XML consistently — eIDAS PAdES (PDF Advanced Electronic Signatures, ETSI EN 319 142) applies rather than XAdES.
6. **Portability across CIUS profiles.** XRechnung, CIUS-PT, Singapore BIS, future French PPF profiles each tighten EN 16931 differently. The STM and TVE structures don't conflict with any of these (they live in extension points the EN explicitly reserves for foreign content), but per-profile schematron rules must be written for each to avoid false validation failures. The right governance home for the *cross-profile* version is CEN/TC 434 under TR 16931-5 (see §2.5), not OpenPeppol.
7. **Document types beyond invoices.** The PTE is invoice-focused. Credit notes share the trust requirements with trivial schema reuse. Orders, despatch advices, and other Peppol BIS document types need a different supporting-credential set (no IBAN on an order, no VAT on a despatch advice) — straightforward to profile but not yet done.

---

## 7. One-line synthesis

A portable, end-to-end **trust envelope (PTE)** that rides inside the Peppol business-document layer, sourcing legal-entity identity, intermediation/sealing authority, qualified seals, and registry-backed data validation from the EUBW/QTSP trust ecosystem — leaving the AS4/Peppol-PKI transport untouched. Standardizable because every component already exists as a standard; the unbuilt part is the profile binding them plus cross-PKI trust reconciliation. The forcing function is ViDA/CTC, not the EUBW regulation itself.
