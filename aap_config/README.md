# Network Drift — AAP Setup Guide

This guide walks you through standing up the **Network Drift** solution inside
**Ansible Automation Platform (AAP)** from scratch. Follow the steps in order.

The heavy lifting is done by the config-as-code playbooks in this directory
(`deploy_aap.yml` imports `00`–`06`), which create the credential types,
credentials, inventory, execution environment, job templates, and the
remediation workflow for you. The steps below are the small number of **manual**
actions a person must perform (because they involve secrets or one-time
bootstrapping) plus the order in which to run things.

> **Why some steps are manual:** Per the "no secrets in git" standard, sensitive
> values (registry login, ServiceNow login) are never stored in this repo. You
> enter them once in the AAP UI after the object shells are created.

---

## Prerequisites

- An AAP 2.4+ instance you can log in to as an administrator.
- The **`Red Hat network organization`** organization exists in AAP.
- A working **`Controller Credential`** (AAP/Controller credential type) so the
  bootstrap job template can talk to the AAP API.
- The **Network Workshop** lab objects are present (`Workshop Inventory`,
  `Workshop Credential`, `Demo Inventory`, `registry.redhat.io credential`,
  `Default execution environment`).

---

## Step 1 — Create the project (manual)

Create a Project in AAP that points at this Git repository.

| Field | Value |
|---|---|
| Name | `network_drift` |
| Organization | `Red Hat network organization` |
| Source Control Type | Git |
| Source Control URL | `https://github.com/mlowcher61/network_drift.git` |
| Source Control Branch | `main` |
| Update revision on launch | ✅ enabled |

> The project is created manually (rather than by `02_project.yml`) because the
> bootstrap job template in Step 3 needs the project to already exist in order to
> find the `aap_config/deploy_aap.yml` playbook.

---

## Step 2 — Set the registry credential (manual)

Edit the **`registry.redhat.io credential`** and replace the username and
password with **your own** Red Hat registry service-account credentials.

This lets AAP pull the execution environment image used by the job templates.

---

## Step 3 — Create and run the "Deploy network drift" template

Create a **Job Template** that runs the config-as-code bootstrap playbook.

| Field | Value |
|---|---|
| Name | `Deploy network drift` |
| Inventory | `Demo Inventory` |
| Project | `network_drift` |
| Playbook | `aap_config/deploy_aap.yml` |
| Execution Environment | `Default execution environment` |
| Credentials | `Controller Credential` |

**Launch this template.**

It creates everything the solution needs:

| Object | Name |
|---|---|
| Credential type | `servicenow` |
| Credential (shell) | `cred_servicenow` |
| Inventory | `Network Drift Inventory` (cloned from `Workshop Inventory`) |
| Execution environment | `network_drift_ee` |
| Job templates | `Drift detect`, `Push Config`, `Apply initial config to devices` |
| Workflow template | `Network Drift and Remediation workflow` |

> The bootstrap playbook is **idempotent** — you can re-run "Deploy network
> drift" any time to reconcile drift in AAP itself.

---

## Step 4 — Disable rtr3 in the inventory (manual)

Open **`Network Drift Inventory`** → **Hosts**, select **`rtr3`**, and toggle it
to **Disabled**.

> `rtr3` is excluded from this demo run. Disabling the host keeps it out of all
> job-template runs without removing it from the cloned inventory.

---

## Step 5 — Fill in the ServiceNow credential (manual)

Open the **`cred_servicenow`** credential and replace the `PLACEHOLDER` values
with your real ServiceNow details:

| Field | Value |
|---|---|
| ServiceNow Instance URL (`SN_HOST`) | your ServiceNow instance URL |
| Username (`SN_USERNAME`) | your ServiceNow username |
| Password (`SN_PASSWORD`) | your ServiceNow password |

> ⚠️ **`SN_HOST` must point at your ServiceNow instance** — not the AAP
> hostname. Pointing it at AAP causes `Drift detect` to fail when it tries to
> open the incident ticket.

---

## Step 6 — Apply the initial configuration

Launch the **`Apply initial config to devices`** job template.

This pushes the Source of Truth from `host_vars/` to every (enabled) router,
establishing the known-good baseline that drift is later measured against.

---

## You're ready

The environment is now fully provisioned. To drive the demo:

1. SSH into a router and change its `ntp`, `logging`, or `snmp` configuration.
2. Launch the **`Network Drift and Remediation workflow`**.
   - `Drift detect` runs in check mode, finds the diff, and opens a ServiceNow
     incident.
   - The workflow pauses at the **Human Approval** node.
3. Approve it under **Automation Execution → Workflow Approvals**.
4. `Push Config` reapplies `host_vars/` to the device and resolves the ticket.

See the [top-level README](../README.md) for the full solution design.
