# Odoo–Facebook Lead Ads Integration (Meta Graph API)

**Purpose:** Automatically sync leads submitted through Facebook/Instagram Lead Ads forms into Odoo CRM as `crm.lead` records.

> **Note:** This document is a structured template based on the standard architecture for this type of integration. Fill in the placeholders (marked `<<...>>`) with your actual App credentials, module name, code, and field mappings to make this fully accurate to your implementation.

---

## 1. Overview

| Item | Detail |
|---|---|
| Integration type | Facebook/Instagram Lead Ads → Odoo CRM |
| Meta API used | Graph API `<<version, e.g. v19.0>>` |
| Sync method | `<<Webhook (real-time) / Scheduled polling>>` |
| Odoo module | `<<module name, e.g. fb_leadads_sync>>` |
| Odoo version | 19.0 Community |
| Target model | `crm.lead` |

---

## 2. Meta (Facebook) Side Setup

1. Create/verify a **Meta App** in [Meta for Developers](https://developers.facebook.com/apps).
2. Add the **Leads Retrieval** and **Webhooks** products to the app.
3. Generate credentials:
   - App ID: `<<App ID>>`
   - App Secret: `<<App Secret>>`
   - Page Access Token: `<<Page Access Token>>` (long-lived, via System User in Business Manager recommended)
4. Subscribe the Page to the `leadgen` webhook field.
5. Configure the Webhook callback URL and Verify Token pointing to the Odoo endpoint (see §4).
6. Confirm required permissions are granted: `leads_retrieval`, `pages_manage_ads`, `pages_show_list`.

---

## 3. Odoo Module Structure

```
<<module_name>>/
├── __init__.py
├── __manifest__.py
├── controllers/
│   ├── __init__.py
│   └── main.py          # Webhook endpoint (verification + lead receipt)
├── models/
│   ├── __init__.py
│   └── crm_lead.py       # Field mapping / lead creation logic
└── data/
    └── ir_config_parameter_data.xml   # Stores App ID/Secret/Token as system params
```

---

## 4. Webhook Controller (Verification + Receiver)

Odoo exposes a public HTTP route to handle Meta's webhook handshake (`GET`) and lead events (`POST`).

```python
from odoo import http
from odoo.http import request
import requests
import json

class FacebookLeadWebhook(http.Controller):

    @http.route('/facebook/leads/webhook', type='http', auth='public', csrf=False, methods=['GET'])
    def verify(self, **kwargs):
        verify_token = request.env['ir.config_parameter'].sudo().get_param('fb.verify_token')
        if kwargs.get('hub.verify_token') == verify_token:
            return kwargs.get('hub.challenge')
        return 'Invalid verify token', 403

    @http.route('/facebook/leads/webhook', type='json', auth='public', csrf=False, methods=['POST'])
    def receive_lead(self, **kwargs):
        data = json.loads(request.httprequest.data)
        for entry in data.get('entry', []):
            for change in entry.get('changes', []):
                leadgen_id = change.get('value', {}).get('leadgen_id')
                if leadgen_id:
                    request.env['crm.lead'].sudo()._fetch_and_create_from_leadgen(leadgen_id)
        return {'status': 'ok'}
```

---

## 5. Fetching Lead Details from Graph API

Once notified of a `leadgen_id`, Odoo calls the Graph API to fetch the full lead payload.

```python
def _fetch_and_create_from_leadgen(self, leadgen_id):
    access_token = self.env['ir.config_parameter'].sudo().get_param('fb.page_access_token')
    url = f"https://graph.facebook.com/<<version>>/{leadgen_id}"
    params = {'access_token': access_token}
    response = requests.get(url, params=params)
    lead_data = response.json()
    field_data = {f['name']: f['values'][0] for f in lead_data.get('field_data', [])}

    self.create({
        'name': field_data.get('full_name', 'Facebook Lead'),
        'contact_name': field_data.get('full_name'),
        'email_from': field_data.get('email'),
        'phone': field_data.get('phone_number'),
        'source_id': self.env.ref('<<utm_source_facebook_xml_id>>').id,
        'medium_id': self.env.ref('<<utm_medium_social_xml_id>>').id,
    })
```

---

## 6. Field Mapping

| Facebook Lead Field | Odoo `crm.lead` Field |
|---|---|
| `full_name` | `contact_name` / `name` |
| `email` | `email_from` |
| `phone_number` | `phone` |
| `<<custom form field>>` | `<<custom crm.lead field>>` |

---

## 7. Configuration Parameters (Odoo `ir.config_parameter`)

| Key | Value |
|---|---|
| `fb.app_id` | `<<App ID>>` |
| `fb.app_secret` | `<<App Secret>>` |
| `fb.page_access_token` | `<<Page Access Token>>` |
| `fb.verify_token` | `<<Custom string used in webhook handshake>>` |

---

## 8. Testing

1. Use Meta's **Lead Ads Testing Tool** to submit a test lead.
2. Confirm the webhook `GET` verification succeeds (check Meta App dashboard → Webhooks).
3. Submit a test lead and confirm a `crm.lead` record is created in Odoo.
4. Check Odoo server logs for Graph API response errors (expired token, missing permissions).

---

## 9. Known Issues / Gotchas

- `<<e.g. Page Access Tokens expire — use a System User token from Business Manager for a long-lived/non-expiring token>>`
- `<<e.g. Webhook must respond within a few seconds or Meta will retry/disable it>>`
- `<<e.g. leadgen_id fetch requires the page token, not the app token>>`

---

## 10. Next Steps

- `<<e.g. Add deduplication check against existing crm.lead by email/phone>>`
- `<<e.g. Add retry queue for failed Graph API fetches>>`
