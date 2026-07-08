# task_module

Custom Odoo module that adds a **timer** to project tasks with automatic **timesheet entry** generation and a **kanban button** for documentation.

## Features

### 1. Task Timer (Start / Stop)

- Adds **Start Timer** and **Stop Timer** buttons in the task form view header (`project.task` form).
- Start creates a new `account.analytic.line` record (timesheet line) with `unit_amount = 0` and `x_timer_start` set to the current datetime.
- Stop opens a **wizard** asking for a work description; the wizard calculates elapsed time and writes it as `unit_amount`.

### 2. Timesheet Integration

- Timer entries are stored as `account.analytic.line` records, Odoo's native timesheet model.
- Each timesheet line tracks:
  - `x_timer_start` (Datetime) — when the timer was started
  - `x_is_timer_running` (Boolean) — whether the timer is currently active
  - `name` — work description (filled on stop)
  - `unit_amount` — elapsed hours (filled on stop)

### 3. Kanban View Button

- A **play button** (`fa-play`) is added in the kanban footer next to the user_ids field, allowing timer control directly from the kanban view.

### 4. Stage Change Protection

- Prevents changing the task stage (e.g., status/column) while the timer is running — raises a `ValidationError`.

### 5. Single Active Timer Enforcement

- A user can only have **one active timer** across all tasks. Attempting to start a second timer raises a `UserError`.

## Module Structure

```
task_module/
├── __init__.py
├── __manifest__.py
├── controllers/
│   ├── __init__.py
│   └── controllers.py          # (unused — placeholder)
├── demo/
│   └── demo.xml                # (placeholder demo data, commented)
├── models/
│   ├── __init__.py
│   ├── account.py              # Inherits account.analytic.line (unused duplicate fields — see task.py)
│   ├── models.py               # (placeholder — commented out)
│   ├── task.py                 # Main logic: timer, computed fields, stage protection
│   └── wized.py                # Timer stop wizard (transient model)
├── security/
│   └── ir.model.access.csv     # ACLs for project.task, account.analytic.line, timer.stop.wizard
├── test_timer.py               # Quick test script for timer + stage-blocking logic
└── views/
    ├── task.xml                # Form & kanban view inheritance, record rule
    ├── templates.xml           # (placeholder — commented)
    ├── views.xml               # (placeholder — commented)
    └── wizad.xml               # Wizard form view
```

## Key Models

### `project.task` (inherited)
| Field | Type | Description |
|-------|------|-------------|
| `is_timer_running` | Boolean (computed) | Whether the current user has a running timer on this task |

### `account.analytic.line` (inherited) — defined in `task.py`
| Field | Type | Description |
|-------|------|-------------|
| `x_timer_start` | Datetime | When the timer was started |
| `x_is_timer_running` | Boolean | Whether this line is an active timer |

### `timer.stop.wizard` (transient)
| Field | Type | Description |
|-------|------|-------------|
| `description` | Text | Work description (required) |
| `task_id` | Many2one → `project.task` | The task being timed |
| `analytic_line_id` | Many2one → `account.analytic.line` | The timesheet line to stop |

## Usage

1. Open a **Project → Task** form view.
2. Click **▶ Start Timer** to begin tracking.
3. Work on the task — the stage cannot be changed while the timer runs.
4. Click **⏹ Stop Timer** → enter a description → click **Save**.
5. The timesheet entry is created with elapsed hours automatically calculated.
6. From the **kanban view**, click the **play icon** to toggle the timer.

## Dependencies

- `base`
- `project`
- `hr_timesheet`

## Notes

- `models/account.py` defines `timer_start` and `is_timer_running` (without `x_` prefix) on `account.analytic.line` but these fields are **not used** by the main logic. The active fields are `x_timer_start` and `x_is_timer_running` from `models/task.py`. `account.py` can be safely removed or consolidated.
