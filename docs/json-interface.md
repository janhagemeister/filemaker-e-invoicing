# JSON interface

The generator posts invoice JSON to

```text
POST <API-BASE-URL>/<JSON-TO-XML-ENDPOINT>/<MODEL-NAME>
```

The schema endpoint is authoritative for element names. Do not hard-code the element list in the FileMaker file.

```text
GET <API-BASE-URL>/<SCHEMA-ENDPOINT>/<MODEL-NAME>
Accept: application/json
x-api-key: <API-KEY>
```

## Schema response

The root is a `JSONArray`. Each entry is a four-element array:

| Position | Content | Example FileMaker target |
|---|---|---|
| 1 | Full XML path | `Schema::Xml_Pfad` |
| 2 | JSON path to use in the request | `Schema::Json_Pfad` |
| 3 | Expected data type | `Schema::Feldtyp` |
| 4 | Associated XML element | `Schema::Xml_Element` |

An error object must not be imported as a schema list. Presence of an element in the schema does not mean it is allowed, required, or sufficient for the business case.

## JSON types in FileMaker

| Value | FileMaker constant |
|---|---|
| Text, codes, IDs, date strings | `JSONString` |
| Quantities, prices, percentages, totals | `JSONNumber` |
| True / false | `JSONBoolean` |
| Object or array | `JSONObject` / `JSONArray` |

Formatted number strings such as `"1.234,56"` must not be sent as invoice amounts.

## Header paths (excerpt)

These paths appear in [E-Rechnung in FileMaker erstellen](https://www.hagemeister-conception.de/blog/e-rechnung-in-filemaker-erstellen). They are not a complete invoice.

| Path | Meaning |
|---|---|
| `exchanged_document_context.guideline_specified_document_context_parameter.id.value` | Profile / guideline identifier |
| `exchanged_document.id.value` | Invoice number |
| `exchanged_document.type_code.value` | Document type code, `BT-3` (`380` = commercial invoice) — see [Credit notes and document type codes](#credit-notes-and-document-type-codes) |
| `exchanged_document.issue_date_time.date_time_string.format` | Date format code (`102` = YYYYMMDD) |
| `exchanged_document.issue_date_time.date_time_string.value` | Issue date |
| `supply_chain_trade_transaction.applicable_header_trade_settlement.invoice_currency_code.value` | Invoice currency |

Required fields depend on document type, profile, and business case. Seller, buyer, tax breakdown, totals, and payment data must be added according to the retrieved schema.

## Line-item paths (excerpt)

Loop over invoice lines with a zero-based index `$i`:

```text
supply_chain_trade_transaction.included_supply_chain_trade_line_item[$i]
```

| Relative path | Meaning |
|---|---|
| `associated_document_line_document.line_id.value` | Line number |
| `specified_trade_product.name.value` | Item name |
| `specified_line_trade_delivery.billed_quantity.value` | Quantity |
| `specified_line_trade_delivery.billed_quantity.unit_code` | Unit of measure code |
| `specified_line_trade_settlement.applicable_trade_tax.rate_applicable_percent.value` | VAT rate |
| `specified_line_trade_settlement.specified_trade_settlement_line_monetary_summation.line_total_amount.value` | Line net amount |

See [../sample-data/invoice.json](../sample-data/invoice.json).

## Credit notes and document type codes

The document type is not determined by positive or negative amounts alone. EN 16931 uses a document type code (`BT-3`) to identify the business document.

Common document type codes include:

| Code | Document |
|---|---|
| `380` | Commercial invoice |
| `381` | Credit note |
| `384` | Corrected invoice |
| `389` | Self-billed invoice |

A credit note must therefore be generated explicitly as the appropriate document type. Do not create a normal invoice and merely negate the amounts.

### UBL credit notes use a separate schema

UBL uses different document structures for invoices and credit notes.

An invoice uses:

```xml
<Invoice xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2">
    <cbc:InvoiceTypeCode>380</cbc:InvoiceTypeCode>
    ...
    <cac:InvoiceLine>
        <cbc:InvoicedQuantity>...</cbc:InvoicedQuantity>
    </cac:InvoiceLine>
</Invoice>
```

A credit note uses the separate UBL CreditNote schema:

```xml
<CreditNote xmlns="urn:oasis:names:specification:ubl:schema:xsd:CreditNote-2">
    <cbc:CreditNoteTypeCode>381</cbc:CreditNoteTypeCode>
    ...
    <cac:CreditNoteLine>
        <cbc:CreditedQuantity>...</cbc:CreditedQuantity>
    </cac:CreditNoteLine>
</CreditNote>
```

Consequently, the JSON-to-XML model selected by the FileMaker solution must match the intended document type. Changing only the document type code is **not sufficient** when switching between a UBL invoice (`Ubl21`) and a UBL credit note (`Ubl21cn`).

CII-based formats such as ZUGFeRD / Factur-X use the same CII document structure for invoices and credit notes; the business document type is identified by the corresponding type code.

The schema endpoint should therefore always be queried for the exact model that is going to be generated.

## Models differ

The structure differs per model. `Facturxextended` is not `Facturxen16931` with extra fields — repeatable elements such as `applicable_trade_tax`, `defined_trade_contact`, and `specified_trade_payment_terms` behave differently there.
