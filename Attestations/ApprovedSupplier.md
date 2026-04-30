# Approved Supplier Attestation

**Version:** Draft  
**Date:** 29-04-2026  
**Lead author:** Rune Kjørlaug

## Attestation introduction

The Approved Supplier Attestation proves the existence of a trusted and acknowledged contractual relationship between two legal entities whose identities are established through separate legal entity attestations (e.g. EUCC). The attestation enables a buyer to establish a controlled trust perimeter for receiving electronic invoices, allowing only suppliers explicitly acknowledged by the buyer to send eInvoices via service providers.

The attestation does not itself identify the buyer or supplier. The parties are implicitly determined by the Wallet Instances involved, each bound to a verified legal entity through EUCC or equivalent attestations.

The attestation is inspired by the contract award notice model used in public procurement (eForms) and may be issued as:

- a Qualified Electronic Attestation of Attributes (QEAA / PubEAA) based on an authoritative public source (e.g. TED), or
- an Electronic Attestation of Attributes (EAA) issued via a Business Wallet based on contractual evidence or buyer acknowledgement.

The attestation asserts the existence, scope, and validity of a relationship only and does not cover invoice content, performance, or financial data.

## Attribute specification

The attestation attributes are defined in the tables below:

- The first column specifies the identifiers of the attestation attributes. The attribute identifiers in this column SHALL be used in requests and responses. There SHALL be at most one attribute with the same attribute identifier in each attestation.
- The second column describes the meaning of the attribute.
- The third column specifies whether the presence of the attribute in an attestation is mandatory (M) or optional (O).
- The fourth column indicates how the data elements SHALL be encoded, using the CDDL representation types defined in [RFC 8610].

> **NOTE:** If an attribute is indicated as mandatory, this solely means that the Issuer SHALL ensure that this element is present in the attestation. It does not imply that a Relying Party is required to request such an attribute when interacting with the Wallet Instance. Neither does it imply that the User cannot refuse to release a mandatory attribute if requested.

## Model

The Approved Supplier Attestation consists of a **relationship object** with a **structured scope**, allowing network-specific constraints to be expressed without hard-coding them into the core relationship.

```
Buyer–Supplier Relationship
 ├─ Relationship attributes
 ├─ Relationship scope
 │   └─ Peppol scope (optional)
 └─ Source / Evidence reference
```

The legal identity of the parties is established through separate attestations (e.g. EUCC) and is out of scope for this attestation.

## Attributes

### Buyer–Supplier Relationship object

| **Attribute** | **Definition** | **M/O** | **Encoding format** |
|---|---|---|---|
| `relationship_id` | Unique identifier of the acknowledged relationship. | M | tstr |
| `relationship_type` | Type of acknowledged relationship between the wallet holders. | M | tstr |
| `relationship_status` | Current status of the relationship. | M | tstr |
| `relationship_start_date` | Start date of the acknowledged relationship. | M | tdate |
| `relationship_end_date` | End date of the relationship, if applicable. | O | tdate |
| `authoritative_source` | Reference to the authoritative source or evidence supporting the relationship. | O | Source reference object |

### Source reference object

| **Attribute** | **Definition** | **M/O** | **Encoding format** |
|---|---|---|---|
| `source_type` | Type of authoritative source used as evidence. | M | tstr |
| `source_identifier` | Identifier of the source record (e.g. contract notice ID). | M | tstr |
| `source_system` | System in which the source is published. | O | tstr |
| `publication_date` | Date on which the source was published. | O | tdate |
| `source_uri` | URI pointing to the source record, if available. | O | tstr |

### Code lists

**`relationship_type`**

| Value | Description |
|---|---|
| `contractual` | |
| `framework_agreement` | |
| `public_contract_award` | |
| `private_contract` | |
| `buyer_acknowledgement` | |

**`relationship_scope`**

| Value | Description |
|---|---|
| `einvoicing` | |
| `einvoicing_peppol` | |
| `procurement` | |
| `multiple` | |

**`source_type`**

| Value | Description |
|---|---|
| `ted_contract_award_notice` | |
| `national_procurement_notice` | |
| `private_contract_reference` | |
| `buyer_assertion` | |

## Integrity rules

- If `relationship_end_date` is provided, it SHALL be later than `relationship_start_date`.
- The attestation SHALL be considered valid only when both Wallet Instances involved are bound to valid legal entity attestations (e.g. EUCC).
- If `source_type` refers to a public authoritative source, `source_identifier` SHALL be present.
- For use in eInvoicing trust decisions, `relationship_status` SHALL be `active`.
