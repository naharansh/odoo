# Leads API — Documentation

## Overview
A public HTTP endpoint that lets external systems (websites, forms, third-party apps) create CRM leads in Odoo via a simple POST request, authenticated with a static bearer token stored in system parameters.

**Base module:** custom controller (`LeadsAPI`)
**Endpoint:** `/api/create_lead`
**Method:** `POST` (with `OPTIONS` supported for CORS preflight)
**Auth type:** `public` (auth is handled manually via bearer token, not Odoo session)

---

## Authentication

The endpoint expects a bearer token in the `Authorization` header:

```
Authorization: Bearer <token>
```

The token is compared against the value stored in:

```
Settings > Technical > System Parameters > api.auth_token
```

If the header is missing, malformed, or the token doesn't match, the API returns:

```json
{
  "status": 401,
  "message": "Unauthorized"
}
```

**Note:** Set `api.auth_token` in System Parameters before this endpoint can be used. There is currently no rotation or expiry mechanism — it's a single static shared secret.

---

## CORS

All responses (including the 401/400/500 error responses and the `OPTIONS` preflight) include:

| Header | Value |
|---|---|
| `Access-Control-Allow-Origin` | `*` |
| `Access-Control-Allow-Methods` | `POST, OPTIONS` |
| `Access-Control-Allow-Headers` | `Authorization, Content-Type` |

This allows the endpoint to be called directly from browser-based JavaScript on any domain.

---

## Request

**URL:** `POST /api/create_lead`
**Content-Type:** `application/json`

### Body parameters

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | Yes | Lead/opportunity title |
| `email_from` | string | Yes | Contact email |
| `contact_name` | string | No | Name of the contact person |
| `phone` | string | No | Contact phone number |
| `website` | string | No | Website URL |
| `description` | string | No | Free-text notes |
| `tags` | string or list of strings | No | Defaults to `["yuvmedia"]` if omitted. Tags that don't exist in `crm.tag` are created automatically. |

### Example request

```bash
curl -X POST https://erp.yuvmedia.com/api/create_lead \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Website Inquiry - John Doe",
    "email_from": "john@example.com",
    "contact_name": "John Doe",
    "phone": "+91xxxxxxxxxx",
    "website": "https://example.com",
    "description": "Interested in ERP consulting",
    "tags": ["website", "consulting"]
  }'
```

---

## Responses

### 200 — Success
```json
{
  "status": 200,
  "message": "Lead created successfully",
  "data": {
    "lead_id": 123
  }
}
```

### 400 — Invalid JSON body
```json
{
  "status": 400,
  "message": "Invalid JSON body"
}
```

### 400 — Missing required field
```json
{
  "status": 400,
  "message": "Lead name is required"
}
```
```json
{
  "status": 400,
  "message": "Email is required"
}
```

### 401 — Unauthorized
```json
{
  "status": 401,
  "message": "Unauthorized"
}
```

### 500 — Server error
```json
{
  "status": 500,
  "message": "Error creating lead",
  "error": "<exception message>"
}
```

---

## Processing flow

1. If method is `OPTIONS`, return CORS headers immediately (preflight).
2. Validate the bearer token against `api.auth_token`. Reject with 401 if invalid/missing.
3. Parse the JSON body. Reject with 400 if body is missing/invalid.
4. Validate required fields (`name`, `email_from`). Reject with 400 if either is missing.
5. Resolve tags:
   - Use provided `tags` (string or list), or default to `["yuvmedia"]`.
   - For each tag name, look up `crm.tag` by exact name match; create it if it doesn't exist.
6. Create the `crm.lead` record (`type = 'lead'`) with the resolved values and tag IDs.
7. Return the new lead's ID in a 200 response.

---

## Known limitations / things to review

- **Single static token** — no per-client tokens, no expiry, no rate limiting. Anyone with the token has full create access to leads.
- **No email format validation** — `email_from` is only checked for presence, not validity.
- **Tag matching is case-sensitive exact match** — `"Website"` and `"website"` would create two separate tags.
- **`data.get('force=True, silent=True')` on `get_json`** — invalid JSON is silently swallowed and turned into a 400, which is fine, but any malformed-but-truthy body (e.g. a JSON array instead of object) would need testing to confirm `data.get(...)` calls don't raise before your own validation runs.
- **Broad `except Exception`** — returns the raw exception string in the response body (`error` field). Useful for debugging now, but consider redacting this in production to avoid leaking internal details (e.g. DB constraint messages) to external callers.
