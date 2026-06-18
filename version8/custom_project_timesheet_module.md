# Custom Module: `custom_project_timesheet`

**Odoo Version:** 19.0
**Module Type:** Custom / Functional Extension
**Depends On:** `project`, `hr_timesheet`, `analytic`

---

## 1. Overview

This custom module extends Odoo's native **Timesheet** functionality by linking timesheet entries directly to **Projects** and **Tasks**. It builds on top of the standard `hr_timesheet` module, allowing employees to log time against a specific Project and, optionally, a specific Task within that project — improving time tracking, reporting, and project profitability analysis.

In stock Odoo 19, the link between Timesheets and Projects already exists at a basic level. This module enhances that relationship with additional constraints, defaults, computed fields, and reporting views tailored to business requirements.

---

## 2. Module Structure

```
custom_project_timesheet/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── account_analytic_line.py      # Timesheet model extension
│   └── project_task.py               # Task model extension (optional reverse link)
├── views/
│   ├── timesheet_views.xml           # Form/List/Tree view inheritance
│   └── project_task_views.xml        # Task view showing related timesheets
├── security/
│   └── ir.model.access.csv
├── reports/
│   └── timesheet_project_report.xml  # Pivot/Graph report views
├── data/
│   └── ir_sequence_data.xml          # (if applicable)
└── static/
    └── description/
        └── icon.png
```

---

## 3. Manifest File (`__manifest__.py`)

```python
{
    'name': 'Project Linked Timesheets',
    'version': '19.0.1.0.0',
    'category': 'Services/Project',
    'summary': 'Link Timesheets with Projects and Tasks',
    'description': """
        This module enhances Odoo Timesheets by adding a structured
        link to Projects and Tasks, enabling better time tracking
        and project-based reporting.
    """,
    'author': 'Your Company',
    'depends': ['project', 'hr_timesheet', 'analytic'],
    'data': [
        'security/ir.model.access.csv',
        'views/timesheet_views.xml',
        'views/project_task_views.xml',
        'reports/timesheet_project_report.xml',
    ],
    'installable': True,
    'application': False,
    'auto_install': False,
    'license': 'LGPL-3',
}
```

---

## 4. Technical Implementation

### 4.1 Model Extension — `account.analytic.line`

The Timesheet entry model in Odoo is `account.analytic.line` (inherited by `hr_timesheet`). This module extends it to enforce and enrich the Project–Task relationship.

```python
from odoo import models, fields, api

class AccountAnalyticLine(models.Model):
    _inherit = 'account.analytic.line'

    project_id = fields.Many2one(
        'project.project',
        string='Project',
        help='Project this timesheet entry is linked to.'
    )
    task_id = fields.Many2one(
        'project.task',
        string='Task',
        domain="[('project_id', '=', project_id)]",
        help='Specific task within the selected project.'
    )
    is_project_linked = fields.Boolean(
        string='Linked to Project',
        compute='_compute_is_project_linked',
        store=True
    )

    @api.depends('project_id')
    def _compute_is_project_linked(self):
        for line in self:
            line.is_project_linked = bool(line.project_id)

    @api.onchange('project_id')
    def _onchange_project_id(self):
        if self.project_id:
            self.task_id = False  # reset task when project changes
        return {'domain': {'task_id': [('project_id', '=', self.project_id.id)]}}

    @api.constrains('project_id', 'task_id')
    def _check_task_belongs_to_project(self):
        for line in self:
            if line.task_id and line.task_id.project_id != line.project_id:
                raise models.ValidationError(
                    "The selected task does not belong to the selected project."
                )
```

**Key Points:**
- `task_id` domain dynamically filters tasks based on the selected `project_id`.
- A SQL/Python constraint ensures data integrity — a task must belong to the chosen project.
- `is_project_linked` is a stored computed field used for filtering/reporting (e.g., timesheets not yet linked to any project).

### 4.2 Model Extension — `project.task` (Reverse Relation)

```python
from odoo import models, fields

class ProjectTask(models.Model):
    _inherit = 'project.task'

    timesheet_ids = fields.One2many(
        'account.analytic.line',
        'task_id',
        string='Timesheet Entries'
    )
    total_hours_logged = fields.Float(
        string='Total Hours Logged',
        compute='_compute_total_hours_logged',
        store=True
    )

    @api.depends('timesheet_ids.unit_amount')
    def _compute_total_hours_logged(self):
        for task in self:
            task.total_hours_logged = sum(task.timesheet_ids.mapped('unit_amount'))
```

This allows each Task form to display all timesheet entries logged against it, along with a total-hours summary.

---

## 5. View Changes

### 5.1 Timesheet Form/List View Inheritance (`timesheet_views.xml`)

```xml
<odoo>
    <record id="view_timesheet_form_inherit" model="ir.ui.view">
        <field name="name">timesheet.form.inherit.project</field>
        <field name="model">account.analytic.line</field>
        <field name="inherit_id" ref="hr_timesheet.hr_timesheet_line_form"/>
        <field name="arch" type="xml">
            <xpath expr="//field[@name='project_id']" position="after">
                <field name="task_id" options="{'no_create': True}"/>
                <field name="is_project_linked" invisible="1"/>
            </xpath>
        </field>
    </record>

    <record id="view_timesheet_tree_inherit" model="ir.ui.view">
        <field name="name">timesheet.tree.inherit.project</field>
        <field name="model">account.analytic.line</field>
        <field name="inherit_id" ref="hr_timesheet.hr_timesheet_line_tree"/>
        <field name="arch" type="xml">
            <xpath expr="//field[@name='project_id']" position="after">
                <field name="task_id"/>
            </xpath>
        </field>
    </record>
</odoo>
```

### 5.2 Task Form View — Timesheet Smart Button (`project_task_views.xml`)

```xml
<odoo>
    <record id="view_task_form_inherit_timesheet" model="ir.ui.view">
        <field name="name">project.task.form.inherit.timesheet</field>
        <field name="model">project.task</field>
        <field name="inherit_id" ref="project.view_task_form2"/>
        <field name="arch" type="xml">
            <xpath expr="//div[@name='button_box']" position="inside">
                <button class="oe_stat_button" type="object"
                        name="action_view_timesheets" icon="fa-clock-o">
                    <field name="total_hours_logged" widget="statinfo" string="Hours"/>
                </button>
            </xpath>
        </field>
    </record>
</odoo>
```

---

## 6. Security (`ir.model.access.csv`)

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_account_analytic_line_project,access.analytic.line.project,model_account_analytic_line,project.group_project_user,1,1,1,0
```

---

## 7. Functional Use Case

### 7.1 Business Scenario

A services company needs employees to log daily work hours against specific client projects and, more granularly, against individual tasks (e.g., "Bug Fixing," "Client Meeting," "Feature Development") within that project — to support accurate project costing and client billing.

### 7.2 User Flow

1. **Navigate** to **Timesheets → My Timesheets**.
2. Click **New** to create a timesheet entry.
3. Select the **Project** from the dropdown (e.g., "Website Redesign").
4. The **Task** field becomes enabled and filtered to show only tasks under that project (e.g., "Homepage UI," "Checkout Flow").
5. Enter the **Date**, **Hours Spent**, and a **Description**.
6. Save the entry — it now appears in both the Timesheet log and on the related Task's form (via the **Hours** smart button).

### 7.3 Reporting Benefits

- **Project Managers** can open any Task and instantly see total hours logged against it.
- **Pivot/Graph views** (via `timesheet_project_report.xml`) allow grouping timesheets by Project → Task → Employee for profitability and utilization analysis.
- The `is_project_linked` field allows filtering out "orphan" timesheet entries not yet tied to a project, helping enforce data discipline.

---

## 8. Testing Checklist

| # | Test Case | Expected Result |
|---|-----------|------------------|
| 1 | Create timesheet without selecting a project | Task field remains empty/disabled |
| 2 | Select a project, then a task from another project | Validation error raised |
| 3 | Change project after selecting a task | Task field resets to empty |
| 4 | Open a Task with logged timesheets | "Hours" smart button shows correct total |
| 5 | Filter timesheets by `is_project_linked = False` | Shows only entries with no project set |

---

## 9. Future Enhancements

- Add **billing rate** override per Project-Task combination.
- Auto-suggest the **last used task** per project for faster timesheet entry.
- Add **approval workflow** for timesheets linked to specific high-priority projects.
- Dashboard widget showing **hours logged vs. planned hours** per project.

---

## 10. Changelog

| Version | Date | Notes |
|---------|------|-------|
| 19.0.1.0.0 | Initial Release | Project–Task linkage on Timesheets, validation, reporting views |

---

*Document generated for internal development reference — Custom Odoo 19.0 Module.*
