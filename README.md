# cloud-itonami-isco-5223

Open Occupation Blueprint for **ISCO-08 5223**: Shop Sales Assistants.

This repository designs a forkable OSS business for an independent retail floor sales assistant: a shelf-scanning robot performs stock and price checks under a governor-gated actor, so a small shop keeps its own sales and inventory records instead of renting a closed POS/retail SaaS.

## Robotics premise

All cloud-itonami verticals are designed on the premise that a **robot performs
the physical domain work**. Here a shelf-scanning robot performs stock checks, price-tag verification and restock flagging on the sales floor under an actor that proposes
actions and an independent **Retail Floor Governor** that gates them. The governor never
dispatches hardware itself; `:high`/`:safety-critical` actions (such as
operating near customers or during store opening hours in aisles) require human sign-off.

A live sample of the operator console (robotics safety console, shared template) is rendered in [docs/samples/operator-console.html](docs/samples/operator-console.html) — pure-data HTML output of `kotoba.robotics.ui`.

## Core Contract

```text
store plan + product catalog + customer request
        |
        v
Sales Advisor -> Retail Floor Governor -> advise/sell, or human sign-off
        |
        v
robot actions (gated) + operating records + audit ledger
```

No automated advice can dispatch a robot action the governor refuses, suppress
an operating record, or disclose sensitive data without governor approval and
audit evidence.

## Capability layer

Resolves via [`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation)
(ISCO-08 `5223`). Required capabilities:

- :robotics
- :forms
- :telemetry
- :audit-ledger
- :bpmn

See [`docs/business-model.md`](docs/business-model.md) and
[`docs/operator-guide.md`](docs/operator-guide.md).

## Reference implementation

`src/retail_floor/{store,governor}.cljc` is a minimal but real
implementation of the Core Contract above (pure cljc, no external deps):

- `retail-floor.store` — `Store` protocol + `MemStore`: skus, price
  changes, sales. A price-change/sale can only be recorded against a
  registered sku (sku provenance).
- `retail-floor.governor` — `RetailFloorGovernor`: `assess` gates a
  proposal against the sku env. Hard invariants force `:hold` (no sku,
  direct-write instead of `:propose`, or a price-change discounting a sku
  by more than `discount-threshold` below list price at below `:high`
  safety-class); a deep discount always requires `:high`+ safety-class
  and thus `:human-approval` — it can never be auto-approved;
  low-confidence proposals also escalate.

```bash
clojure -M:test   # 8 tests, 13 assertions, green
```

This is what backs this repo's `:maturity :implemented` entry in
[`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation) —
the 11th `cloud-itonami-isco-*` occupation to reach that tier, after
`cloud-itonami-isco-6112`, `-2221`, `-7126`, `-4321`, `-9312`, `-5322`,
`-8332`, `-1321`, `-3253` and `-6210` (ADR-2607012000).

## License

AGPL-3.0-or-later.
