# Odoo 19.0 — Custom Module Changes

**Project:** Custom Approval & Document Sign Enhancements
**Odoo Version:** 19.0
**Last Updated:** 2025-06-27

---

## Overview

This document describes the custom changes made to two core Odoo modules:

1. **Approvals** — Restricting edit and delete actions on approved requests.
2. **Sign (Document Sign)** — Adding "Send Later", notification, and reminder features for draft documents.

---

## Module 1: Approvals

### Change: Lock Approved Requests from Editing and Deletion

#### Description

Once an approval request reaches the **Approved** status, users are no longer permitted to edit any field on the request or delete the record. This prevents accidental or unauthorized modification of finalized approvals.

#### Affected Model

- `approval.request`

#### Behavior

| State         | Edit Allowed | Delete Allowed |
|---------------|:------------:|:--------------:|
| Draft         | ✅ Yes        | ✅ Yes          |
| Pending       | ✅ Yes        | ✅ Yes          |
| **Approved**  | ❌ No         | ❌ No           |
| Refused       | ✅ Yes        | ✅ Yes          |

#### Implementation Details

**Python Model Override (`approval_request.py`)**

The `write()` and `unlink()` methods are overridden to raise a `UserError` when the record is in the `approved` state.

```python
# models/approval_request.py

from odoo import models, _
from odoo.exceptions import UserError

class ApprovalRequest(models.Model):
    _inherit = 'approval.request'

    def write(self, vals):
        for record in self:
            if record.request_status == 'approved':
                raise UserError(_(
                    "You cannot edit an approval request that has already been approved."
                ))
        return super().write(vals)

    def unlink(self):
        for record in self:
            if record.request_status == 'approved':
                raise UserError(_(
                    "You cannot delete an approval request that has already been approved."
                ))
        return super().unlink()
```

**Form View Override (`approval_request_views.xml`)**

The form view is set to read-only when the request is in the approved state using `attrs` or `invisible`/`readonly` conditions.

```xml
<!-- views/approval_request_views.xml -->

<record id="approval_request_form_view_inherit" model="ir.ui.view">
    <field name="name">approval.request.form.inherit.lock</field>
    <field name="model">approval.request</field>
    <field name="inherit_id" ref="approvals.approval_request_view_form"/>
    <field name="arch" type="xml">
        <form position="attributes">
            <attribute name="edit">
                request_status != 'approved'
            </attribute>
        </form>
    </field>
</record>
```

> **Note:** The Delete button is automatically hidden by Odoo when `unlink()` raises a `UserError`. Additionally, access rules or `create_uid` checks can be layered on top for stricter security.

---

## Module 2: Sign (Document Sign)

### Change: Send Later, Notification, and Reminder for Draft Documents

#### Description

When a document's status is **Draft**, the user now has additional options available directly from the document form:

1. **Send Later** — Schedule the document to be sent at a future date and time.
2. **Send Notification** — Immediately notify relevant signatories or contacts that the document is ready or pending.
3. **Set Reminder** — Configure a reminder that will alert the user (or signatories) at a specified date/time if the document has not been signed.

#### Affected Model

- `sign.request`

#### Behavior by Status

| Feature           | Draft | Sent | Signed | Cancelled |
|-------------------|:-----:|:----:|:------:|:---------:|
| Send Later        | ✅    | ❌   | ❌     | ❌         |
| Send Notification | ✅    | ✅   | ❌     | ❌         |
| Set Reminder      | ✅    | ✅   | ❌     | ❌         |

---

### Feature 1: Send Later

Allows the user to pick a future datetime. At that datetime, an automated action (cron job) sends the document to the signatories automatically.

**New Fields on `sign.request`**

```python
# models/sign_request.py

from odoo import models, fields, _
from odoo.exceptions import UserError

class SignRequest(models.Model):
    _inherit = 'sign.request'

    send_later_date = fields.Datetime(
        string="Send Later Date",
        help="If set, the document will be sent automatically at this date and time."
    )
    is_scheduled = fields.Boolean(
        string="Scheduled",
        default=False,
        help="Indicates the document is queued for scheduled sending."
    )

    def action_send_later(self):
        """Mark the document for scheduled sending."""
        self.ensure_one()
        if self.state != 'shared':  # 'shared' = draft in sign module
            raise UserError(_("You can only schedule sending for draft documents."))
        if not self.send_later_date:
            raise UserError(_("Please set a 'Send Later Date' before scheduling."))
        self.is_scheduled = True
        return {
            'type': 'ir.actions.client',
            'tag': 'display_notification',
            'params': {
                'title': _("Scheduled"),
                'message': _("Document will be sent on %s.") % self.send_later_date,
                'type': 'success',
            }
        }
```

**Cron Job (`ir.cron`)**

```xml
<!-- data/sign_cron.xml -->

<record id="ir_cron_send_scheduled_documents" model="ir.cron">
    <field name="name">Sign: Send Scheduled Documents</field>
    <field name="model_id" ref="sign.model_sign_request"/>
    <field name="state">code</field>
    <field name="code">model._cron_send_scheduled()</field>
    <field name="interval_number">1</field>
    <field name="interval_type">hours</field>
    <field name="numbercall">-1</field>
    <field name="active">True</field>
</record>
```

---

### Feature 2: Send Notification

Sends an immediate in-app notification and/or email to the relevant signatories or followers when the document is in draft, informing them that the document is pending their attention.

**Method on `sign.request`**

```python
def action_send_notification(self):
    """Send an immediate notification to signatories."""
    self.ensure_one()
    partner_ids = self.mapped('request_item_ids.partner_id')
    if not partner_ids:
        raise UserError(_("No signatories found to notify."))

    self.message_post(
        body=_("You have been notified: the document '%s' is awaiting your signature.") % self.reference,
        partner_ids=partner_ids.ids,
        message_type='email',
        subtype_xmlid='mail.mt_comment',
    )
    return {
        'type': 'ir.actions.client',
        'tag': 'display_notification',
        'params': {
            'title': _("Notification Sent"),
            'message': _("Signatories have been notified."),
            'type': 'success',
        }
    }
```

---

### Feature 3: Set Reminder

Lets the user set a specific reminder date. An automated cron checks for overdue reminders and sends a follow-up notification to the user and/or signatories.

**New Fields on `sign.request`**

```python
reminder_date = fields.Datetime(
    string="Reminder Date",
    help="Send a reminder notification at this date if the document is still unsigned."
)
reminder_sent = fields.Boolean(
    string="Reminder Sent",
    default=False
)
```

**Reminder Cron**

```xml
<record id="ir_cron_sign_reminder" model="ir.cron">
    <field name="name">Sign: Send Draft Reminders</field>
    <field name="model_id" ref="sign.model_sign_request"/>
    <field name="state">code</field>
    <field name="code">model._cron_send_reminders()</field>
    <field name="interval_number">1</field>
    <field name="interval_type">hours</field>
    <field name="numbercall">-1</field>
    <field name="active">True</field>
</record>
```

**Cron Logic**

```python
def _cron_send_reminders(self):
    """Check and send reminders for draft sign requests."""
    now = fields.Datetime.now()
    pending = self.search([
        ('state', '=', 'shared'),
        ('reminder_date', '<=', now),
        ('reminder_sent', '=', False),
    ])
    for record in pending:
        record.action_send_notification()
        record.reminder_sent = True
```

---

### Form View Changes (`sign_request_views.xml`)

New buttons and fields are added to the Sign Request form, visible only when the document is in **Draft** status.

```xml
<record id="sign_request_form_view_inherit_draft_actions" model="ir.ui.view">
    <field name="name">sign.request.form.inherit.draft.actions</field>
    <field name="model">sign.request</field>
    <field name="inherit_id" ref="sign.sign_request_view_form"/>
    <field name="arch" type="xml">

        <!-- Add buttons in the header -->
        <xpath expr="//header" position="inside">
            <button name="action_send_later"
                    string="Send Later"
                    type="object"
                    class="btn-secondary"
                    invisible="state != 'shared'"/>
            <button name="action_send_notification"
                    string="Send Notification"
                    type="object"
                    class="btn-secondary"
                    invisible="state != 'shared'"/>
        </xpath>

        <!-- Add Send Later Date and Reminder Date fields in the form body -->
        <xpath expr="//sheet//group" position="after">
            <group string="Scheduling &amp; Reminders"
                   invisible="state != 'shared'">
                <field name="send_later_date" widget="datetime"/>
                <field name="reminder_date" widget="datetime"/>
                <field name="is_scheduled" readonly="1"/>
                <field name="reminder_sent" readonly="1"/>
            </group>
        </xpath>

    </field>
</record>
```

---

## Module Structure

```
custom_approval_sign/
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── approval_request.py      # Edit/delete lock on approved requests
│   └── sign_request.py          # Send later, notification, reminder logic
├── views/
│   ├── approval_request_views.xml
│   └── sign_request_views.xml
├── data/
│   └── sign_cron.xml            # Cron jobs for scheduled send & reminders
└── security/
    └── ir.model.access.csv
```

---

## Testing Checklist

### Approvals

- [ ] Create an approval request and submit it.
- [ ] Approve the request.
- [ ] Attempt to edit any field — confirm `UserError` is raised.
- [ ] Attempt to delete the record — confirm `UserError` is raised.
- [ ] Verify refused/pending requests can still be edited and deleted.

### Sign — Draft Actions

- [ ] Open a sign request in Draft state.
- [ ] Confirm "Send Later", "Send Notification" buttons are visible.
- [ ] Set a `Send Later Date` and click **Send Later** — confirm scheduled flag is set.
- [ ] Click **Send Notification** — confirm signatories receive a notification.
- [ ] Set a `Reminder Date` in the past and run the cron manually — confirm reminder is sent and `reminder_sent` is set to `True`.
- [ ] Confirm none of the above buttons appear on Signed or Cancelled documents.

---

## Notes

- The `state` field in the Sign module uses the value `'shared'` internally for the Draft state. Verify this against your specific Odoo 19.0 build, as it may differ from community to enterprise editions.
- Ensure the cron jobs are activated in **Settings → Technical → Automation → Scheduled Actions** after installation.
- Access rights should be reviewed so that only authorized users (e.g., managers) can use the "Send Later" and "Set Reminder" features if required.
