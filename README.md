# Vessel Check API

Screen a ship by IMO number for sanctions, ownership opacity and a vetting grade. The first call needs no key and no signup.

```bash
curl https://arcnautical.com/api/v1/vessels/9034690/check
```

The response, abridged (live on 2026-10-10):

```json
{
  "imo": "9034690",
  "vesselName": "LUMA",
  "checkDigitValid": true,
  "sanctions": {
    "status": "RED",
    "detail": "2 confirmed matches on vessel identifier.",
    "coverageComplete": true,
    "coverageGaps": []
  },
  "ownership": { "opacity": "HIGH", "score": 100 },
  "ownershipFactors": [
    { "label": "Owner not identified, so no LEI (GLEIF) lookup was possible", "points": 35 },
    { "label": "Flag jurisdiction VU — Tier 3 — Opaque", "points": 25 },
    { "label": "No public beneficial-ownership record on a flag-of-convenience register", "points": 10 },
    { "label": "OFAC-designated entity in ownership chain (chain-sanctions floor applied)", "points": 30 }
  ],
  "vetting": { "grade": "E", "score": 90, "status": "unacceptable" },
  "assessed": true,
  "checkedAt": "2026-10-10T18:45:05.318Z",
  "fullReport": "https://arcnautical.com/check?imo=9034690",
  "tier": "keyless"
}
```

The full API reference is at **[arcnautical.com/developers](https://arcnautical.com/developers/)**. The OpenAPI 3.1 spec is at [`/api/v1/openapi.json`](https://arcnautical.com/api/v1/openapi.json).

## What one call checks

- **Sanctions.** Five lists: OFAC SDN, UK OFSI, EU Financial Sanctions (FSD), the UN Security Council Consolidated List, and OpenSanctions' maritime dataset. The hull is matched on its IMO number, its current name and the other names our sources hold for it. A former name that none of our sources holds is not screened, so a ship renamed after a designation under a name we never saw can still read `GREEN`.
- **Ownership opacity.** A 0-100 score for how much of the ownership is on the public record: whether the owner has a GLEIF LEI, how transparent the flag register is, and whether a designated company sits in the chain. `ownershipFactors` lists the lines the score is the sum of.
- **Vetting grade.** A to E, a composite of five factors: detention history, flag performance, vessel age, inspection gap and ownership opacity. Detention and inspection history comes from port state control records. Which regimes we cover, and the gaps, are listed in the [methodology](https://arcnautical.com/methodology.html).

## Response fields

| Field | Type | Meaning |
|---|---|---|
| `imo` | string | The seven-digit IMO that was screened |
| `vesselName` | string \| null | The name our sources hold for this hull, so you can confirm which ship was screened. `null` when no source names it |
| `alsoRecordedAs` | array | Other names our sources hold for the hull, each with its source. Every one was screened on this call |
| `identityBasis` | `sources` \| `user` \| `none` | `sources` means our records name the hull. `none` means nothing named it: its identifiers were screened, a name was not |
| `checkDigitValid` | boolean | `false` when the IMO fails its check digit. That is almost always a typo |
| `sanctions.status` | `RED` \| `AMBER` \| `INCOMPLETE` \| `GREEN` | See below |
| `sanctions.detail` | string | The verdict in a sentence |
| `sanctions.coverageComplete` | boolean | `true` when every sanctions list loaded |
| `sanctions.coverageGaps` | string[] | The lists that did not load |
| `ownership.opacity` | `LOW` \| `MEDIUM` \| `HIGH` \| null | Opacity band |
| `ownership.score` | number \| null | 0 = transparent, 100 = opaque |
| `ownershipFactors` | array \| null | The lines `ownership.score` is the sum of, with their points. Labels are for people to read, not codes to parse |
| `vetting.grade` | `A`-`E` \| null | Vetting grade |
| `vetting.score` | number \| null | 0-100, lower is better |
| `vetting.status` | string | `acceptable`, `marginal`, `substandard`, `unacceptable`, or `not_assessed` |
| `assessed` | boolean | `false` means ownership and vetting were not measured: their values are `null`, with no stand-in. The sanctions verdict is unaffected |
| `checkedAt` | ISO 8601 | When the check ran |
| `fullReport` | URL | The human-readable report for this hull |
| `tier` / `tierDetail` | string | Always `keyless` here, and what a free key adds |

### Sanctions status

| Status | Meaning |
|---|---|
| `RED` | Confirmed match. Do not proceed without legal review |
| `AMBER` | Possible match that needs a person to review it |
| `INCOMPLETE` | No match, but the screen cannot be called clear. A core list (OFAC SDN, OpenSanctions, EU FSD) did not load, or no source could name the hull, or the IMO fails its check digit. `detail` says which. **Never treat it as clear.** It is never cached, so a retry gets a fresh screen |
| `GREEN` | No match on the lists that loaded. If `coverageComplete` is `false`, one supplementary list was unavailable: re-screen rather than treat it as a clean pass |

### Ownership opacity

| Band | Score |
|---|---|
| `LOW` | 0-30 |
| `MEDIUM` | 31-60 |
| `HIGH` | 61-100 |

Opacity measures the public record, not wrongdoing. Ordinary tonnage on an open register can score `HIGH` simply because the owner has no LEI. Read `ownershipFactors` to see why.

### Vetting grade

| Grade | Score | `vetting.status` |
|---|---|---|
| A | 0-20 | `acceptable` |
| B | 21-40 | `acceptable` |
| C | 41-60 | `marginal` |
| D | 61-80 | `substandard` |
| E | 81-100 | `unacceptable` |

## Limits

- **100 requests an hour per IP address.** The `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers on every response are the authority. The window slides, so `X-RateLimit-Reset` is when your oldest request ages out.
- A refused request gets `429` with `Retry-After`. A tight loop is refused before the hourly allowance is spent. Honor `Retry-After` and the loop completes.
- A hull anyone checked in the last hour answers from cache. A hull that needs a fresh screen waits for one of a few screening slots shared by every keyless caller. If none frees up in time you get `503 keyless_busy` with `Retry-After`, and it does not count against your allowance.
- An IMO without exactly one seven-digit number in it gets `400 invalid_imo`, which does not count either. `9811000` and `IMO9811000` both work.
- The keyless check is a sample for evaluation and occasional lookups, not a data feed. An automated client that runs a product on it can be held to a small monthly sample. The refusal names the free key that replaces it.

Every limit and error code is documented at [arcnautical.com/developers](https://arcnautical.com/developers/).

## With a free key

A self-serve API key is free, with no card. `POST /api/v1/screenings` runs the same engine and returns the full record the keyless check leaves out: every sanctions match with its list, programme and confidence class, each source reported separately, the five weighted vetting factors, and a stored screening record you can cite later. The same key does batches of 50, standing monitors, fleets, signed webhooks and voyage risk scoring.

Get one at **[arcnautical.com/get-a-key](https://arcnautical.com/get-a-key?from=github)**.

## Examples

### Python

```python
import requests

r = requests.get("https://arcnautical.com/api/v1/vessels/9034690/check", timeout=30)
if r.status_code == 429:
    raise SystemExit(f"Rate limited, retry in {r.headers.get('Retry-After')} s")
r.raise_for_status()
v = r.json()

status = v["sanctions"]["status"]
if status in ("RED", "AMBER"):
    print(f"Review IMO {v['imo']} ({v['vesselName']}): {v['sanctions']['detail']}")
elif status == "INCOMPLETE":
    print(f"Not a clearance: {v['sanctions']['detail']}")
elif not v["sanctions"]["coverageComplete"]:
    print(f"Clear on the lists that loaded; missing {v['sanctions']['coverageGaps']}. Re-screen.")
else:
    print(f"No match for IMO {v['imo']} ({v['vesselName']})")
```

### JavaScript

```javascript
const res = await fetch("https://arcnautical.com/api/v1/vessels/9034690/check");
const v = await res.json();

console.log(`Sanctions: ${v.sanctions.status}`);
if (v.assessed) {
  console.log(`Ownership opacity: ${v.ownership.opacity} (${v.ownership.score}/100)`);
  console.log(`Vetting grade: ${v.vetting.grade} (${v.vetting.status})`);
} else {
  console.log("Ownership and vetting: not assessed");
}
```

### cURL

```bash
curl -s https://arcnautical.com/api/v1/vessels/9811000/check | jq '{imo, vesselName, sanctions: .sanctions.status, grade: .vetting.grade}'

# A few hulls, one after another
for imo in 9034690 9811000 9703318; do
  curl -s "https://arcnautical.com/api/v1/vessels/$imo/check" | jq -c '{imo, vesselName, sanctions: .sanctions.status}'
  sleep 1
done
```

### Google Sheets (Apps Script)

```javascript
function VESSEL_SANCTIONS(imo) {
  const res = UrlFetchApp.fetch(`https://arcnautical.com/api/v1/vessels/${imo}/check`);
  return JSON.parse(res.getContentText()).sanctions.status; // RED, AMBER, INCOMPLETE or GREEN
}
```

## From an AI agent (MCP)

The same check is an MCP tool. Add the remote endpoint `https://mcp.arcnautical.com/mcp` with no authentication and `check_vessel` and `find_port` work at once. To run it locally instead: `npx -y @arcnautical/mcp`. Setup for Claude, ChatGPT, Cursor and others: [arcnautical.com/developers/#mcp](https://arcnautical.com/developers/#mcp).

## Try these

| IMO | Vessel | What you will see |
|---|---|---|
| `9034690` | LUMA | `RED`: confirmed matches on the vessel identifier |
| `9811000` | EVER GIVEN | `GREEN`, grade A, opacity `HIGH` (70): a clean ship whose ownership is not on the public record |
| `9703318` | MSC ZOE | `GREEN`, grade A |
| `9295345` | none | `INCOMPLETE`: the number fails its IMO check digit, so it belongs to no vessel |

These were the answers on 2026-10-10. Verdicts change when the lists change.

## Data

Sanctions lists from the US Treasury (OFAC), HM Treasury (OFSI), the European Union, the UN Security Council and [OpenSanctions](https://www.opensanctions.org/). Ownership data from [GLEIF](https://www.gleif.org/). Port state control records from the MoU regimes named in the [methodology](https://arcnautical.com/methodology.html).

Use of the API is covered by the [terms of service](https://arcnautical.com/terms.html). Questions: admin@arcnautical.com.

Built by [ArcNautical](https://arcnautical.com).
