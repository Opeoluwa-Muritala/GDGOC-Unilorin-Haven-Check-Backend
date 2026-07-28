# Housing Scout API Documentation

Housing Scout is a FastAPI backend for hostel listings, utility reports,
AI-assisted housing checks, and email escalation.

## Base URL and interactive docs

Local base URL: `http://127.0.0.1:8000`

- Swagger UI: `http://127.0.0.1:8000/docs`
- ReDoc: `http://127.0.0.1:8000/redoc`
- OpenAPI JSON: `http://127.0.0.1:8000/openapi.json`

No authentication is currently required.

## Setup

Python 3.11+ is required. PostgreSQL is optional; without `DATABASE_URL`, the
application uses the local SQLite file `housing_scout.db`.

```powershell
cd backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
# Edit .env, then:
python seed.py
uvicorn main:app --reload
```

Tables are created when the app starts. `python seed.py` replaces existing rows
in seeded tables, so do not use it as a production migration command.

## Environment variables

Put these in `backend/.env`. Never commit real API keys.

```dotenv
# Optional; defaults to SQLite
DATABASE_URL=postgresql://postgres:password@localhost:5432/housing_scout

# Configure at least one AI provider for POST /agent
GEMMA_API_KEY=replace-with-google-gemini-api-key
OPENROUTER_API_KEY=replace-with-openrouter-api-key

# Optional AI overrides
GEMMA_MODEL=gemma-2-27b-it
GEMMA_API_URL=https://generativelanguage.googleapis.com/v1beta/models/gemma-2-27b-it:generateContent
OPENROUTER_MODEL=google/gemma-2-27b-it:free
OPENROUTER_API_URL=https://openrouter.ai/api/v1/chat/completions

# Email service used by alert endpoints
EMAIL_URL=https://your-email-service.example.com/send
AUTHORITY_EMAIL_TO=authority@example.com
SECURITY_EMAIL_TO=security@example.com

# Required only for image file uploads
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_API_KEY=your-api-key
CLOUDINARY_API_SECRET=your-api-secret
```

| Variable | Required | Purpose |
| --- | --- | --- |
| `DATABASE_URL` | No | SQLAlchemy URL. Defaults to `sqlite:///./housing_scout.db`. PostgreSQL URLs are converted to the psycopg driver automatically. |
| `GEMMA_API_KEY` | For primary AI | Google API key used by `/agent`. |
| `OPENROUTER_API_KEY` | For AI fallback | Used if Google fails. At least one AI key is needed for useful agent replies. |
| `GEMMA_MODEL` | No | Google model; default `gemma-2-27b-it`. |
| `GEMMA_API_URL` | No | Complete Google-compatible `generateContent` URL. |
| `OPENROUTER_MODEL` | No | OpenRouter model; default `google/gemma-2-27b-it:free`. |
| `OPENROUTER_API_URL` | No | OpenRouter chat-completions URL. |
| `EMAIL_URL` | For alerts | HTTP endpoint accepting `{to, subject, text, html}` JSON. |
| `AUTHORITY_EMAIL_TO` | For authority alerts | Recipient for `notify-authority`. |
| `SECURITY_EMAIL_TO` | For security alerts | Recipient for `alert-security`. |
| `CLOUDINARY_CLOUD_NAME` | For file uploads | Cloudinary cloud name. |
| `CLOUDINARY_API_KEY` | For file uploads | Cloudinary API key. |
| `CLOUDINARY_API_SECRET` | For file uploads | Cloudinary API secret. |

`RESEND_API_KEY` and `RESEND_FROM_EMAIL` are not read by this application. If
the service behind `EMAIL_URL` uses Resend, configure those values in that
separate service.

## Standard response

Except for `/mock-email`, endpoints use:

```json
{
  "message": "Human-readable result",
  "data": {},
  "action": null,
  "error": null
}
```

Validation errors return `422` with `VALIDATION_ERROR`; missing resources return
`404`; unexpected failures return `500`.

### Hostel returned by the API

```json
{
  "id": "hostel-tanke-01",
  "name": "Harmony Lodge",
  "location": "Tanke",
  "priceNaira": 150000,
  "amenities": ["borehole", "prepaid meter"],
  "description": "Inspection available",
  "photo_url": "https://example.com/photo.jpg",
  "lat": 8.4799,
  "lng": 4.5418,
  "isSchoolManaged": false,
  "scamRiskLevel": null,
  "utility_reports": []
}
```

## Endpoint summary

| Method | Path | Takes | Returns |
| --- | --- | --- | --- |
| `GET` | `/health` | Nothing | Health status |
| `POST` | `/mock-email` | Any JSON object | Echoed JSON |
| `POST` | `/hostels` | JSON or form hostel data | Created hostel |
| `GET` | `/hostels` | Optional filters | Hostel array |
| `GET` | `/hostels/{hostel_id}` | Hostel ID | Hostel and reports |
| `PUT` | `/hostels/{hostel_id}/photo` | Multipart image | Updated hostel |
| `POST` | `/hostels/{hostel_id}/reports` | Report JSON | Created report |
| `POST` | `/hostels/{hostel_id}/notify-authority` | Complaint JSON | Notification status |
| `POST` | `/hostels/{hostel_id}/alert-security` | Alert JSON | Alert status |
| `POST` | `/agent` | Message JSON | AI reply and tool calls |

## Endpoint reference

### `GET /health`

Takes nothing. Returns `200`:

```json
{
  "message": "API is healthy",
  "data": {"status": "ok"},
  "action": null,
  "error": null
}
```

### `POST /mock-email`

Accepts any JSON object and echoes it. This development endpoint does not send
email and does not use the standard response envelope.

Request:

```json
{"to": "student@example.com", "subject": "Test"}
```

Returns `200`:

```json
{
  "message": "Email sent mock successfully",
  "payload": {"to": "student@example.com", "subject": "Test"}
}
```

### `POST /hostels`

Accepts JSON or multipart form data.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `name` | string | Yes | Hostel name |
| `location` | string | Yes | Area |
| `priceNaira` | integer | Yes | Greater than zero; `price_naira` also works |
| `amenities` | string[] | No | Defaults to `[]` |
| `description` | string | Yes | Description |
| `lat` | number/null | No | Latitude |
| `lng` | number/null | No | Longitude |
| `photoUrl` | string/null | No | Existing URL; `photo_url` also works |

JSON request:

```json
{
  "name": "Test Lodge",
  "location": "Tanke",
  "priceNaira": 145000,
  "amenities": ["borehole"],
  "description": "Inspection available",
  "lat": 8.48,
  "lng": 4.54,
  "photoUrl": "https://example.com/hostel.jpg"
}
```

For multipart, use the same text fields and a `photo` file. `amenities` must be
a JSON-encoded array. JPEG, PNG, GIF, and WebP files up to 5 MB are supported.
Cloudinary must be configured.

```bash
curl -X POST http://127.0.0.1:8000/hostels \
  -F "name=Test Lodge" -F "location=Tanke" -F "priceNaira=145000" \
  -F 'amenities=["borehole"]' -F "description=Inspection available" \
  -F "photo=@hostel.jpg"
```

Returns `201` with the created hostel in `data.hostel`. Errors include `400` for
bad images/uploads, `415` for unsupported content types, and `422` for fields.

### `GET /hostels`

Takes optional `location` (case-insensitive exact match) and `maxPrice` (positive
integer) query parameters:

```http
GET /hostels?location=Tanke&maxPrice=170000
```

Returns `200` with `data.hostels`, ordered by name:

```json
{
  "message": "Hostels retrieved successfully",
  "data": {"hostels": []},
  "action": null,
  "error": null
}
```

### `GET /hostels/{hostel_id}`

Takes the hostel UUID or seeded string ID in the path. Returns `200` with
`data.hostel`, including its `utility_reports`. Returns `404` if not found.

### `PUT /hostels/{hostel_id}/photo`

Takes multipart form data with one `photo` image. The same 5 MB/type/Cloudinary
rules as hostel creation apply.

```bash
curl -X PUT http://127.0.0.1:8000/hostels/HOSTEL_ID/photo \
  -F "photo=@hostel.jpg"
```

Returns `200` with the updated `data.hostel`, `400` for upload errors, or `404`.

### `POST /hostels/{hostel_id}/reports`

All fields are required:

```json
{
  "water_available": false,
  "electricity_issue": true,
  "comment": "No water since Monday"
}
```

Returns `201` with `data.report`:

```json
{
  "id": 1,
  "hostel_id": "hostel-tanke-01",
  "water_available": false,
  "electricity_issue": true,
  "comment": "No water since Monday",
  "reported_at": "2026-07-25T12:00:00+00:00"
}
```

Returns `404` if the hostel does not exist.

### `POST /hostels/{hostel_id}/notify-authority`

```json
{
  "issueType": "water",
  "details": "No water for three days."
}
```

Both strings are required; `issue_type` also works. Returns `200` with
`data.status` (`sent` or `failed`) and `data.timestamp`. An email failure still
uses HTTP `200` but sets `error.code` to `NOTIFY_FAILED`. Returns `404` if the
hostel is unknown.

### `POST /hostels/{hostel_id}/alert-security`

```json
{
  "reason": "Suspected fake landlord",
  "evidence": "The account appears on unrelated listings."
}
```

`reason` is required; `evidence` is optional and defaults to `""`. Returns `200`
with `data.status` and `data.timestamp`. An email failure uses HTTP `200` with
`ALERT_FAILED`. Returns `404` if the hostel is unknown.

### `POST /agent`

```json
{
  "message": "Has this hostel had water problems?",
  "hostelId": "hostel-tanke-01"
}
```

`message` is required and must not be empty. `hostelId` is optional;
`hostel_id` also works. Returns `200`:

```json
{
  "message": "Agent request completed",
  "data": {
    "reply": "The assistant's response",
    "tool_used": "checkUtilityReliability",
    "tool_calls": [
      {
        "name": "checkUtilityReliability",
        "arguments": {"hostelId": "hostel-tanke-01"},
        "result": {},
        "error": null
      }
    ]
  },
  "action": null,
  "error": null
}
```

`tool_used` is a comma-separated string or `null`. Provider failures also return
HTTP `200`, but `reply` is `null`, `tool_calls` is empty, and `error.code` is
`GEMMA_UNAVAILABLE` or `AGENT_ERROR`.

## Common errors

```json
{
  "message": "Hostel not found",
  "data": {},
  "action": null,
  "error": {"code": "HTTP_404", "details": "Hostel not found"}
}
```

```json
{
  "message": "Request validation failed",
  "data": {},
  "action": null,
  "error": {"code": "VALIDATION_ERROR", "details": []}
}
```
