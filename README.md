# Nutanix X-Play Reference Playbooks

A curated collection of reference **Nutanix X-Play** playbooks that you can import, study and adapt.

## What is a Nutanix X-Play playbook?

**Nutanix X-Play** is the automation and remediation engine built into **Nutanix Prism Central**. It lets you define **playbooks** — automated workflows that react to conditions in your environment — and run them against Nutanix entities such as VMs, clusters and hosts.

X-Play playbooks are stored internally as **action rules** and are managed in Prism Central under **Operations → Playbooks**. They can also be created, read, updated, exported and executed through the Prism Central v3 REST API (`/api/nutanix/v3/action_rules`). Playbooks have a `rule_type` of `XPLAY`.

A playbook is built from three kinds of blocks:

**1. Triggers** — what starts the playbook:

| Trigger | Starts when… |
|---|---|
| **Manual** | Run on demand against one or more selected entities (e.g. pick a *category* to target all its VMs). |
| **Alert** / **Alerts Matching Criteria** | A Prism alert fires (optionally filtered by severity, category, policy). |
| **Event** | A Nutanix event occurs, optionally filtered by category. |
| **Webhook** | An external system calls a generated URL. |
| **Time** | On a schedule (once or recurring). |

**2. Actions** — the individual steps, e.g. *Power On/Off VM, Restart VM, Create Snapshot, Add/Remove vCPU, Add/Remove Memory, Lookup VM Details, Run REST API call, Send Email, Run Script, Wait*.

**3. Branch** — if/else decision logic that routes execution based on expressions over action outputs, e.g. `{{action[2].power_state}} == "OFF"`.

## Playbooks in this repository

| Playbook | What it does | Trigger | Link |
|---|---|---|---|
| **Advanced Processor Compatibility (APC) maintenance window** | Powers a VM off, verifies it reached `OFF` (if/else), sets the CPU compatibility baseline through the Prism VMM API, sends Mailtrap email notifications before/after, and powers the VM back on. | Manual (runs once per selected VM) | [x-playbooks/advanced-processor-compatibility](x-playbooks/advanced-processor-compatibility/) |

## Importing a playbook

Each playbook folder contains a `playbook.json` in the Prism Central **action-rule API format**, plus a `README.md` explaining every step and what to configure.

**Option A — REST API (recommended, reproducible)**

```bash
PC=<prism-central-host>
curl -k -u admin:CHANGEME \
  -H 'Content-Type: application/json' \
  -X POST "https://$PC:9440/api/nutanix/v3/action_rules" \
  --data @x-playbooks/advanced-processor-compatibility/playbook.json
```

**Option B — Prism Central UI**

Recreate the playbook on the **Operations → Playbooks** page following the playbook's `README.md`, or import an X-Play export file if one is provided.

> Values such as `<PRISM_HOST>`, `<PC_PASSWORD>`, `<MAILTRAP_API_TOKEN>` and the email addresses are **placeholders** — replace them with real values before/after import. **Never commit real secrets.**

## Security

- No credentials, API tokens, hostnames or email addresses are committed; every environment-specific value is a placeholder.
- Set secrets at import time (see each playbook's `README.md`).
- See `.gitignore` for the local files that are intentionally excluded.

## References

- Nutanix Support & Insights (documentation portal) — https://portal.nutanix.com/
- Nutanix.dev (developer docs, APIs and samples) — https://www.nutanix.dev/
