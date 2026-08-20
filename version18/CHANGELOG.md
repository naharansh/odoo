# Changelog - 20 August 2026

## 1. Admin-Only Deletion Restriction

### Domains (`ym.server.servers`)
- **File:** `models/servers.py`
- Added `unlink()` method that raises `UserError` if the user is not in `group_server_management_manager`
- Added `UserError` import

### Domain Users (`ym.server.management`)
- **File:** `models/server.py`
- Modified existing `unlink()` method to check admin group before allowing deletion
- Raises `UserError` if the user is not an administrator
- Preserved existing check for associated servers

---

## 2. Cpanel Selection from WHM

### Domain Form (`ym.server.servers`)
- **File:** `models/servers.py`
- Changed `cpanel_ids` (Many2many) to `cpanel_id` (Many2one)
- Added domain filter: `[('whm_id', '=', whm_id)]` to only show cpanels from the selected WHM
- Only one cpanel can be selected per domain

### Views
- **File:** `views/views.xml`
- Updated list views to use `cpanel_id` instead of `cpanel_ids`
- Updated form view to show single cpanel selection instead of multiple
- Removed `readonly="1"` to allow editing
