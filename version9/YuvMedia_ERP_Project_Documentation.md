# Project Documentation: YuvMedia ERP Customizations

**System:** erp.yuvmedia.com
**Platform:** Odoo Community Edition
**Document Version:** 1.0
**Last Updated:** 20 June 2026
**Prepared By:** [Your Name]

---

## 1. Project Overview

This document covers two customizations made to the YuvMedia Odoo Community Edition instance:

1. A modification to the **All Timesheets** list view to improve usability/reporting.
2. A new **custom module — `YmApprovals`** — which automates approval workflows and generates a PDF document containing request details once an approver approves it.

---

## 2. Environment Details

| Item | Detail |
|---|---|
| ERP URL | erp.yuvmedia.com |
| Odoo Edition | Community Edition |
| Odoo Version | [e.g. 17.0 — please confirm] |
| Affected App(s) | Timesheets, Custom (YmApprovals) |
| Deployment Type | [Self-hosted / Odoo.sh / Other] |
| Environment | [Production / Staging] |

---

## 3. Modification: "All Timesheets" List View

### 3.1 Objective
[Briefly describe why the list view was changed — e.g. "to surface project/task/employee details more clearly for management review."]

### 3.2 Changes Made
- **Fields Added/Removed:** [List the fields added or removed from the list view]
- **Filters / Group By:** [Any new filters or default groupings added]
- **Sorting:** [Default sort order, if changed]
- **Access Rules / Visibility:** [Any changes to who can see which records]
- **View Type Affected:** `tree`/`list` view of `hr.timesheet` (or relevant model)

### 3.3 Technical Implementation
- **Module Modified:** [Name of module — core `hr_timesheet` via inheritance, or a custom module]
- **View XML ID Inherited:** [e.g. `hr_timesheet.timesheet_view_tree`]
- **Method:** XML view inheritance using `<xpath>` / `position="attributes"` (or similar)

```xml
<!-- Example placeholder snippet -->
<record id="view_timesheet_tree_custom" model="ir.ui.view">
    <field name="name">timesheet.tree.custom</field>
    <field name="model">account.analytic.line</field>
    <field name="inherit_id" ref="hr_timesheet.hr_timesheet_line_tree"/>
    <field name="arch" type="xml">
        <xpath expr="//field[@name='...']" position="after">
            <field name="..."/>
        </xpath>
    </field>
</record>
```

### 3.4 Impact
- [Who is affected — employees, managers, HR, accounting?]
- [Any reporting changes downstream]

---

## 4. Custom Module: `YmApprovals`

### 4.1 Purpose
A custom Odoo module built to manage approval requests. When a designated approver approves a request, the system automatically generates a PDF document containing all relevant details of that request.

### 4.2 Key Features
- Custom approval request model/workflow
- Approver assignment (single or multi-level — [confirm which])
- Approve / Reject actions with status tracking
- **Auto-generated PDF report** triggered on approval, containing full request details
- [Email notification on approval? Yes/No]
- [Attachment stored against the record? Yes/No]

### 4.3 Module Structure

```
ym_approvals/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── ym_approval_request.py
├── views/
│   ├── ym_approval_views.xml
│   └── ym_approval_menus.xml
├── report/
│   ├── ym_approval_report.xml
│   └── ym_approval_report_templates.xml
├── security/
│   ├── ir.model.access.csv
│   └── ym_approval_security.xml
└── static/
    └── description/
        └── icon.png
```
[Adjust the above to match your actual file/folder structure.]

### 4.4 Approval Workflow

1. **Request Creation** — [Who creates the request, and from where (e.g. a form, a button on another model)?]
2. **Submission** — Request moves to "Pending Approval" / "To Approve" state.
3. **Approver Review** — Approver(s) see the request in their queue ([list view / activity / inbox]).
4. **Approval Action** — Approver clicks "Approve."
5. **PDF Generation (Automated)** — On approval:
   - The system triggers a QWeb report action.
   - A PDF is generated using the `ym_approval_report` template.
   - PDF contains: [list exact fields — e.g. requester name, date, department, approval details, approver name & timestamp, request type, amount/hours, remarks, etc.]
   - PDF is [attached to the record / emailed to requester / both].
6. **Rejection Path** — [What happens if rejected? Any notification/reason capture?]

### 4.5 Technical Implementation — PDF Generation

- **Trigger:** Python method on the `action_approve()` (or equivalent) function of the model, calling the report action.
- **Report Type:** QWeb-PDF
- **Report Action XML ID:** `ym_approvals.action_report_ym_approval`

```python
# Example placeholder snippet
def action_approve(self):
    self.write({'state': 'approved', 'approved_by': self.env.user.id, 'approval_date': fields.Datetime.now()})
    report = self.env.ref('ym_approvals.action_report_ym_approval')
    pdf, _ = report._render_qweb_pdf(self.id)
    attachment = self.env['ir.attachment'].create({
        'name': f'Approval_{self.name}.pdf',
        'type': 'binary',
        'datas': base64.b64encode(pdf),
        'res_model': self._name,
        'res_id': self.id,
    })
    return True
```

### 4.6 Security & Access

| Group | Access |
|---|---|
| Requester | Create, Read (own records) |
| Approver | Read, Write (approve/reject) |
| Admin/HR | Full access |

[Confirm/edit actual security groups defined in `ym_approval_security.xml`]

---

## 5. Testing & Validation

- [ ] Verified list view changes display correctly for all relevant user roles
- [ ] Verified approval request creation and submission flow
- [ ] Verified PDF generates correctly on approval with accurate data
- [ ] Verified PDF is correctly attached/sent
- [ ] Verified rejection flow (if applicable)
- [ ] Regression check on existing Timesheet functionality

---

## 6. Known Issues / Limitations

- [List any current bugs, edge cases, or limitations]

## 7. Future Enhancements

- [e.g. Multi-level approval chains, mobile notifications, digital signature on PDF, etc.]

---

## 8. Change Log

| Date | Version | Description | Author |
|---|---|---|---|
| 20 June 2026 | 1.0 | Initial documentation created | [Your Name] |

