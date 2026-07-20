# Odoo Development Spec — Facebook Sync Module + Timesheet & Document Sign Updates

**Target:** Odoo 19.0 (Community Edition), erp.yuvmedia.com / testerp.yuvmedia.com
**Prepared for:** Yuvmedia ERP development

---

## 1. New Module: `yuv_facebook_sync`

### 1.1 Purpose

A custom module that connects the Odoo instance to a Facebook Page (via the Facebook Graph API) to:

- Sync Facebook Page posts into Odoo (read-only mirror, optionally allow posting from Odoo)
- Pull Lead Ads / Messenger leads into Odoo `crm.lead`
- Sync comments/messages on posts for social monitoring
- Log sync activity and API errors
- Run sync automatically via scheduled cron jobs, with a manual "Sync Now" button

### 1.2 Module Structure

```
yuv_facebook_sync/
├── __init__.py
├── __manifest__.py
├── controllers/
│   ├── __init__.py
│   └── webhook_controller.py        # Facebook webhook receiver (real-time updates)
├── models/
│   ├── __init__.py
│   ├── facebook_account.py          # Page/App connection config
│   ├── facebook_post.py             # Synced posts
│   ├── facebook_comment.py          # Synced comments
│   ├── facebook_lead.py             # Lead Ads → crm.lead bridge
│   ├── facebook_sync_log.py         # Sync run history / error log
│   └── res_config_settings.py       # Global settings extension
├── services/
│   ├── __init__.py
│   └── graph_api_client.py          # Thin wrapper around Facebook Graph API calls
├── data/
│   └── ir_cron_data.xml             # Scheduled sync jobs
├── security/
│   ├── ir.model.access.csv
│   └── facebook_sync_security.xml   # Record rules / groups
├── views/
│   ├── facebook_account_views.xml
│   ├── facebook_post_views.xml
│   ├── facebook_comment_views.xml
│   ├── facebook_lead_views.xml
│   ├── facebook_sync_log_views.xml
│   ├── res_config_settings_views.xml
│   └── menu_views.xml
├── static/
│   └── description/
│       └── icon.png
└── README.md
```

### 1.3 `__manifest__.py`

```python
{
    "name": "Yuvmedia Facebook Sync",
    "version": "19.0.1.0.0",
    "category": "Marketing",
    "summary": "Sync Facebook Page posts, comments, and Lead Ads into Odoo",
    "author": "Yuvmedia",
    "depends": ["base", "mail", "crm"],
    "data": [
        "security/ir.model.access.csv",
        "security/facebook_sync_security.xml",
        "data/ir_cron_data.xml",
        "views/facebook_account_views.xml",
        "views/facebook_post_views.xml",
        "views/facebook_comment_views.xml",
        "views/facebook_lead_views.xml",
        "views/facebook_sync_log_views.xml",
        "views/res_config_settings_views.xml",
        "views/menu_views.xml",
    ],
    "installable": True,
    "application": True,
    "license": "LGPL-3",
}
```

### 1.4 Models

#### `facebook.account` (Page connection)
| Field | Type | Notes |
|---|---|---|
| `name` | Char | Page display name |
| `page_id` | Char | Facebook Page ID |
| `page_access_token` | Char | Long-lived Page token (store encrypted; restrict field to admins) |
| `app_id` / `app_secret` | Char | Facebook App credentials |
| `webhook_verify_token` | Char | Token used to verify webhook subscription |
| `active` | Boolean | |
| `last_sync_date` | Datetime | |
| `sync_posts` / `sync_leads` / `sync_comments` | Boolean | Toggle what to sync |

Methods:
- `action_test_connection()` — calls Graph API `/me` to validate token
- `action_sync_now()` — manual trigger, calls the three sync methods below
- `_sync_posts()`, `_sync_leads()`, `_sync_comments()` — each writes/updates child records and a `facebook.sync.log` entry

#### `facebook.post`
- `facebook_post_id` (Char, unique), `account_id` (Many2one), `message` (Text), `permalink_url` (Char), `created_time` (Datetime), `likes_count`/`comments_count`/`shares_count` (Integer), `raw_json` (Text, for debugging)

#### `facebook.comment`
- `facebook_comment_id` (Char, unique), `post_id` (Many2one → facebook.post), `from_name` (Char), `message` (Text), `created_time` (Datetime)

#### `facebook.lead`
- Fields mirroring Facebook Lead Ads form data, plus `crm_lead_id` (Many2one → crm.lead, created automatically), `campaign_name`, `form_name`, `raw_field_data` (Text/JSON)
- On creation: `_create_crm_lead()` maps standard fields (name, email, phone) into a new `crm.lead`

#### `facebook.sync.log`
- `sync_type` (Selection: posts/leads/comments/webhook), `status` (Selection: success/error), `message` (Text), `record_count` (Integer), `create_date`

### 1.5 Graph API Client (`services/graph_api_client.py`)

Wraps `requests` calls to `https://graph.facebook.com/v19.0/...`:
- `get_page_posts(page_id, token)`
- `get_post_comments(post_id, token)`
- `get_lead(leadgen_id, token)`
- Centralized error handling — raises a custom `FacebookAPIError` that the calling model catches and logs to `facebook.sync.log` instead of crashing the cron.

### 1.6 Webhook Controller (real-time updates)

`controllers/webhook_controller.py` exposes:
- `GET /facebook/webhook` — verification handshake (`hub.challenge` response)
- `POST /facebook/webhook` — receives real-time events (new lead, new comment) and calls the matching model method to fetch/store the new object immediately, rather than waiting for the cron

Both routes are `auth="public"`, `csrf=False`, and validate the `X-Hub-Signature-256` header against `app_secret` before processing.

### 1.7 Scheduled Jobs (`data/ir_cron_data.xml`)

Three `ir.cron` records (e.g., every 30 minutes) calling `facebook.account` sync methods for all active accounts — acts as a fallback/reconciliation pass in addition to the webhook.

### 1.8 Security

- New group: `Facebook Sync / Manager` (full access, can view/edit tokens) and `Facebook Sync / User` (read-only on posts/comments/leads)
- `ir.model.access.csv` grants CRUD per group
- `page_access_token` and `app_secret` fields marked `groups="yuv_facebook_sync.group_facebook_sync_manager"` in the form view so regular users never see them

### 1.9 Config Settings

Extend `res.config.settings` with a "Facebook Sync" section under **Settings → General Settings** for quick access to add a Page connection without navigating to the dedicated menu.

### 1.10 Open Items to Confirm Before Build

- Which Facebook Page(s) will be connected, and do you already have a Meta App ID/Secret + long-lived Page token?
- Should synced posts be one-way (Facebook → Odoo) only, or do you also want to publish posts from Odoo to Facebook?
- Should Lead Ads always auto-create a `crm.lead`, or should that be a manual "Convert" button?
- Preferred sync interval for the cron fallback (15 min / 30 min / hourly)?

---

## 2. Timesheet Change — Lock Form While Timer Is Running

**Module affected:** `task_module` (existing custom module)

### 2.1 Requirement

While the timer is active on a timesheet/task entry, the entire form should become **readonly** (not just the timer-related fields) so the user cannot edit other fields mid-timer. Fields unlock again once the timer is stopped.

### 2.2 Approach

1. Add/confirm a boolean field that reflects timer state, e.g. `is_timer_running` (Boolean, already likely present given the existing timer feature — reuse it instead of adding a duplicate field).
2. In the timesheet form view (`account.analytic.line` inherited view, or the custom timesheet view used by `task_module`):
   - Wrap the fields (or the whole `<sheet>`/`<group>` block) with `attrs="{'readonly': [('is_timer_running', '=', True)]}"` (Odoo ≤17 syntax) or, for Odoo 19's new expression syntax:
     ```xml
     <sheet readonly="is_timer_running">
         ...
     </sheet>
     ```
   - Odoo 19 supports `readonly="<python expression>"` directly on the `<form>`/`<sheet>`/`<group>` tag, which cascades to all contained fields — this is the cleanest way to lock the whole form rather than tagging every field individually.
3. Keep the **Stop Timer** button itself always clickable — it must NOT inherit the readonly state, or the user could never stop the timer. Buttons are unaffected by field-level `readonly`, but if the button is wrapped in a readonly group, explicitly move it outside that group or override with `invisible`/`readonly="0"` on the button.
4. Server-side safety net: add a `write()` override on the model that raises a `UserError` if `is_timer_running` is `True` and the incoming `vals` touch any field other than the whitelisted timer fields (`is_timer_running`, `timer_start`, `unit_amount`, etc.). This prevents edits via API/import even if a user bypasses the UI.

### 2.3 Example (Odoo 19 view syntax)

```xml
<record id="view_task_timesheet_form_readonly_on_timer" model="ir.ui.view">
    <field name="name">task.timesheet.form.readonly.timer</field>
    <field name="model">account.analytic.line</field>
    <field name="inherit_id" ref="hr_timesheet.hr_timesheet_line_form"/>
    <field name="arch" type="xml">
        <sheet position="attributes">
            <attribute name="readonly">is_timer_running</attribute>
        </sheet>
        <xpath expr="//button[@name='action_stop_timer']" position="attributes">
            <attribute name="readonly">0</attribute>
        </xpath>
    </field>
</record>
```

*(Field/method names above — `action_stop_timer`, `is_timer_running` — should be confirmed against your actual `task_module` implementation before applying.)*

---

## 3. `document_sign` Module Change — Post-Signature Confirmation Email

**Module affected:** custom `document_sign` module (previously extended with geolocation + IP logging)

### 3.1 Requirement

After a document is fully signed, send a confirmation email to:
1. The **signer** (the person who signed)
2. The **sender** (the person/user who originally sent the document for signature)

### 3.2 Approach

1. Identify the method currently called when the last required signature is completed (likely something like `action_complete_signature()` or a state change to `state = 'signed'`). Hook the email logic there.
2. Create two email templates (`mail.template`) via XML data:
   - `mail_template_signer_confirmation` — "Thank you, your signature has been recorded" — sent to `signer_email`/`partner_id` on the signature request.
   - `mail_template_sender_confirmation` — "Document fully signed by [signer name]" — sent to `create_uid`/`sent_by` (whoever initiated the request).
3. In the Python method that finalizes signing:
   ```python
   def action_complete_signature(self):
       for rec in self:
           rec.write({'state': 'signed', 'signed_date': fields.Datetime.now()})
           template_signer = self.env.ref('document_sign.mail_template_signer_confirmation')
           template_sender = self.env.ref('document_sign.mail_template_sender_confirmation')
           template_signer.send_mail(rec.id, force_send=True)
           template_sender.send_mail(rec.id, force_send=True)
       return True
   ```
4. Make sure the templates pull the correct recipient dynamically:
   - Signer template: `email_to` = `${object.signer_email}` (or the partner linked to the signature request line, if multiple signers exist on one document — loop per signer rather than per document if so).
   - Sender template: `email_to` = `${object.create_uid.email}` or a dedicated `sent_by_partner_id` field if one already exists from the earlier geolocation/IP-logging update.
5. Attach the signed PDF (and optionally the signing access log / audit trail already built) to at least the sender's confirmation email using `report_template` or a static attachment on the template.
6. If the document supports **multiple signers**, decide: does the sender get one email per signer, or a single email once *all* signers are done? Recommend the latter (one summary email to sender when the last signature lands) to avoid inbox spam — confirm this before building.

### 3.3 Open Items to Confirm

- Exact model/method name where "fully signed" state is set in your current `document_sign` module.
- Whether documents can have multiple signers, and if so, sender email timing (per-signature vs. all-complete).
- Whether the signed PDF should be attached to these emails or just linked (given file size / mail server limits).

---

## 4. Suggested Build Order

1. Timesheet readonly-on-timer (smallest change, quick win)
2. `document_sign` confirmation emails (medium — depends on confirming current model/method names)
3. `yuv_facebook_sync` module (largest — needs Meta App credentials and scope decisions before development starts)
