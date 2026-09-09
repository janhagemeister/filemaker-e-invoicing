# E-Invoice Visualizer for FileMaker

**File:** `erechnungvisualisierung.fmp12`

**Requires:** FileMaker Pro 16 or later

This FileMaker example demonstrates how a validation and visualization service is integrated into a FileMaker workflow, following [E-Rechnung in FileMaker validieren und visualisieren](https://www.hagemeister-conception.de/blog/e-rechnung-in-filemaker-validieren-visualisieren).

An XML or PDF invoice is sent to the service, which validates it and returns a readable PDF visualization. The service stores no invoice data.

## Fields (table `Rechnung`)

| Field | Type | Purpose |
|---|---|---|
| `Rechnung::Originaldatei` | Container | XML / PDF input |
| `Rechnung::Visualisierung` | Container | Returned PDF |
| `Rechnung::API_Antwort` | Text | Full JSON response, for diagnostics |
| `Rechnung::Validierungen` | Text | Checks that were executed |
| `Rechnung::Pruefberichte` | Text | SVRL reports |
| `Rechnung::StrukturierteDaten` | Text | Optional data from `content` |
| `Rechnung::Validierungsstatus` | Text | Summarized status |

The API key lives in a separate settings table: `Einstellungen::API_Key`.

## Response

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
| `validation` | Array | Names of the XSD / rule checks that actually ran |
| `reports` | Array | Reports in SVRL format; `svrl:failed-assert` marks a rule violation |
| `format` | Text | Detected format, e.g. `factur-x` |
| `profile` | Text | `minimum`, `basicwl`, `basic`, `en16931`, `extended` |
| `pdf` | Text | Base64-encoded visualization; may be absent |
| `content` | Object | Structured invoice data; not a substitute for the original file |
| `detail` | Text/Object | Error detail on HTTP 400, 403, 500 |
| `Message` | Text | General processing message |
| `error` | Text/Object | Error text or error structure |

## Script

The full script is in [script-validate-invoice.md](script-validate-invoice.md). In outline:

1. **Check the input** — abort if `Rechnung::Originaldatei` is empty.
2. **Build the request** — `Base64EncodeRFC ( 4648 ; Rechnung::Originaldatei )` as `data`, with no added line breaks.
3. **Build the cURL options** — `POST`, `Content-Type: application/json`, optional `x-api-key`, `--connect-timeout 15`, `--max-time 120`, `--show-error`, `--dump-header`. Do not use `--fail`.
4. **Send** — `Insert from URL` with `Verify SSL Certificates`; capture `Get ( LastError )` and `Get ( LastErrorDetail )` in the same first `Set Variable` step; store the raw response in `Rechnung::API_Antwort`.
5. **Handle HTTP status and transport errors** — read the status line backwards out of the dumped headers; a FileMaker error with no HTTP status is `transport_error`; anything outside 200–299 is `http_error`.
6. **Check the JSON** — `JSONFormatElements` to detect a parse error, verify the root is a `JSONObject`, then `JSONListKeys` + `FilterValues` before reading `detail`, `Message`, `error`.
7. **Take over the validations** — `JSONListValues ( $response ; "validation" )` into `Rechnung::Validierungen`.
8. **Count and store the SVRL reports** — loop over `reports[$i]`, sum `PatternCount ( $report ; "<svrl:failed-assert" )`, store the formatted array in `Rechnung::Pruefberichte`.
9. **Store the visualization** — `Base64Decode ( $pdfBase64 ; "Rechnungsvisualisierung.pdf" )` into `Rechnung::Visualisierung`. The file name is what makes FileMaker produce usable container data.
10. **Take over the structured data** — `JSONFormatElements ( $content )` into `Rechnung::StrukturierteDaten`.

## Reading the SVRL reports

Inside `reports`, a violation appears as `<svrl:failed-assert>` with:

- `id` — rule ID
- `location` — XPath of the offending node
- `flag` — severity
- `<svrl:text>` — message

## Status values

| Status | Meaning |
|---|---|
| `transport_error` | FileMaker could not reach the service |
| `invalid_response` | Response is not parsable JSON |
| `http_error` | Service answered with 400, 403, 500 … |
| `processing_error` | Response contains `detail`, `Message`, or `error` |
| `validation_incomplete` | No XSD check, or no evaluable report |
| `validation_failed` | At least one non-accepted `failed-assert` |
| `valid` | XSD check present, no non-accepted `failed-assert` |

## Visualization is not validation

A successfully rendered invoice is not necessarily a valid invoice, and HTTP 200 does not mean the invoice is correct. An empty `reports` array means neither "valid" nor "invalid" — it may mean no rule check ran, so check `validation` for what actually executed.

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

## Acceptance test

1. A valid XML invoice
2. An XML invoice that is readable but wrong on the business level
3. An invalid or corrupted XML file
4. A PDF/A invoice with and without an embedded `factur-x.xml`
5. Empty request, correct API key, invalid API key
6. Unreachable endpoint or forced timeout
7. A response with and one without a PDF
8. Execution in FileMaker Pro and as a FileMaker Server script

**Pass criterion:** the solution visibly distinguishes transport errors, HTTP errors, business-level errors, and a valid invoice from one another.

## Security

- Store the API key in a protected settings table or in secrets management — never in global fields, script comments, or error logs.
- Use HTTPS only, with `Verify SSL Certificates` enabled. Do not use `--insecure`.
- Keep the original file, the full response, and a timestamp for the audit trail.

## Technical reference

- [`Insert from URL`](https://help.claris.com/en/pro-help/content/insert-from-url.html)
- [Supported cURL options](https://help.claris.com/en/pro-help/content/curl-options.html)
- [`Base64EncodeRFC`](https://help.claris.com/en/pro-help/content/base64encoderfc.html)
- [`Base64Decode`](https://help.claris.com/en/pro-help/content/base64decode.html)
- [Working with the JSON functions](https://help.claris.com/en/pro-help/content/json-functions.html)

See [DISCLAIMER.md](../../DISCLAIMER.md).

Further information: https://www.hagemeister-conception.de/
