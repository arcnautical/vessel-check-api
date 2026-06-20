# Vessel Check API

Free, open API for instant vessel sanctions screening, ownership transparency, and vetting intelligence.

**No API key required. No signup. No rate limit registration.**

```bash
curl https://arcnautical.com/api/v1/vessels/9034690/check
```

```json
{
  "imo": "9034690",
  "sanctions": {
    "status": "RED",
    "detail": "1 match found"
  },
  "ownership": {
    "opacity": "MEDIUM",
    "score": 60
  },
  "vetting": {
    "grade": "B",
    "score": 36,
    "status": "marginal"
  },
  "checkedAt": "2026-03-22T17:45:52.419Z",
  "provider": "ArcNautical",
  "fullReport": "https://arcnautical.com/check?imo=9034690"
}
```

## What it checks

Every request screens the vessel against:

- **OFAC SDN** — U.S. Treasury Specially Designated Nationals list
- **EU Financial Sanctions** — European Union consolidated sanctions
- **UN Security Council** — United Nations consolidated list
- **OpenSanctions Maritime** — 3,700+ vessels, 3,300+ companies from aggregated global sanctions data
- **GLEIF** — Global Legal Entity Identifier Foundation for corporate ownership chain verification
- **Equasis PSC** — Port State Control detention and inspection records (when available)

## Response format

| Field | Type | Description |
|-------|------|-------------|
| `imo` | string | 7-digit IMO number queried |
| `sanctions.status` | `GREEN` \| `AMBER` \| `INCOMPLETE` \| `RED` | Sanctions screening result |
| `sanctions.detail` | string | Human-readable explanation |
| `sanctions.coverageComplete` | boolean | `true` only when every sanctions source loaded. `false` when a supplementary source was unavailable for this check |
| `sanctions.coverageGaps` | string[] | Names of any sources that did not load (empty when `coverageComplete`) |
| `ownership.opacity` | `LOW` \| `MEDIUM` \| `HIGH` | Corporate ownership transparency level |
| `ownership.score` | number (0-100) | Opacity score. 0 = fully transparent, 100 = completely opaque |
| `vetting.grade` | `A` \| `B` \| `C` \| `D` \| `E` | Overall vetting grade |
| `vetting.score` | number (0-100) | Composite vetting score |
| `vetting.status` | string | `acceptable`, `marginal`, `substandard`, or `unacceptable` |
| `checkedAt` | ISO 8601 | Timestamp of the check |
| `fullReport` | URL | Link to full interactive dossier |

### Sanctions status

| Status | Meaning |
|--------|---------|
| `GREEN` | No matches found across the comprehensive sanctions sources, which all loaded |
| `AMBER` | Low-confidence or fuzzy matches found — manual review recommended |
| `INCOMPLETE` | The screen could **not** be attested as clear — a core source was unreachable, the screen failed, or the vessel identity was unresolved. **Not a clearance**; re-check before relying on it |
| `RED` | Direct match on one or more sanctions lists — **do not proceed without legal review** |

### Ownership opacity

| Level | Score | What it means |
|-------|-------|---------------|
| `LOW` | 0-30 | Registered owner has a GLEIF LEI record, verifiable parent chain, transparent jurisdiction |
| `MEDIUM` | 31-60 | Partial transparency — some ownership data available but gaps exist |
| `HIGH` | 61-100 | No GLEIF record, shell company jurisdiction (Marshall Islands, Panama, Liberia, BVI, etc.), unverifiable ownership chain |

### Vetting grade

| Grade | Score | Interpretation |
|-------|-------|---------------|
| A | 0-20 | Low risk — clean detention record, transparent ownership, reputable flag state |
| B | 21-40 | Acceptable — minor risk factors present |
| C | 41-60 | Marginal — elevated risk, enhanced due diligence recommended |
| D | 61-80 | Substandard — significant risk factors, proceed with caution |
| E | 81-100 | Unacceptable — major risk indicators, strong recommendation against engagement |

## Rate limits

- **100 requests per hour** per IP address
- Results are cached for 1 hour per IMO number
- No API key required

Need higher limits? [Contact us](mailto:admin@arcnautical.com) for enterprise API access.

## Usage examples

### Python

```python
import requests

response = requests.get("https://arcnautical.com/api/v1/vessels/9034690/check")
data = response.json()

if data["sanctions"]["status"] == "RED":
    print(f"WARNING: Vessel IMO {data['imo']} has sanctions matches!")
    print(f"Details: {data['sanctions']['detail']}")
```

### JavaScript / Node.js

```javascript
const response = await fetch("https://arcnautical.com/api/v1/vessels/9034690/check");
const data = await response.json();

console.log(`Sanctions: ${data.sanctions.status}`);
console.log(`Ownership Opacity: ${data.ownership.opacity} (${data.ownership.score}/100)`);
console.log(`Vetting Grade: ${data.vetting.grade}`);
```

### cURL

```bash
# Check a specific vessel
curl -s https://arcnautical.com/api/v1/vessels/9811000/check | jq .

# Batch check (shell script)
for imo in 9034690 9811000 9295345 9703318; do
  echo "=== IMO $imo ==="
  curl -s "https://arcnautical.com/api/v1/vessels/$imo/check" | jq '{imo, sanctions: .sanctions.status, grade: .vetting.grade}'
  sleep 1
done
```

### Google Sheets (Apps Script)

```javascript
function checkVessel(imo) {
  const response = UrlFetchApp.fetch(`https://arcnautical.com/api/v1/vessels/${imo}/check`);
  const data = JSON.parse(response.getContentText());
  return data.sanctions.status; // "GREEN", "AMBER", "INCOMPLETE", or "RED"
}
```

## Try these vessels

| IMO | Vessel | What you'll see |
|-----|--------|----------------|
| `9034690` | LUMA | Sanctions: **RED** — OFAC-designated Feb 2026 (Iran sanctions) |
| `9811000` | EVER GIVEN | Sanctions: **GREEN** — clean record |
| `9295345` | KALLISTA | Ownership opacity: **HIGH** — Altomare SA / OFAC identity theft case |
| `9703318` | MSC OSCAR | One of the world's largest container ships |

## Data sources

All data is sourced from publicly available databases:

- [OFAC SDN List](https://ofac.treasury.gov/specially-designated-nationals-and-blocked-persons-list-sdn-human-readable-lists) — U.S. Treasury
- [EU Financial Sanctions](https://data.europa.eu/data/datasets/consolidated-list-of-persons-groups-and-entities-subject-to-eu-financial-sanctions) — European Commission
- [UN Security Council Consolidated List](https://www.un.org/securitycouncil/content/un-sc-consolidated-list) — United Nations
- [OpenSanctions](https://www.opensanctions.org/) — Aggregated global sanctions data (maritime dataset)
- [GLEIF](https://www.gleif.org/) — Legal Entity Identifier Foundation
- [Equasis](https://www.equasis.org/) — Port State Control inspection data

## Full platform

This free API provides a traffic-light summary. For full vessel intelligence including:

- Detailed sanctions match evidence (which lists, confidence scores, matched entities)
- Complete corporate ownership chain with GLEIF parent walking
- Shell company jurisdiction detection and opacity scoring
- PSC detention history and detention probability model
- Route-level voyage risk scoring (10 intelligence signals, 500 Monte Carlo simulations)
- Downloadable PDF dossier reports
- Fleet monitoring with automated alerts
- RESTful API with webhook delivery

Visit [arcnautical.com](https://arcnautical.com) or try the [interactive vessel check](https://arcnautical.com/check).

## About

Built by [ArcNautical](https://arcnautical.com) — maritime risk intelligence for insurers, compliance teams, and fleet managers.

Contact: admin@arcnautical.com

## License

The API is free to use for any purpose. No attribution required, but appreciated.

Data sources retain their original licenses. OFAC, EU, and UN sanctions data is public domain. OpenSanctions data is licensed under CC BY 4.0. GLEIF data is subject to GLEIF terms of use.
