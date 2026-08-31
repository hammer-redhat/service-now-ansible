# ServiceNow Ansible Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create two Ansible playbooks — one to create a ServiceNow Change Request with RHBA/RHSA/RHEA advisory notes, one to close a CR that is in the implement state.

**Architecture:** Both playbooks target `localhost` and make REST calls to ServiceNow via the `servicenow.itsm` collection. Auth is injected automatically by the AAP `ServiceNow ITSM` credential type as environment variables. The create playbook accepts advisory data as `extra_vars`; the close playbook accepts a CR number as `extra_vars` and guards against closing a CR that is not in the implement state.

**Tech Stack:** Ansible, `servicenow.itsm` collection (>=2.0.0), AAP (Ansible Automation Platform)

**Spec:** `docs/superpowers/specs/2026-08-31-snow-ansible-integration-design.md`

## Global Constraints

- Collection: `servicenow.itsm`, version `>=2.0.0`
- All playbooks target `localhost` with `connection: local` and `gather_facts: false`
- No credentials in any playbook file; auth comes from AAP credential type env vars: `SN_HOST`, `SN_USERNAME`, `SN_PASSWORD`
- `extra_vars` names are exact: `patch_advisories` (list), `change_request_number` (string)
- Static field values must match exactly: short_description `"AO-CVE-Pipeline"`, assignment_group `"Conditional Script Writer"`, change_type `"normal"`, category `"Server Reboot"`, close_code `"successful"`

---

### Task 1: Scaffold — collection requirements and inventory

**Files:**
- Create: `collections/requirements.yml`
- Create: `inventory/localhost.yml`

**Interfaces:**
- Produces: collection pin used by all playbooks; inventory used by all playbooks

- [ ] **Step 1: Create `collections/requirements.yml`**

```yaml
---
collections:
  - name: servicenow.itsm
    version: ">=2.0.0"
```

- [ ] **Step 2: Create `inventory/localhost.yml`**

```yaml
---
all:
  hosts:
    localhost:
      ansible_connection: local
```

- [ ] **Step 3: Verify syntax of both files**

Run:
```bash
ansible-playbook --syntax-check -i inventory/localhost.yml /dev/null 2>&1 || true
python3 -c "import yaml; yaml.safe_load(open('collections/requirements.yml'))" && echo "requirements.yml: valid YAML"
python3 -c "import yaml; yaml.safe_load(open('inventory/localhost.yml'))" && echo "inventory/localhost.yml: valid YAML"
```

Expected: Both print "valid YAML" with no errors.

- [ ] **Step 4: Commit**

```bash
git add collections/requirements.yml inventory/localhost.yml
git commit -m "feat: add servicenow.itsm collection pin and localhost inventory"
```

---

### Task 2: Create Change Request playbook

**Files:**
- Create: `playbooks/snow_create_change_request.yml`

**Interfaces:**
- Consumes: `inventory/localhost.yml` (Task 1), `collections/requirements.yml` (Task 1)
- Consumes extra_vars: `patch_advisories` — YAML list; each item may be a multiline string of newline-separated advisory IDs (e.g. `"RHSA-2026:61247\nRHBA-2026:60213\n..."`)
- Produces: Change Request created in ServiceNow; CR number printed to stdout via `debug`

**Notes on `patch_advisories`:** The variable arrives as a YAML list. Each element may itself be a multiline block-scalar string. Using `patch_advisories | join('\n')` safely concatenates all elements regardless of how many are passed.

**Notes on `servicenow.itsm.change_request`:** The module reads `SN_HOST`, `SN_USERNAME`, `SN_PASSWORD` from the environment automatically (no `instance` key needed when env vars are set). `work_notes` is passed via the `other` dict and appends to the CR's activity log. `change_type` must be lowercase (`normal`, not `Normal`).

- [ ] **Step 1: Write the playbook**

Create `playbooks/snow_create_change_request.yml`:

```yaml
---
- name: Create ServiceNow Change Request for CVE Patching
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    snow_short_description: "AO-CVE-Pipeline"
    snow_assignment_group: "Conditional Script Writer"
    snow_change_type: "normal"
    snow_category: "Server Reboot"

  tasks:
    - name: Validate patch_advisories is provided
      ansible.builtin.fail:
        msg: "patch_advisories extra_var is required and must not be empty."
      when: patch_advisories is not defined or patch_advisories | length == 0

    - name: Create change request with advisory work notes
      servicenow.itsm.change_request:
        state: new
        short_description: "{{ snow_short_description }}"
        assignment_group: "{{ snow_assignment_group }}"
        change_type: "{{ snow_change_type }}"
        category: "{{ snow_category }}"
        other:
          work_notes: "{{ patch_advisories | join('\n') }}"
      register: created_cr

    - name: Display created Change Request number
      ansible.builtin.debug:
        msg: "Created Change Request: {{ created_cr.record.number }}"
```

- [ ] **Step 2: Run syntax check**

Run:
```bash
ansible-playbook --syntax-check -i inventory/localhost.yml playbooks/snow_create_change_request.yml
```

Expected: `playbook: playbooks/snow_create_change_request.yml` with no errors. If the `servicenow.itsm` collection is not locally installed, install it first:
```bash
ansible-galaxy collection install -r collections/requirements.yml
```

- [ ] **Step 3: Run ansible-lint**

Run:
```bash
ansible-lint playbooks/snow_create_change_request.yml
```

Expected: No rule violations. If `ansible-lint` is not installed: `pip install ansible-lint`. Address any violations before proceeding.

- [ ] **Step 4: Commit**

```bash
git add playbooks/snow_create_change_request.yml
git commit -m "feat: add snow_create_change_request playbook with advisory work notes"
```

---

### Task 3: Close Change Request playbook

**Files:**
- Create: `playbooks/snow_close_change_request.yml`

**Interfaces:**
- Consumes: `inventory/localhost.yml` (Task 1), `collections/requirements.yml` (Task 1)
- Consumes extra_vars: `change_request_number` — string, e.g. `"CHG0012345"`
- Produces: CR closed in ServiceNow if and only if its current state is `implement`; fails with descriptive message otherwise

**Notes on state guard:** `servicenow.itsm.change_request_info` returns a `records` list. Index `[0]` is the matched CR. The `state` field is a human-readable string from the collection — `"implement"` (lowercase). The playbook must check this before attempting to close.

**Notes on `close_code`:** Valid values for the `servicenow.itsm.change_request` module are `"successful"`, `"successful_issues"`, and `"unsuccessful"` (lowercase, underscored).

- [ ] **Step 1: Write the playbook**

Create `playbooks/snow_close_change_request.yml`:

```yaml
---
- name: Close ServiceNow Change Request (implement state only)
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Validate change_request_number is provided
      ansible.builtin.fail:
        msg: "change_request_number extra_var is required (e.g. CHG0012345)."
      when: change_request_number is not defined or change_request_number | length == 0

    - name: Fetch Change Request details
      servicenow.itsm.change_request_info:
        number: "{{ change_request_number }}"
      register: cr_info

    - name: Fail if no matching CR found
      ansible.builtin.fail:
        msg: "No Change Request found with number {{ change_request_number }}."
      when: cr_info.records | length == 0

    - name: Fail if CR is not in implement state
      ansible.builtin.fail:
        msg: >-
          CR {{ change_request_number }} is currently in '{{ cr_info.records[0].state }}' state.
          Automation will only close CRs in 'implement' state. Take no action.
      when: cr_info.records[0].state != "implement"

    - name: Close Change Request
      servicenow.itsm.change_request:
        state: closed
        number: "{{ change_request_number }}"
        close_code: "successful"
        close_notes: "Change request closed successfully by AO-CVE-Pipeline automation."
```

- [ ] **Step 2: Run syntax check**

Run:
```bash
ansible-playbook --syntax-check -i inventory/localhost.yml playbooks/snow_close_change_request.yml
```

Expected: `playbook: playbooks/snow_close_change_request.yml` with no errors.

- [ ] **Step 3: Run ansible-lint**

Run:
```bash
ansible-lint playbooks/snow_close_change_request.yml
```

Expected: No rule violations.

- [ ] **Step 4: Commit**

```bash
git add playbooks/snow_close_change_request.yml
git commit -m "feat: add snow_close_change_request playbook with implement-state guard"
```

---

### Task 4: Update README

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: all files from Tasks 1–3

- [ ] **Step 1: Write README content**

Replace `README.md` with:

```markdown
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
```

- [ ] **Step 2: Verify README renders correctly**

Open `README.md` and confirm the tables and code blocks look correct (no broken markdown).

- [ ] **Step 3: Commit**

```bash
git add README.md docs/
git commit -m "docs: add README and design spec for ServiceNow Ansible integration"
```
