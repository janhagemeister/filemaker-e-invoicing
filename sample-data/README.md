# Sample data

## `invoice.json`

An **excerpt** of the invoice JSON that the generator posts to

```text
POST <API-BASE-URL>/<JSON-TO-XML-ENDPOINT>/<MODEL-NAME>
```

It contains only the paths shown in
[E-Rechnung in FileMaker erstellen](https://www.hagemeister-conception.de/blog/e-rechnung-in-filemaker-erstellen).

This is **not a complete EN 16931 invoice**. It is missing, among other things, seller and buyer, tax breakdown, totals, and payment terms.

The full path list and JSON type rules: [../docs/json-interface.md](../docs/json-interface.md).

## The schema endpoint is authoritative

Do not treat this file as the element list. Retrieve the elements for the model you are generating:

```text
GET <API-BASE-URL>/<SCHEMA-ENDPOINT>/<MODEL-NAME>
```

The structure differs per model, and `Facturxextended` in particular is not `Facturxen16931` with extra fields — repeatable elements such as `applicable_trade_tax`, `defined_trade_contact`, and `specified_trade_payment_terms` behave differently there.

See [DISCLAIMER.md](../DISCLAIMER.md).
