# Validation

Electronic invoice validation is more than XML Schema validation, and a technically successful XML or PDF/A-3 file is not automatically a valid electronic invoice.

A practical workflow may include:

1. XML well-formedness
2. XML Schema (XSD) validation
3. EN 16931 semantic business rules, expressed as Schematron
4. National rules such as XRechnung, the German CIUS of EN 16931 published by KoSIT
5. Profile-specific requirements such as ZUGFeRD / Factur-X
6. PDF/A validation for hybrid PDF invoices

Levels 3 to 5 are rule checks, not schema checks. A document can be schema-valid and still violate dozens of business rules — which is why the `validation` array matters as much as `reports`.

Visualization is separate from validation: the ability to render an invoice does not establish conformance.

B2G note: for German public-sector invoices the Leitweg-ID in `BT-10` is subject to its own rules. It must be supplied as invoice data; no validator can reconstruct it.

## Reading the response

The validation service returns:

```json
{
  "validation": ["<EXECUTED-XSD>", "<EXECUTED-RULE-CHECK>"],
  "reports": ["<svrl:schematron-output>...</svrl:schematron-output>"],
  "format": "factur-x",
  "profile": "en16931",
  "pdf": "<BASE64-ENCODED-PDF>",
  "content": {}
}
```

Not every field is present in every response.

| Element | Type | Meaning |
|---|---|---|
| `validation` | Array | Names of the XSD / rule checks that were actually executed |
| `reports` | Array | Validation reports in SVRL format |
| `format` | Text | Detected format, e.g. `factur-x` |
| `profile` | Text | Detected profile: `minimum`, `basicwl`, `basic`, `en16931`, `extended` |
| `pdf` | Text | Base64-encoded visualization; may be absent |
| `content` | Object | Structured data extracted from the invoice; not a substitute for the original file |
| `detail` | Text/Object | Error detail (HTTP 400, 403, 500) |
| `Message` | Text | General processing message |
| `error` | Text/Object | Error text or error structure |

Use `format` and `profile` only when they are actually present. UBL Invoice, UBL CreditNote, and CII-based formats can return different `content` structures.

## SVRL

Rule violations appear in `reports` as `<svrl:failed-assert>` elements:

- `id` — rule ID
- `location` — XPath of the offending node
- `flag` — severity
- `<svrl:text>` — message

Count them with `PatternCount ( $report ; "<svrl:failed-assert" )`. For rule ID, severity, XPath, and message individually, parse the SVRL with an XML parser.

Two traps:

- **HTTP 200 does not mean the invoice is correct.** The transport succeeded; that is all.
- **An empty `reports` array means neither "valid" nor "invalid".** It may simply mean no rule check ran. Check `validation` for which checks were actually executed.

## Status values

### Validation (visualizer)

| Status | Meaning |
|---|---|
| `transport_error` | FileMaker could not reach the service |
| `invalid_response` | Response is not parsable JSON |
| `http_error` | Service answered with 400, 403, 500 … |
| `processing_error` | Response contains `detail`, `Message`, or `error` |
| `validation_incomplete` | No XSD check, or no evaluable report |
| `validation_failed` | At least one non-accepted `failed-assert` |
| `valid` | XSD check present, no non-accepted `failed-assert` |

### Creation (generator)

| Status | Meaning |
|---|---|
| `transport_error` | Service not reachable |
| `invalid_response` | Response is not parsable JSON / not a JSON object |
| `http_error` | HTTP status outside 200–299 |
| `xml_created` | XML generated, not yet validated |
| `pdfa3_created` | PDF/A-3 generated, not yet validated |
| `validation_failed` | Validation reported errors |
| `created_and_validated` | Generated **and** validated |

Only `created_and_validated` may release an invoice for automatic dispatch.

## Typical errors

| Observation | Likely cause | Action |
|---|---|---|
| FileMaker error 1631 | Connection, DNS, TLS, or timeout | Check error detail, certificate, reachability |
| HTTP 400 | Empty request or invalid JSON | Inspect `$request` with `JSONFormatElements` |
| HTTP 403 | Invalid API key | Check key and header; do not send an empty header |
| HTTP 500 | Unexpected server error | Keep the full response and timestamp for support |
| `failed-assert` in a report | Schema passed, business rule failed | Read rule ID, XPath, and message |
| `pdf` missing | Format not visualizable, or transformation failed | Inspect the full response |
| PDF unusable in the container | Base64 stored as text instead of decoded | `Base64Decode` with a `.pdf` file name |
