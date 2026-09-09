# E-Invoicing with FileMaker

Technical reference and FileMaker examples for **EN 16931 electronic invoicing with Claris FileMaker**.

The examples follow the approach described in these articles:

- [E-Rechnung in FileMaker erstellen](https://www.hagemeister-conception.de/blog/e-rechnung-in-filemaker-erstellen)
- [E-Rechnung in FileMaker validieren und visualisieren](https://www.hagemeister-conception.de/blog/e-rechnung-in-filemaker-validieren-visualisieren)
- [FileMaker und E-Rechnung](https://www.hagemeister-conception.de/filemaker-und-erechnung)

This repository contains two FileMaker example files:

- **E-Invoice Generator** (`erechnungerstellung.fmp12`) — invoice JSON → XML → PDF/A-3 → validation.
- **E-Invoice Visualizer** (`erechnungvisualisierung.fmp12`) — validation and visualization of XRechnung and Factur-X / ZUGFeRD.

See [DISCLAIMER.md](DISCLAIMER.md).

## Scope

This repository covers **document generation and validation** — building the invoice XML, assembling the hybrid PDF/A-3, and checking the result against schema and business rules.

Out of scope, and not implemented by the example files:

- **Peppol transport.** Sending through the Peppol network requires an Access Point; Peppol BIS Billing 3.0 is not among the supported models below.
- **Archiving.** Retention, GoBD-compliant storage, and audit-proof archiving remain the responsibility of the FileMaker solution.
- **ERP or accounting export**, e.g. towards DATEV.
- **Legal and tax assessment.** Whether an invoice must be issued electronically at all — B2B, B2G, the German E-Rechnungspflicht — is a question for the operator, not for these examples.

For B2G invoices, note that the **Leitweg-ID** is carried in the buyer reference (`BT-10`) and must be supplied by the FileMaker solution as part of the invoice data. The examples do not derive it.

## Requirements

Both example files require **FileMaker Pro 16 or later**.

FileMaker 16 introduced the JSON functions and `Insert from URL` with cURL options that the examples use.

## Process

```text
Invoice data
    ↓
Generate XML
    ↓
Generate PDF/A-3 (if applicable)
    ↓
Validate
    ↓
Only then send
```

FileMaker holds the business data and the workflow. XML serialization, PDF/A-3 assembly, validation, and visualization are delegated to a micro web service that processes the invoice in memory only and stores no invoice data.

The PDF/A-3 step produces a **hybrid invoice**: one file that is readable as a PDF and machine-readable through the embedded XML. Factur-X and ZUGFeRD 2.5.2 are the hybrid formats here; XRechnung is pure XML.

## Not a FileMaker add-on — by design

The examples are deliberately not built as an add-on. An add-on would impose a data model on the host solution. Instead:

- the business logic stays in FileMaker,
- format handling (profiles, code lists, schemas, validation artifacts) stays in the service,
- format updates happen centrally, without touching the FileMaker file.

## Supported models

The generator addresses the service by model name:

| Model | Format |
|---|---|
| `Facturxen16931` | Factur-X / ZUGFeRD, EN 16931 profile |
| `Facturxextended` | Factur-X / ZUGFeRD, EXTENDED profile |
| `Ubl21` | UBL 2.1 Invoice |
| `Ubl21cn` | UBL 2.1 Credit Note |
| `Cii16buncoupled` | UN/CEFACT Cross Industry Invoice (CII) D16B, uncoupled |
| `Cii16bcoupled` | UN/CEFACT CII D16B, coupled |

`EXTENDED` is not `EN16931` plus a few extra fields. Repeatable elements such as `applicable_trade_tax`, `defined_trade_contact`, and `specified_trade_payment_terms` differ between the profiles, which is why `Facturxextended` has its own endpoint.

## Why JSON?

A structured intermediate representation separates the invoice business model from CII or UBL syntax. The authoritative element list per model is retrieved from the schema endpoint rather than hard-coded in the FileMaker file.

See [docs/json-interface.md](docs/json-interface.md).

## Why not build XML with FileMaker text functions?

String concatenation is technically possible, but production implementations must handle XML escaping, namespaces, element ordering, cardinalities, code lists, data types, schema changes, profile requirements, and business-rule validation.

## Generated is not valid

A technically successful XML or PDF/A-3 file is not automatically a valid electronic invoice. HTTP 200 does not mean the invoice is correct. Only the status `created_and_validated` may release an invoice for automatic dispatch.

See [docs/validation.md](docs/validation.md).

## Example files

Both files require FileMaker Pro 16 or later.

### E-Invoice Generator

`examples/e-invoice-generator/erechnungerstellung.fmp12`

- [Documentation](examples/e-invoice-generator/README.md)
- [Script](examples/e-invoice-generator/script-generate-invoice.md)

### E-Invoice Visualizer

`examples/e-invoice-visualizer/erechnungvisualisierung.fmp12`

- [Documentation](examples/e-invoice-visualizer/README.md)
- [Script](examples/e-invoice-visualizer/script-validate-invoice.md)

## Documentation

- [docs/architecture.md](docs/architecture.md) — division of responsibility, transport, why this is not an add-on
- [docs/json-interface.md](docs/json-interface.md) — schema endpoint, JSON types, CII paths
- [docs/validation.md](docs/validation.md) — response schema, SVRL, status values
- [sample-data/README.md](sample-data/README.md) — invoice JSON excerpt
- [DISCLAIMER.md](DISCLAIMER.md) — demonstration only, not compliance
- [LICENSE.md](LICENSE.md) — all rights reserved

## Business semantics remain in FileMaker

A conversion library, plugin, or web service cannot determine the correct business meaning of incomplete or incorrectly modeled invoice data. The FileMaker solution remains responsible for VAT treatment, invoice type, references, allowances, charges, payment terms, tax breakdowns, and other transaction-specific information.

Clarify those questions before mapping fields. Typical integration pitfalls: [36 typische Stolpersteine bei der E-Rechnungsintegration in FileMaker](https://www.hagemeister-conception.de/blog/e-rechnung-filemaker-zugferd-xrechnung).

## Terminology

| Term | Meaning |
|---|---|
| **EN 16931** | European standard defining the semantic data model for the core invoice |
| **CIUS** | Core Invoice Usage Specification — a national narrowing of EN 16931 |
| **XRechnung** | German CIUS of EN 16931, pure XML (occasionally written *X-Rechnung*) |
| **ZUGFeRD / Factur-X** | German / French hybrid format: PDF/A-3 with embedded CII XML |
| **CII** | Cross Industry Invoice — the UN/CEFACT XML syntax, here in version D16B |
| **UBL** | Universal Business Language — the alternative XML syntax, here 2.1 |
| **PDF/A-3** | Archivable PDF variant that may carry embedded files, such as `factur-x.xml` |
| **Schematron** | Rule language used to express EN 16931 and national business rules |
| **SVRL** | Schematron Validation Report Language — the XML report format returned by the validator |
| **KoSIT** | Coordination office publishing the XRechnung specification and the reference validator |
| **Leitweg-ID** | Routing identifier for German public-sector invoices, carried in `BT-10` |
| **Peppol** | Network for exchanging e-invoices; transport, not a document format |

## Security

- Store the API key in a protected settings table or in secrets management — never in global fields, script comments, or error logs.
- Always use HTTPS with `Verify SSL Certificates` enabled. Do not use `--insecure`.
- Keep the original file, the full API response, and a timestamp for the audit trail.

All URLs, model names, and credentials in this repository are placeholders.

## Further documentation

**Hagemeister Conception** — https://www.hagemeister-conception.de/

## Author

**Jan Hagemeister**  
Hagemeister Conception

FileMaker and Python development with a focus on integrating technologies that are not natively available in Claris FileMaker.
