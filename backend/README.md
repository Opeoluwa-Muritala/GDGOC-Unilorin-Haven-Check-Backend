# Housing Scout Backend

Independent FastAPI API for checking Unilorin-area hostel rent fairness, utility reliability, and scam risk. Gemma performs function calling: it selects from five declared functions, Python executes the selected function(s), then Gemma turns deterministic results into a final reply. This is ordinary model function calling, not an MCP server.

## Setup

Requires Python 3.11+ and PostgreSQL. SQLite is used automatically when `DATABASE_URL` is absent, which makes first-hour demos easy.

```powershell
cd backend
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
# Edit .env, then:
python seed.py
uvicorn main:app --reload
```

Open `http://127.0.0.1:8000/docs` for interactive API docs. For PostgreSQL, create the database and either run `psql "$DATABASE_URL" -f schema.sql` or let SQLAlchemy create the tables when the app/seed script starts.

## Stable response contract

Every endpoint, including validation errors and 404/500 responses, returns the same top-level structure:

```json
{
  "message": "Human-readable result",
  "data": {},
  "action": null,
  "error": null
}
```

Successful endpoint-specific values live inside `data` (for example `data.hostel`, `data.hostels`, `data.report`, or the agent's `data.reply`). On errors, `error` contains both `code` and `details`, while `data` remains an object and `action` remains present. Nullable nested fields are serialized as JSON `null`; collection fields use `[]` or `{}` rather than being omitted.

The default Gemma transport is Google's `generateContent` envelope. Set `GEMMA_API_URL` and `GEMMA_MODEL` if your Gemma provider uses a different Google-compatible endpoint. A provider with an OpenAI-only response format needs a small adapter in `gemma_client.py`.

## Resend email setup

1. Create a Resend account and API key, then put it in `.env` as `RESEND_API_KEY`.
2. Set `AUTHORITY_EMAIL_TO` and `SECURITY_EMAIL_TO` to inboxes visible during the demo.
3. For a quick test, leave `RESEND_FROM_EMAIL` unset; the code uses `Housing Scout <onboarding@resend.dev>`. Resend's test sender may restrict which recipient can receive mail.
4. For the live demo, verify a domain in Resend and set `RESEND_FROM_EMAIL=Housing Scout <alerts@yourdomain.com>`. The sender must belong to that verified domain.

The escalation tools return the Resend email ID when accepted. A `sent` result means Resend accepted the message; delivery can still be affected by recipient filtering. Test both demo inboxes before presenting.

## Test with curl

Health:

```bash
curl http://127.0.0.1:8000/health
```

Browse and filter:

```bash
curl "http://127.0.0.1:8000/hostels"
curl "http://127.0.0.1:8000/hostels?location=Tanke&maxPrice=170000"
curl "http://127.0.0.1:8000/hostels/hostel-tanke-01"
```

Create a hostel (`priceNaira` is the public API name; seed files use `price_naira`):

```bash
curl -X POST http://127.0.0.1:8000/hostels -H "Content-Type: application/json" -d '{"name":"Test Lodge","location":"Tanke","priceNaira":145000,"amenities":["borehole"],"description":"Inspection available","photo_url":null,"lat":8.48,"lng":4.54}'
```

Submit a utility report:

```bash
curl -X POST http://127.0.0.1:8000/hostels/hostel-tanke-01/reports -H "Content-Type: application/json" -d '{"water_available":false,"electricity_issue":true,"comment":"No water since Monday"}'
```

Ask the agent:

```bash
curl -X POST http://127.0.0.1:8000/agent -H "Content-Type: application/json" -d '{"message":"Is 150k for a room in Tanke fair?"}'
curl -X POST http://127.0.0.1:8000/agent -H "Content-Type: application/json" -d '{"message":"Has hostel-tanke-01 had water problems?"}'
curl -X POST http://127.0.0.1:8000/agent -H "Content-Type: application/json" -d '{"message":"Is this a scam: Pay now before viewing, address later, today only!"}'
```

Run all three agent samples with `python test_agent.py`, optionally passing a base URL.

## Seed data contract

All four files must contain a JSON array. JSON has no comments, so do not add header comments inside the files. This section is the authoritative, copyable shape. IDs referenced by reports/transcripts must exist in `hostels.json`. Use JSON `null`, `true`, and `false`, not Python spellings.

`seed_data/hostels.json`:

```json
[{"id":"hostel-tanke-01","name":"Harmony Lodge","location":"Tanke","price_naira":150000,"amenities":["borehole","prepaid meter"],"description":"Listing text","photo_url":null,"lat":8.4799,"lng":4.5418}]
```

`seed_data/area_prices.json`:

```json
[{"id":1,"location":"Tanke","avg_price_naira":155000,"price_range_low":120000,"price_range_high":190000}]
```

`seed_data/utility_reports.json` (ISO-8601 timestamps with timezone):

```json
[{"id":1,"hostel_id":"hostel-tanke-01","water_available":true,"electricity_issue":false,"comment":"Water runs each morning.","reported_at":"2026-07-20T09:00:00+00:00"}]
```

`seed_data/scam_transcripts.json` (`hostel_id` may be `null`):

```json
[{"id":1,"hostel_id":null,"transcript_text":"Pay before inspection.","is_scam":true}]
```

`python seed.py` replaces existing rows in these four tables. It is intended for demo/reset use, not production migrations.

## Tool behavior

- `checkRentFairness`: exact area lookup, percentage difference, and range verdict in Python.
- `checkUtilityReliability`: water percentage and electricity issue count in Python.
- `flagScamRisk`: regex/rule first pass; Gemma explanations are optional and safely fall back to fixed reasons.
- `notifyHostelAuthority`: emails `AUTHORITY_EMAIL_TO` through Resend.
- `alertCommunitySecurity`: emails `SECURITY_EMAIL_TO` through Resend.

Email functions return `sent` or `failed` with an ISO timestamp rather than crashing. The agent similarly returns a stable JSON error response if Gemma or one tool is unavailable. Do not send real personal evidence to demo inboxes.
