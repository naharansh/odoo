# Ym_ServerManagement

Custom Odoo 19 module for **Ym Server Management** — it centralizes the management of **server/domain credentials and subscriptions for clients**.

The module lets a company (Yuvmedia) track each client's domains and servers, link them to packages (with validity), attach control panels (WHM / cPanel), monitor expiry dates, and automatically notify both clients and administrators before, on, and after expiration.

---

## Table of Contents

- [Features](#features)
- [Technical Details](#technical-details)
- [Models](#models)
  - [ym.server.management](#ymservermanagement--domain-users)
  - [ym.server.servers](#ymserverservers--domains)
  - [ym.server.package](#ymserverpackage--packages)
  - [ym.server.whm](#ymserverwhm--control-panels)
  - [ym.server.cpanel](#ymservercpanel--cpanels)
  - [ym.server.tags](#ymservertags--website-type-tags)
- [Relationships](#relationships)
- [Automation & Notifications](#automation--notifications)
- [Cron Jobs](#cron-jobs)
- [Security & Access Rights](#security--access-rights)
- [Menus](#menus)
- [Installation](#installation)
- [Usage](#usage)
- [Project Structure](#project-structure)

---

## Features

- **Client (User) records** linked to Odoo `res.partner` contacts, with auto-filled client name, business name, email and phone.
- **Domain/Server records** holding the domain/URL, website type tags, AMC/Backup flags, domain holder, remarks and links to control panels.
- **Package management** with a validity (in months) that drives automatic expiry computation.
- **Automatic lifecycle statuses**: `Draft → Active → Expiring → Expired`.
- **Automatic expiry emails**:
  - 20-day renewal notice to the client.
  - 7-day urgent warning to the client.
  - Expired alert to the client + **ATTENTION ADMINISTRATOR** notification.
- **Systray activities** created for admins for expiring and expired servers.
- **Daily cron** that keeps server statuses in sync and triggers alerts.
- **Server ⇄ User state synchronization** — a user is automatically deactivated when all of their servers are expired.
- **Email notifications** via `mail.thread` / `mail.activity.mixin` chatter integration.
- **Legacy data migration** for the old `server_id` → many2many `website_type` relation.

---

## Technical Details

| Item | Value |
|---|---|
| Module name | `ym__server_management` |
| Odoo version | 19.0 |
| License / author | My Company |
| Category | Sales |
| Dependencies | `base`, `mail` |
| Main menu | Server Management |
| Access group | `group_server_management_manager` (Administrator) |

---

## Models

### `ym.server.management` — Domain Users

Represents a **client / user** who owns one or more servers.

| Field | Type | Notes |
|---|---|---|
| `sequence` | Char | Auto-generated ID, prefix `US/%(year)s/`, readonly |
| `user_id` | Many2one `res.users` | Assigned internal user, defaults to current user |
| `partner_id` | Many2one `res.partner` | Client contact (must not be a company) |
| `client_name` | Char | Related to `partner_id.name`, stored |
| `business_name` | Char | Related to `partner_id.commercial_partner_id.name`, stored |
| `email` | Char | Related to `partner_id.email`, stored |
| `phone` | Char | Related to `partner_id.phone`, stored |
| `description` | Html | Notes |
| `server_ids` | Many2many `ym.server.servers` | Servers owned by this user |
| `state` | Selection | `draft`, `active`, `deactivated` |
| `active` | Boolean | Archive flag |

Key behavior (`models/server.py`):

- New records default to `state = "active"`.
- `_compute_display_name` renders `"<sequence> - <client_name>"`.
- Deleting a record whose state is `active` raises a `UserError`.

### `ym.server.servers` — Domains

Represents a **single domain/server subscription** owned by a user.

| Field | Type | Notes |
|---|---|---|
| `sequence` | Char | Auto-generated ID, prefix `USR/%(year)s/`, readonly |
| `website_type` | Many2many `ym.server.tags` | Categorizes the domain type |
| `domain_url` | Char | Domain/URL (required) |
| `amc` | Boolean | AMC flag |
| `backup` | Boolean | Backup flag |
| `domain_holder` | Selection | `business_owner`, `yuvmedia` |
| `remarks` | Html | Free remarks |
| `whm_id` | Many2one `ym.server.whm` | Control panel (WHM) |
| `server_user_id` | Many2one `ym.server.management` | Owning user (tracked) |
| `cpanel_ids` | One2many | Related to `whm_id.cpanel_ids`, readonly |
| `package_id` | Many2one `ym.server.package` | Main package |
| `package_ids` | Many2many `ym.server.package` | Additional packages |
| `package_description` | Html | Related to `package_id.description` |
| `start_date` | Date | Subscription start |
| `validity` | Integer | Months (computed from packages, default 12) |
| `expiry_date` | Date | Computed = `start_date + validity` − 1 day |
| `days_left` | Integer | Computed, min 0 |
| `state` | Selection | `draft`, `active`, `expiring`, `expired` |
| `active` | Boolean | Archive flag |
| `expiry_email_sent_20d` / `_7d` / `expiry_email_sent` | Boolean | Prevent duplicate emails |

Key behavior (`models/servers.py`):

- `_compute_validity`: sums `package_ids.validity`; falls back to `package_id.validity` or 12.
- `_compute_expiry_date`: `start_date + relativedelta(months=validity) - timedelta(days=1)`.
- State is derived from dates (`_compute_state_from_dates`): within 30 days → `expiring`; on/past expiry → `expired`.
- `_sync_user_state` deactivates users whose servers are all expired.
- Manual state buttons: `action_confirm`, `action_expiring`, `action_expired`, `action_draft`.

### `ym.server.package` — Packages

Predefined hosting packages with a validity in months.

| Field | Type | Notes |
|---|---|---|
| `name` | Char | Package name (required) |
| `validity` | Integer | Months (default 12, required) |
| `description` | Html | Description |
| `active` | Boolean | Archive flag |

`_cron_update_package_stages` forwards to the server-stage updater (legacy cron bridge).

### `ym.server.whm` — Control Panels

| Field | Type | Notes |
|---|---|---|
| `name` | Char | WHM name (required) |
| `cpanel_ids` | One2many `ym.server.cpanel` | cPanels under this WHM |

### `ym.server.cpanel` — Cpanels

| Field | Type | Notes |
|---|---|---|
| `name` | Char | cPanel name (required) |
| `color` | Integer | Random color for the tag widget |
| `whm_id` | Many2one `ym.server.whm` | Parent WHM (cascade delete) |

### `ym.server.tags` — Website Type Tags

| Field | Type | Notes |
|---|---|---|
| `name` | Char | Website type (required) |
| `color` | Integer | Tag color |
| `server_ids` | Many2many `ym.server.servers` | Servers using this tag |

`init()` migrates legacy `server_id` values from `ym_server_tags` into the m2m relation table (`ym_server_servers_tags_rel`).

---

## Relationships

```
ym.server.management (Domain User)
        │  1
        │  many2many server_ids
        ▼  many
ym.server.servers (Domain/Server)
        │  1 whm_id
        ▼  many
ym.server.whm (Control Panel)
        │  1
        │  one2many cpanel_ids
        ▼  many
ym.server.cpanel (cPanel)
        ▲
ym.server.servers ── package_id / package_ids ──► ym.server.package
ym.server.servers ── website_type (m2m) ─────────► ym.server.tags
```

---

## Automation & Notifications

All alerts are orchestrated by `_check_and_send_alerts()` in `models/servers.py`:

| Trigger | Action |
|---|---|
| 0 < days ≤ 20 | Email template `email_template_server_expiry_20d` → client |
| 0 < days ≤ 7 | Email template `email_template_server_expiry_7d` → client |
| state = expiring (≤30 days) | Create `Domain/Server Expiring` activity for every admin |
| state = expired | In-app notification to admins + expired email to client + `Domain/Server Expired` activity |

Notification building blocks defined in `data/cron_action.xml`:

- `mt_server_expired`, `mt_server_expiring` — mail.message.subtype
- `mail_activity_type_server_expiring`, `mail_activity_type_server_expired` — activity types
- `email_template_server_expiry_20d`, `email_template_server_expiry_7d`, `email_template_server_expired` — HTML email templates (branded, responsive)

---

## Cron Jobs

| Cron | Interval | Code | Purpose |
|---|---|---|---|
| Server Management: Update Server Stages | 1 day | `model._cron_update_server_stages()` | Recomputes state from expiry date, posts chatter notes on changes, runs alerts and user-state sync |

`data/cleanup.xml` removes the obsolete legacy cron `Update Package Stages` on install.

---

## Security & Access Rights

Defined in `security/security_groups.xml` and `security/ir.model.access.csv`:

- **Category**: `Server Management` (`category_server_management`)
- **Privilege**: `privilege_server_management` (Odoo 19 Access Rights tab)
- **Group**: `group_server_management_manager` (Administrator) — full CRUD on all module models, record rule sees all records.

Model access (all granted to `group_server_management_manager`):

| Model | read | write | create | unlink |
|---|---|---|---|---|
| `ym.server.management` | ✅ | ✅ | ✅ | ✅ |
| `ym.server.package` | ✅ | ✅ | ✅ | ✅ |
| `ym.server.servers` | ✅ | ✅ | ✅ | ✅ |
| `ym.server.tags` | ✅ | ✅ | ✅ | ✅ |
| `ym.server.whm` | ✅ | ✅ | ✅ | ✅ |
| `ym.server.cpanel` | ✅ | ✅ | ✅ | ✅ |

---

## Menus

```
Server Management
├── Domains          → ym.server.servers
├── Domain Users     → ym.server.management
└── Configuration
    ├── Packages        → ym.server.package
    ├── Control Panels  → ym.server.whm
    └── Tags            → ym.server.tags
```

---

## Installation

The project is set up for **Odoo 19 via Docker Compose**:

```yaml
# docker-compose.yml (project root)
odoo:
  image: odoo:19.0
  volumes:
    - ./extra-addons:/mnt/extra-addons
    - ./config:/etc/odoo
```

1. Place the module under `extra-addons/ym__server_management`.
2. `docker compose up -d odoo`
3. Open Odoo → **Apps** → Update Apps List.
4. Install **Ym_ServerManagement**.
5. Under *Settings → Users → Access Rights*, assign the **Server Management → Administrator** privilege to the relevant users.

---

## Usage

1. **Configuration** (one-time):
   - Create **Packages** (name + validity in months).
   - Create **Control Panels** (WHM) and add its **cPanels**.
   - Create **Website Type Tags** (e.g. WordPress, Laravel, Custom).

2. **Domain Users**: create a user (select a contact) — client name, business name, email and phone are pulled automatically.

3. **Domains**: create a server record with domain/URL, select the owning **Server User**, **Package**, **WHM/cPanels**, **start date** and **website type**. Validity, expiry date and days left are computed automatically, and the status moves `draft → active` (or straight to the correct state based on the dates).

4. **Monitoring**: the daily cron keeps statuses fresh. Clients receive 20-day and 7-day renewal emails, admins get systray activities, and expired domains trigger an in-app admin alert plus a client email.

---

## Project Structure

```
extra-addons/ym__server_management/
├── __init__.py
├── __manifest__.py
├── controllers/
│   └── controllers.py          # (reserved — currently commented out)
├── data/
│   ├── cleanup.xml             # removes legacy cron
│   └── cron_action.xml         # cron, subtypes, activities, email templates
├── demo/
│   └── demo.xml                # demo data (commented out)
├── models/
│   ├── __init__.py
│   ├── models.py               # (template, commented out)
│   ├── server.py               # ym.server.management
│   ├── servers.py              # ym.server.servers + alerts logic
│   ├── package.py              # ym.server.package
│   ├── whm.py                  # ym.server.whm
│   ├── cpanel.py               # ym.server.cpanel
│   └── tags.py                 # ym.server.tags
├── security/
│   ├── ir.model.access.csv
│   └── security_groups.xml
├── static/description/icon.png
└── views/
    ├── views.xml               # menus, actions, sequences, list/form/search views
    └── templates.xml
```
