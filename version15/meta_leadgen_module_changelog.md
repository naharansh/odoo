# Meta Leadgen & Leads – Odoo Module Documentation

## Overview

This custom Odoo module integrates with the **Meta (Facebook) Marketing API** to fetch:

- **Leadgen Forms** — the lead forms configured under a Meta Ad campaign.
- **Leads** — the individual lead submissions generated from those forms.

This document tracks the recent updates made to the module: a **pagination fix** and the addition of **created-date-based filters**.

---

## Changelog

### 1. Pagination Fix (Leadgen Forms & Leads)

**Problem:**
The previous implementation fetched only the first page of results returned by the Meta Graph API, causing incomplete data when a campaign/page had more leadgen forms or leads than fit in a single API response.

**Fix:**
- Implemented cursor-based pagination handling using the `paging.next` / `paging.cursors.after` field returned by the Meta Graph API.
- The module now loops through all available pages until `paging.next` is empty, aggregating results before creating/updating Odoo records.
- Added safeguards against infinite loops (max page limit / timeout) in case of malformed API responses.

**Affected areas:**
- Leadgen Forms sync method
- Leads sync method

---

### 2. Created-Date Filter

**Feature:**
Added filtering support based on the **created date** of leads/leadgen forms, so users can fetch or view records within a specific date range instead of the full history.

**Implementation details:**
- Added `date_from` and `date_to` filter fields on the sync/import wizard (or search view — *confirm which UI element applies*).
- These are mapped to Meta Graph API's `filtering` parameter (e.g. `created_time` operator `GREATER_THAN` / `LESS_THAN`) when fetching leads.
- Added corresponding filter options in the Odoo list/search view for locally stored leads and leadgen forms, using Odoo's standard `date` domain filters on the `create_date` (or custom `meta_created_date`) field.

---

## Suggested Next Steps

- [ ] Confirm field names used for the date filter (`create_date` vs a custom Meta-provided timestamp field)
- [ ] Add automated tests for pagination (mock multi-page API responses)
- [ ] Add automated tests for date filter edge cases (timezone handling, boundary dates)
- [ ] Update module version number in `__manifest__.py`
- [ ] Document API rate-limit handling if not already covered

---

## Module Info

| Field | Value |
|---|---|
| Module Name | *(fill in)* |
| Version | *(fill in)* |
| Odoo Version | *(fill in, e.g. 17.0)* |
| Meta API Version | *(fill in, e.g. Graph API v19.0)* |
| Author | *(fill in)* |
