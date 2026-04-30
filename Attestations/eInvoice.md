# eInvoice Attestation

**Version:** V0.5 – final draft  
**Date:** 16-03-2026  
**Lead author:** Maarten Boender

## Attestation introduction

The eInvoice Attestation is a verifiable representation of an invoice exchange event, binding invoice content (via a payload hash) to a cryptographic signature associated with the Supplier's legal-person identity. It is used by a Buyer to verify invoice integrity, Supplier identity validity, and (where applicable) the authenticity and validity of referenced evidence.

## Attribute specification

The attestation attributes are defined in the tables below:

- The first column specifies the identifiers of the attestation attributes. The attribute identifiers in this column SHALL be used in requests and responses. There SHALL be at most one attribute with the same attribute identifier in each attestation.
- The second column describes the meaning of the attribute.
- The third column specifies whether the presence of the attribute in an attestation is mandatory (M) or optional (O).
- The fourth column indicates how the data elements SHALL be encoded, using the CDDL representation types defined in [RFC 8610].

> **NOTE:** If an attribute is indicated as mandatory, this solely means that the Issuer SHALL ensure that this element is present in the attestation. It does not imply that a Relying Party is required to request such an attribute when interacting with the Wallet Instance. Neither does it imply that the User cannot refuse to release a mandatory attribute if requested.

## Model

The eInvoice Attestation is modelled as one object. It binds to an external structured invoice payload (e.g., EN 16931 / Peppol BIS) via `invoicePayloadHash`. It may include one or more optional `evidenceReferences` to integrity-bind supporting evidence (e.g., purchase order, delivery note).

## Attributes

### eInvoice Attestation object

*Attribute names use EN 16931 semantic term names in camelCase style.*

| **Attribute** | **Definition** | **M/O** | **Encoding format** |
|---|---|---|---|
| `invoicePayloadHash` | Hash of the canonicalized invoice payload as defined by the rulebook. | M | tstr |
| `invoiceFormat` | Identifier of the invoice representation/profile used for the payload. | O | tstr |
| `invoiceNumber` | Invoice identifier unique for the Supplier within the agreed duplicate-detection window. | M | tstr |
| `issueDate` | Invoice issue date (ISO 8601 date). | M | tstr |
| `sellerLegalRegistrationIdentifier` | Reference to Supplier Legal Person identity (e.g., EUCC/LPID identifier reference). | M | tstr |
| `sellerRegistrationName` | The legal name of the supplier. | M | tstr |
| `sellerVatIdentifier` | The tax registration ID of the Supplier. | M | tstr |
| `sellerCountry` | The country code of the supplier. | M | tstr |
| `buyerLegalRegistrationIdentifier` | Reference to Buyer Legal Person identity (e.g., EUCC/LPID identifier reference). | O | tstr |
| `buyerVatIdentifier` | The tax registration ID of the Buyer. | O | tstr |
| `buyerCountry` | The country code of the buyer (ISO 3166). | O | tstr |
| `amountDueForPayment` | Total payable amount as per the chosen binding rules (EN 16931 / BIS). | M | tstr |
| `paymentInstructions` | The group for all payment-related information. | O | container |
| `taxSubtotal` | VAT Breakdown group (one or more entries per tax code). | O | container |
| `invoiceTotalVatAmount` | The total tax amount for this invoice. | M | tstr |
| `InvoiceCurrencyCode` | Currency code (ISO 4217). | M | tstr |
| `BuyerElectronicAddress` | Address/endpoint identifier for delivery/receipt handling (definition TBD). | O | tstr |
| `evidenceReferences` | Optional references to supporting evidence items. | O | array |
| `evidenceReferencesType` | The type of evidence. | O | tstr |
| `status` | Lifecycle status for corrections/cancellations/credit notes (value list TBD). | O | tstr |

### `taxSubtotal` container (optional)

| **Attribute** | **Definition** | **M/O** | **Encoding format** |
|---|---|---|---|
| `taxCategoryCode` | VAT category code. | M | tstr |
| `taxableAmount` | Sum of net amounts for this VAT category/rate. | M | tstr |
| `taxPercent` | The VAT percentage. | M | tstr |
| `taxAmount` | Total VAT calculated for this specific subtotal. | M | tstr |
| `taxExemptionReasonCode` | A standardized code for the exemption. | O | tstr |

### `paymentInstructions` container (optional)

| **Attribute** | **Definition** | **M/O** | **Encoding format** |
|---|---|---|---|
| `paymentMeansCode` | A code representing the payment method (e.g., 30 for Credit Transfer). | M | tstr |
| `paymentMeansText` | A text description of the payment method, e.g. IBAN, VISA, MC, etc. | O | tstr |
| `paymentAccountIdentifier` | Payment account identifier used for IBAN or other bank account numbers. | M | tstr |
| `paymentCardPAN` | Payment card primary account number (masked or full credit/debit card PAN). | O | tstr |

### Code lists

**`invoiceFormat`**

EN 16931 is assumed as default but can be country-specific (ZUGFeRD, INSBOUW, etc.).

**`paymentMeansCode`**

Based on the UNTDED 4461 (UNCL 4461) code list:

| Code | Description |
|---|---|
| 30 | Credit transfer — a payment made by bank transfer from buyer to seller |
| 48 | Bank card — payment via debit or credit card |
| 49 | Direct debit — amount directly debited from the customer's bank account |
| 58 | SEPA credit transfer — a credit transfer compliant with SEPA standards |

**`evidenceReferences`**

Optional references to supporting evidence items. Supported forms:

- PDF as BASE64
- UBL as BASE64
- URI identifier + hash (URI MUST be publicly accessible — no standard authorization mechanism exists)

**`evidenceReferencesType`**

| Value |
|---|
| `purchase_order` |
| `delivery_note` |
| `contract` |
| `acceptance_evidence` |
| `other` |

**`status`**

| Value |
|---|
| `active` |
| `corrected` |
| `cancelled` |
| `credited` |

## Integrity rules

- The attestation SHALL be valid only if the Supplier wallet instance is bound to a valid legal entity attestation (e.g., EUCC or equivalent), and the Buyer identity in the invoice matches the receiving wallet's legal entity identity.
- The attestation SHALL be valid only if the Supplier's signing key is trusted and valid at verification time (not expired/revoked/suspended) and the signature verifies.
- `invoicePayloadHash` SHALL equal the hash of the canonicalized invoice payload, using the rulebook-defined canonicalization and hash algorithm.
- `invoiceNumber` SHALL be present and unique for the Supplier within the verifier's duplicate-detection window; duplicates SHALL trigger reject or quarantine per verifier policy.
- `issueDate` SHALL be present and not unreasonably in the future (allow clock-skew tolerance). If `dueDate` is present (if included in a future profile), it SHALL be ≥ `issueDate`.
- `sellerLegalRegistrationIdentifier` and `buyerRegistrationIdentifier` SHALL match the identities in the invoice payload; mismatches invalidate the attestation.
- If `taxSubtotal` is used, the `totalTaxAmount` MUST strictly equal the sum of all `taxAmount` values provided in the `taxSubtotal`.
- Any referenced or embedded evidence SHALL be integrity-bound and, where applicable, valid at verification time (status/expiry/revocation).
- For corrections/cancellations/credit notes, the attestation SHALL reference the original invoice/attestation and use only rulebook-defined state transitions.

## Open issues

| # | Issue | Status | Responsible |
|---|---|---|---|
| 1 | Decide whether Approved Supplier Attestation (ASA) is optional for baseline conformance or a recommended policy hook. | closed | We will not use ASA |
| 2 | Decide invoice payload canonicalization method(s) and hash strategy. | open | WP4 Architecture WG |
| 3 | Decide invoice semantic (e.g. EN16931 semantic model, Peppol BIS 3.0 UBL syntax, or both with one as primary) or align to ViDA. | closed | EN 16931, but can be country dependent. WP4 Semantic WG to confirm. |
| 4 | Decide trust anchor mechanism for LPID and signing keys (registry vs federation vs QTSP signals). | open | Not in scope of SC5, but a dependency. Will be based on ETSI TS 119 602. WP4 Trust Registry Infrastructure WG to confirm. |
| 5 | Decide minimum disclosed invoice elements for validation vs privacy. | closed | Indicated as Mandatory or Optional in attribute table |
| 6 | Decide if we want an MVP+ and what it should cover. | closed | Will run as **MVP**, not MVP+ |

## Changelog

| Version | Date | Changes | Lead author |
|---|---|---|---|
| V0.1 | 2026-02-12 | First draft | Maarten Boender |
| V0.2 | 2026-03-02 | Second draft | Maarten Boender |
| V0.3 | 2026-03-04 | Third draft | Maarten Boender |
| V0.4 | 2026-03-06 | Final draft | Maarten Boender |
| V0.5 | 2026-03-16 | Payment instructions added | Maarten Boender |
