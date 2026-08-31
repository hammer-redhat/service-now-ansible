# ServiceNow Ansible Integration — Design

**Date:** 2026-08-31  
**Instance:** ven07618.service-now.com  
**Approved in:** conversation on 2026-08-31

## Goal

Two Ansible playbooks that integrate with ServiceNow via the `servicenow.itsm` collection, running in AAP (Ansible Automation Platform).

## Playbook 1: Create Change Request

- Target: `localhost` (REST call — no SSH target)
- Collection: `servicenow.itsm.change_request`
- Auth: AAP credential type `ServiceNow ITSM` injects `SN_HOST`, `SN_USERNAME`, `SN_PASSWORD` as env vars automatically
- Static fields baked into playbook:
  - `short_description`: `"AO-CVE-Pipeline"`
  - `assignment_group`: `"Conditional Script Writer"`
  - `change_type`: `"normal"`
  - `category`: `"Server Reboot"`
- Dynamic field via `extra_vars`:
  - `patch_advisories` — YAML list with a block-scalar item containing newline-separated RHBA/RHSA/RHEA IDs; written to `work_notes` via `other` dict
- Output: CR number printed to AAP job output

## Playbook 2: Close Change Request

- Target: `localhost`
- Collection: `servicenow.itsm.change_request_info` + `servicenow.itsm.change_request`
- Auth: same AAP credential type
- Dynamic field via `extra_vars`:
  - `change_request_number` — e.g. `CHG0012345`
- Guard: fetch CR, check `state == "implement"`; fail with clear message if not
- Close fields (static):
  - `close_code`: `"successful"`
  - `close_notes`: `"Change request closed successfully by AO-CVE-Pipeline automation."`

## Repo Structure

```
service-now-ansible/
├── collections/
│   └── requirements.yml
├── inventory/
│   └── localhost.yml
├── playbooks/
│   ├── snow_create_change_request.yml
│   └── snow_close_change_request.yml
└── README.md
```
