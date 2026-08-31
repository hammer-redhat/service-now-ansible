# service-now-ansible

Ansible playbooks for ServiceNow Change Request management, designed to run in **Ansible Automation Platform (AAP)**.

## Prerequisites

- AAP with a **ServiceNow ITSM** credential type configured for `ven07618.service-now.com`
- `servicenow.itsm` collection (>=2.0.0) installed in your AAP execution environment or via:

  ```bash
  ansible-galaxy collection install -r collections/requirements.yml
  ```

## Playbooks

### `playbooks/snow_create_change_request.yml`

Creates a ServiceNow Change Request and adds RHBA/RHSA/RHEA advisory IDs as work notes.

**Required extra_vars:**

| Variable | Type | Description |
|---|---|---|
| `patch_advisories` | list | One or more block-scalar strings of newline-separated advisory IDs |

**Example extra_vars (YAML):**
```yaml
patch_advisories:
  - |
    RHSA-2026:61247
    RHBA-2026:60213
    RHBA-2026:60221
```

**Static fields set by the playbook:**

| Field | Value |
|---|---|
| Short description | `AO-CVE-Pipeline` |
| Assignment group | `Conditional Script Writer` |
| Change type | `normal` |
| Category | `Server Reboot` |

---

### `playbooks/snow_close_change_request.yml`

Closes a ServiceNow Change Request. Only operates on CRs in the **implement** state — fails with a descriptive message for any other state.

**Required extra_vars:**

| Variable | Type | Description |
|---|---|---|
| `change_request_number` | string | The CR number to close, e.g. `CHG0012345` |

**Close fields (static):**

| Field | Value |
|---|---|
| Close code | `successful` |
| Close notes | `Change request closed successfully by AO-CVE-Pipeline automation.` |

## AAP Setup

1. Add a **ServiceNow ITSM** credential in AAP pointed at `ven07618.service-now.com`.
2. Create a Job Template for each playbook using the `inventory/localhost.yml` inventory.
3. Attach the ServiceNow credential to each Job Template.
4. Add a survey (or workflow variables) to supply the required `extra_vars` listed above.
