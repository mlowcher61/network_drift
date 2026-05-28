# Ansible Automation Platform — Network Drift Demo

An **Ansible Automation Platform (AAP)** solution that detects and remediates
**network configuration drift** on Cisco IOS routers by comparing the running
configuration against a Source of Truth stored in `host_vars/` in this
repository. Drift events open ServiceNow incident tickets; an approval-gated
workflow then pushes the corrected configuration back and resolves the ticket.

> **Demo environment:** This solution is built for the
> [Red Hat Demo Platform](https://demo.redhat.com) "Network Workshop" lab.

---

## Why this design

| Decision | Choice | Reason |
|---|---|---|
| Source of truth | **`host_vars/<router>/*.yaml`** | One YAML file per resource module (ntp_global, logging_global, snmp_server, …) — readable, diffable, fits Git. |
| Drift detection | **`cisco.ios.*` resource modules in check mode** | Certified content. Each resource module returns structured `gathered` vs. `rendered` diffs without parsing CLI text. |
| Remediation gating | **Workflow approval node** | A human approves before any change touches a device. |
| Secrets | **AAP custom credential type** | ServiceNow values are injected as environment variables at run time — never vaulted into git. Aligns with the "no secrets in repo" standard. |
| AAP automation | **`ansible.platform` collection** | Red Hat certified replacement for `ansible.controller`, with `module_defaults` to remove repetition. |

---

## Repository layout

```
network_drift/
├── ansible.cfg                       # roles/, collections/, inventories/ paths
├── collections/requirements.yml      # certified content (ansible.platform, cisco.ios, ...)
├── drift.yml                         # entry point — drift detect
├── push_config.yml                   # entry point — remediation
├── gather_state_to_git.yml           # utility — refresh host_vars from devices
├── host_vars/<router>/*.yaml         # source of truth (per-resource YAML)
├── roles/
│   ├── ntp_global/                    # resource-module drift role
│   ├── logging_global/                # resource-module drift role
│   ├── snmp_server/                   # resource-module drift role
│   ├── interfaces/
│   └── update_incident/               # ServiceNow integration
└── aap_config/                       # config-as-code (ansible.platform)
    ├── deploy_aap.yml                 # orchestrator (imports 00→05)
    ├── 00_credential_types.yml        # servicenow custom credential type
    ├── 01_credentials.yml             # cred_servicenow shell (PLACEHOLDER)
    ├── 02_project.yml                 # network_drift project
    ├── 03_inventory.yml               # clones Workshop Inventory + group vars
    ├── 04_execution_environment.yml   # registers network_drift_ee
    └── 05_job_templates.yml           # Drift detect / Push Config / Apply initial
```

---

## Prerequisites

- AAP 2.4+ with the **`ansible.platform`** collection available.
- An existing **"Workshop Inventory"** (provided by the Network Workshop lab).
- An existing **"Workshop Credential"** for SSH access to the lab routers.
- A ServiceNow instance for the ITSM integration (any developer instance).

---

## Day-0 — Deploy AAP objects (config-as-code)

The `aap_config/` playbooks are idempotent — rerun any time to reconcile drift
in AAP itself.

```bash
ansible-playbook aap_config/deploy_aap.yml \
  -e aap_hostname=aap.example.com \
  -e aap_username=admin \
  -e aap_password='********' \
  -e aap_organization='Red Hat network organization'
```

What you get:

| Object | Name |
|---|---|
| Credential type | `servicenow` |
| Credential (shell) | `cred_servicenow` |
| Project | `network_drift` (SCM = this repo) |
| Inventory | `Network Drift Inventory` (clone of Workshop Inventory) |
| Execution environment | `network_drift_ee` (`quay.io/locust61/network-ee:02`) |
| Job templates | `Drift detect`, `Push Config`, `Apply initial config to devices` |

### After first deploy

1. **Fill in `cred_servicenow`** in the AAP UI: set `SN_HOST`, `SN_USERNAME`,
   `SN_PASSWORD`. The playbook only creates the shell — secrets never live in
   this repo.
2. Run **`Apply initial config to devices`** once to push the source of truth
   from `host_vars/` to every router.
3. Manually create a **workflow template** wiring `Drift detect` → approval
   node → `Push Config`.

---

## Day-1 — Drive the demo

1. SSH into any router and change `ntp`, `logging`, or `snmp` config.
2. Launch the workflow. `Drift detect` runs in check mode, finds the diff, and
   opens a ServiceNow incident per drifted resource.
3. Approve the workflow in **Automation Execution → Administration → Workflow
   Approvals**.
4. `Push Config` reapplies `host_vars/` to the device, comments the
   remediation commands on the ServiceNow ticket, and marks it resolved.

---

## Updating the source of truth

`gather_state_to_git.yml` is a developer utility that re-reads resource state
from every router and writes it back into `host_vars/`. Edit the `repo_root`
variable at the top before running it locally.

---

> Originally derived from the "Managing Configuration Drift with Resource
> Modules" demo and rebuilt around AAP config-as-code.
