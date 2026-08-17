# Meta Lead Connector - Feature List

## Facebook Integration

- Facebook Instance management: User Access Token, connected status, scheduler toggle, and log retention policy (1-12 months)
- Facebook Page sync via `GET /me/accounts` - creates/updates pages with Page ID and Page Access Token
- Lead Form sync via `GET /{page_id}/leadgen_forms` with pagination - creates campaigns and lead form records
- Bulk campaign/lead-form sync across all connected pages

## Field Mapping

- Field import from form questions via `GET /{form_id}?fields=questions`
- Auto field mapping: phone -> phone, email -> email_from, full_name -> contact_name, website, company -> partner_name, custom fields -> description
- Manual per-form field mapping with Odoo `ir.model.fields` selection
- Global Facebook Mapper model for reusable FB-to-CRM field mapping

## Lead Fetching

- Lead fetching via `GET /{form_id}/leads` with `paging.next` pagination
- Single-day lead filtering based on `last_sync_date`
- Deduplication by `fb_lead_id` and normalized phone number
- CRM lead creation with mapped fields, fallback default mapping, and full description
- Lead stats tracking: `leads_count` and `last_fetch_summary`

## CRM Integration

- CRM lead extension: `fb_lead_id`, `fb_form_id`, `fb_campaign_id`, `raw_data`
- Dedicated Facebook Leads view and Facebook Lead Info tab
- UTM source tracking (source = Facebook)

## Logging & UX

- Facebook Logger audit trail for all operations (info/warning/error)
- Friendly Facebook API error handling (codes #10, #190, etc.)
- Facebook Integration menu: Instances, Pages, Forms, Logger, Mapper

---

**Note:** Leads are fetched manually via buttons; there are no webhooks or automatic schedulers active. Deployment is via Docker Compose with Odoo 19 and PostgreSQL.
