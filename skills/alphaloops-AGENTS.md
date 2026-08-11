# loopsh — Agent Usage Guide

`loopsh` is the CLI for the AlphaLoops Freight API, which provides FMCSA carrier data:
profiles, fleet, inspections, crashes, authority history, news, and contact enrichment.

---

## Installation & Auth

```bash
pip install alphaloops-freight-cli
```

The API key is read from `~/.alphaloops` (set by `loopsh login`) or the env var
`ALPHALOOPS_API_KEY`. You can also place a `.alphaloops` file in your project
directory — the CLI walks up from cwd to home looking for it.

Confirm auth with:

```bash
loopsh whoami
```

---

## Key Concepts

| Term | Meaning |
|------|---------|
| **DOT number** | USDOT number — primary carrier identifier |
| **MC number** | Motor Carrier docket number — secondary identifier |
| **`--json`** | Global flag: emit raw JSON instead of pretty tables. Always use this in scripts. |
| **`--python`** | Print equivalent Python SDK code instead of running the command. |
| **`--typescript`** | Print equivalent TypeScript SDK code instead of running the command. |

---

## Command Reference

### Carrier Profiles

```bash
# By DOT number
loopsh carriers get <DOT>
loopsh carriers <DOT>                        # shortcut

# Specific fields only (faster, cheaper)
loopsh carriers get <DOT> --fields legal_name,phone,power_units,safety_rating

# By MC/MX docket number
loopsh carriers mc <MC_NUMBER>

# Fuzzy name search
loopsh carriers search "<NAME>"
loopsh carriers search "<NAME>" --state TX --city Houston --limit 5

# Authority history (grants, revocations, reinstatements)
loopsh carriers authority <DOT>

# News / press mentions
loopsh carriers news <DOT>
loopsh carriers news <DOT> --start-date 2025-01-01 --end-date 2025-12-31
```

### Fleet

```bash
loopsh fleet trucks <DOT>
loopsh fleet <DOT>                           # shortcut for trucks
loopsh fleet trailers <DOT>
loopsh fleet trucks <DOT> --limit 200 --offset 0
```

### Inspections

```bash
loopsh inspections list <DOT>
loopsh inspections <DOT>                     # shortcut
loopsh inspections violations <INSPECTION_ID>
```

### Crashes

```bash
loopsh crashes list <DOT>
loopsh crashes <DOT>                         # shortcut
loopsh crashes <DOT> --severity FATAL
loopsh crashes <DOT> --start-date 2024-01-01 --end-date 2024-12-31
# --severity values: FATAL | INJURY | TOW | PROPERTY_DAMAGE
```

### Contacts

```bash
# Search contacts at a carrier
loopsh contacts search --dot <DOT>
loopsh contacts search --company "<NAME>" --levels c_suite,vp
loopsh contacts search --dot <DOT> --title "operations"
# --levels values: c_suite | vp | director | manager

# Enrich a contact (costs 1 credit — adds email, phone, work history)
loopsh contacts enrich <CONTACT_ID>
```

---

## JSON Output & jq Pipelines

Always use `--json` when parsing output in scripts or agent tool calls.

```bash
# Extract a single field
loopsh --json carriers get <DOT> | jq '.legal_name'

# Search → grab top DOT → fetch full profile
DOT=$(loopsh --json carriers search "Swift Transportation" | jq -r '.results[0].dot_number')
loopsh --json carriers get "$DOT"

# Get all contact IDs for a carrier
loopsh --json contacts search --dot <DOT> | jq -r '.contacts[].id'

# Enrich first contact found
ID=$(loopsh --json contacts search --dot <DOT> | jq -r '.contacts[0].id')
loopsh --json contacts enrich "$ID"

# Count crashes by severity
loopsh --json crashes <DOT> | jq '.crashes | group_by(.severity) | map({(.[0].severity): length}) | add'
```

---

## JSON Response Shapes

### `carriers get`
```json
{
  "dot_number": "69013",
  "legal_name": "WERNER ENTERPRISES INC",
  "dba_name": null,
  "mc_number": "183261",
  "phone": "4028956640",
  "email_address": "info@werner.com",
  "physical_address": { "street": "...", "city": "OMAHA", "state": "NE", "zip": "68138" },
  "power_units": 8200,
  "drivers": 10500,
  "safety_rating": "Satisfactory",
  "operating_authority_status": "AUTHORIZED",
  "carrier_type": ["authorized_for_hire"],
  "authority_types": ["COMMON", "CONTRACT"],
  "technology": { "telematics": [], "tms": [], "fuel_card": [] },
  "growth": { "trucks_added_24m": null, "drivers_added_24m": null },
  "company_officers": [],
  "date_added": "1980-12-01"
}
```

### `carriers search`
```json
{
  "total_results": 44,
  "results": [
    {
      "dot_number": "69013",
      "legal_name": "WERNER ENTERPRISES INC",
      "dba_name": null,
      "physical_city": "OMAHA",
      "physical_state": "NE",
      "confidence": 1.0
    }
  ],
  "pagination": { "page": 1, "limit": 10, "total_results": 44, "total_pages": 5 }
}
```

### `fleet trucks`
```json
{
  "dot_number": "69013",
  "total_trucks": 8200,
  "results": [
    {
      "vin": "1XKAD49X0XJ000001",
      "make": "KENWORTH",
      "model_year": "2024",
      "gvw": "80000",
      "cab_type": "CONVENTIONAL"
    }
  ]
}
```

### `fleet trailers`
```json
{
  "dot_number": "69013",
  "total_trailers": 24000,
  "results": [
    {
      "vin": "1GRAA0622DB000001",
      "manufacturer": "GREAT DANE",
      "model_year": "2023",
      "type": "VAN",
      "reefer": false
    }
  ]
}
```

### `inspections list`
```json
{
  "dot_number": "69013",
  "total_inspections": 1250,
  "results": [
    {
      "inspection_id": "INS-12345",
      "date": "2025-03-01",
      "state": "NE",
      "level": 3,
      "violations_total": 1,
      "oos_total": 0
    }
  ]
}
```

### `inspections violations`
```json
{
  "inspection_id": "INS-12345",
  "total_violations": 1,
  "results": [
    {
      "code": "395.8",
      "description": "Driver not in possession of required logs",
      "basic_category": "HOS",
      "oos": false
    }
  ]
}
```

### `crashes list`
```json
{
  "dot_number": "69013",
  "total_crashes": 85,
  "results": [
    {
      "crash_id": "CRA-98765",
      "date": "2025-01-15",
      "state": "NE",
      "city": "OMAHA",
      "severity": "INJURY",
      "fatalities": 0,
      "injuries": 1,
      "tow_away": true
    }
  ]
}
```

### `contacts search`
```json
{
  "dot_number": "69013",
  "total_contacts": 15,
  "contacts": [
    {
      "id": "HDTokLjK7VouzDxs5htfBQ_0000",
      "full_name": "Derek Leathers",
      "first_name": "Derek",
      "last_name": "Leathers",
      "job_title": "Chairman & CEO",
      "job_title_role": "executive",
      "job_title_levels": ["c_suite"],
      "job_company_name": "Werner Enterprises",
      "linkedin_url": "linkedin.com/in/derek-leathers"
    }
  ]
}
```

### `contacts enrich`
```json
{
  "id": "HDTokLjK7VouzDxs5htfBQ_0000",
  "full_name": "Derek Leathers",
  "job_title": "Chairman & CEO",
  "job_company_name": "Werner Enterprises",
  "work_email": "dleathers@werner.com",
  "personal_emails": [],
  "phone_numbers": ["+14028956640"],
  "mobile_phone": null,
  "linkedin_url": "linkedin.com/in/derek-leathers",
  "location_name": "Omaha, NE",
  "skills": ["Supply Chain", "Logistics", "Fleet Management"],
  "experience": [
    {
      "title": "Chairman & CEO",
      "company": "Werner Enterprises",
      "start_date": "2016-07",
      "end_date": null
    }
  ],
  "credits": { "used": 1, "remaining": 99, "total": 100 }
}
```

---

## Error Handling

Exit code is non-zero on errors. Common errors:

| Error | Meaning |
|-------|---------|
| `No trucks found.` | Carrier has no fleet records in FMCSA |
| `No inspections found.` | No inspection history available |
| `Carrier not found` | DOT or MC number does not exist |
| `Authentication failed` | Bad or missing API key |

In scripts, handle gracefully:
```bash
if loopsh --json fleet trucks "$DOT" > fleet.json 2>/dev/null; then
  jq '.results | length' fleet.json
else
  echo "fleet data unavailable"
fi
```

---

## Common Agent Workflows

### Qualify a carrier by name
```bash
DOT=$(loopsh --json carriers search "Acme Freight" | jq -r '.results[0].dot_number')
loopsh --json carriers get "$DOT" | jq '{
  name: .legal_name,
  dot: .dot_number,
  mc: .mc_number,
  trucks: .power_units,
  safety: .safety_rating,
  status: .operating_authority_status
}'
```

### Find decision-makers at a carrier
```bash
DOT=$(loopsh --json carriers search "Acme Freight" | jq -r '.results[0].dot_number')
loopsh --json contacts search --dot "$DOT" --levels c_suite,vp | jq '.contacts[] | {name: .full_name, title: .job_title}'
```

### Safety due-diligence snapshot
```bash
DOT=69013
echo "--- Profile ---"
loopsh --json carriers get "$DOT" | jq '{safety_rating, operating_authority_status, power_units}'
echo "--- Recent crashes (FATAL) ---"
loopsh --json crashes "$DOT" --severity FATAL | jq '.results | length'
echo "--- Inspection count ---"
loopsh --json inspections "$DOT" | jq '.total_inspections'
```

### Paginate all search results
```bash
PAGE=1
while true; do
  RESP=$(loopsh --json carriers search "Schneider" --page "$PAGE" --limit 50)
  echo "$RESP" | jq '.results[].dot_number'
  TOTAL_PAGES=$(echo "$RESP" | jq '.pagination.total_pages')
  [ "$PAGE" -ge "$TOTAL_PAGES" ] && break
  PAGE=$((PAGE + 1))
done
```

### Generate SDK code for any command
```bash
# See how to do it in Python
loopsh --python carriers search "Werner" --state NE

# See how to do it in TypeScript
loopsh --typescript contacts enrich HDTokLjK7VouzDxs5htfBQ_0000
```
