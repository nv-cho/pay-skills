---
name: earth-pulse-api
title: "Earth Pulse API"
description: "Regional earthquake, wildfire, tornado, cyclone and tsunami data from USGS, NOAA and NASA feeds, with measurements, geometry, source links, observation times, freshness and coverage gaps in structured query results."
use_case: "Use for recent natural-hazard research, regional earthquake analysis, tornado warning lookup, cyclone tracking, tsunami bulletin research and satellite fire-observation aggregation with attributed evidence and explicit coverage limits."
category: data
service_url: https://earth-pulse-api.agentico-labs.workers.dev
version: v1
openapi:
  path: openapi.json
---

# Earth Pulse API

Query normalized natural-hazard data for a geographic bounding box and a recent
UTC interval. A successful purchase costs **$0.01 USDC on Solana mainnet** through
x402 v2 exact. No service subscription or API key is required. A funded Pay
wallet and the user's payment authorization are required to purchase data.

## Choose a query

Check the free endpoints first:

- `GET /health` reports service and dataset readiness.
- `GET /v1/sources` reports source coverage, freshness, query bounds and price.
- `GET /openapi.json` provides the current public request and response contract.

`POST /v1/query` accepts JSON and a required `Idempotency-Key` header. Generate
an unpredictable UUID and save it with the exact body before requesting payment.

| Field | Meaning and constraints |
|---|---|
| `bbox` | `[west, south, east, north]` in longitude/latitude degrees. Antimeridian-crossing boxes are unsupported. |
| `hazards` | One or more of `earthquake`, `wildfire`, `tornado`, `cyclone`, `tsunami`. |
| `from`, `to` | UTC ISO timestamps with `Z`; select source times in `[from, to)`. The positive interval is at most 24 hours and must fit within the recent seven-day query horizon. |
| `limit` | Maximum combined event records and fire cells, from 1 to 100; defaults to 100. This is a completeness bound, not a truncation or pagination setting. |

Prepare current timestamps rather than copying the dated OpenAPI example. This
example saves one California earthquake request for the preceding 24 hours:

```sh
request_dir=$(mktemp -d)
node --input-type=module - "$request_dir" <<'NODE'
import { randomUUID } from 'node:crypto';
import { writeFileSync } from 'node:fs';
const dir = process.argv[2];
const now = Date.now();
const query = {
  bbox: [-125, 32, -114, 42], hazards: ['earthquake'],
  from: new Date(now - 24 * 3600000).toISOString(),
  to: new Date(now).toISOString(), limit: 100
};
writeFileSync(`${dir}/request.json`, JSON.stringify(query), { mode: 0o600 });
writeFileSync(`${dir}/request-key.txt`, randomUUID(), { mode: 0o600 });
NODE
```

Keep `request_dir`, the key and body until delivery is confirmed. Use plain
`curl` first if an unpaid preview of the payment terms is needed. A usable
request returns HTTP 402 after its complete result has been prepared and pinned.
Purchase with Pay only within the user's authorized budget:

```sh
pay curl --silent --show-error --fail-with-body \
  --request POST https://earth-pulse-api.agentico-labs.workers.dev/v1/query \
  --header 'Content-Type: application/json' \
  --header "Idempotency-Key: $(cat "$request_dir/request-key.txt")" \
  --data-binary "@$request_dir/request.json" \
  --output "$request_dir/result.json"
```

## Interpret the result

Responses include `datasetId`, `generatedAt`, the canonical query, `sources`,
`records`, `fireCells`, `counts`, `contextCodes` and a verified payment receipt.
Measurements retain units, source links and relevant observation, issuance or
validity times. Inspect coverage and context before drawing conclusions.

- The API reads its latest successfully exported dataset; queries do not trigger
  collection. Feed checks and observation timestamps describe different clocks.
- USGS earthquake coverage is magnitude 2.5+; magnitude scales and review status
  remain explicit.
- Warning-polygon matching uses geometry bounding-box candidate overlap, not
  exact polygon intersection. Point coordinates may identify an epicenter,
  cyclone center or tsunami bulletin's source earthquake.
- Wildfire results summarize satellite thermal detections in fixed 1-degree
  cells. Whole boundary cells and repeated observations are identified; detection
  counts do not represent unique or confirmed fires.
- Recent query bounds do not guarantee complete historical or worldwide
  coverage. Partial usable source coverage and gaps remain explicit.

## Payment and recovery

- Each new prepared query costs $0.01, including a valid query with no matches.
- Invalid input, an overly broad query or a query with no usable source data is
  rejected before payment. Narrow the region, interval or hazard selection for
  `422`; lowering `limit` does not truncate results.
- Check the challenge for x402 v2 exact, Solana mainnet and USDC with amount
  `10000` base units. Unpaid quotes expire after five minutes.
- Reuse the original key and exact body for the payment retry. Settled results
  are recoverable for 24 hours by repeating that request with plain `curl`,
  without a payment header or another purchase.
- Changing the query under an existing key conflicts. After a timeout or an
  uncertain settlement, preserve the identity and attempt recovery; do not
  generate a new key and pay again until the original outcome is resolved.
- Avoid automatic polling purchases. Batch supported hazards into one bounded
  request where useful, and use free source metadata before spending.

Source code: [nv-cho/earth-pulse-api](https://github.com/nv-cho/earth-pulse-api).
Public visualization: [Earth Pulse observatory](https://earth-pulse-observatory.zigniz.chatgpt.site/).
