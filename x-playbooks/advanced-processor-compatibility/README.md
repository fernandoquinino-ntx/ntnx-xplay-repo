# Playbook: Advanced Processor Compatibility (APC) maintenance window

Manual X-Play playbook that changes a VM's CPU compatibility baseline (Advanced
Processor Compatibility / APC) inside a maintenance window, with email
notifications before and after, and an if/else that only proceeds once the VM is
confirmed **powered off**.

- **File:** [`playbook.json`](playbook.json)
- **Trigger:** Manual (`entity_type = vm`) — runs **once per selected VM**. Select
  several VMs (or a whole *category*) to batch them.
- **Prism Central API used:** v3 VM read/update (`/api/nutanix/v3/vms`) and the
  VMM v4 CPU model reference (`cpuModel.extId`).
- **Notifications:** Mailtrap Email API (`https://send.api.mailtrap.io/api/send`).

---

## Flow

| # | Action | Purpose |
|---|--------|---------|
| 0 | Power Off VM | ACPI power-off of the target VM |
| 1 | Wait for VM to power off | Pause so the guest can shut down |
| 2 | Lookup VM power state | Reads `power_state` into the action output |
| 3 | **Branch — if** `{{action[2].power_state}} == "OFF"` | Only continue when the VM is actually off |
| 4 | Mailtrap — maintenance started | "Started" email (only after OFF is confirmed) |
| 5 | REST API — GET VM | Fetches the full VM object (v3) |
| 6 | String Patch — drop `/status` | Removes the read-only `status` block (v3 PUT rejects it) |
| 7 | String Patch — enable APC | Replaces `apc_config` with the chosen CPU model |
| 8 | REST API — PUT VM | Applies the change |
| 9 | REST API — list VMs in category | Gathers the maintenance scope (VMs tagged with the category) |
| 10 | Mailtrap — maintenance completed | "Completed" email + list of VMs in scope |
| 11 | Power On VM | Powers the VM back on |
| 12 | **Branch — else** | Reached only if the VM did **not** reach OFF |
| 13 | Mailtrap — maintenance failed | "Failed" email; the VM is left unchanged |

If the VM cannot be powered off, execution jumps to the **else** branch (12 → 13)
and the CPU change is skipped.

---

## Prerequisites

1. **Prism Central 7.x** with **X-Play** enabled and API access (`admin` or a
   user that can manage action rules and VMs).
2. A **category** to mark the VMs that are in scope. The playbook assumes
   `APC:enable` (key `APC`, value `enable`). Create it if needed:
   ```bash
   curl -k -u admin:CHANGEME -H 'Content-Type: application/json' \
     -X POST "https://<PRISM_HOST>:9440/api/prism/v4.0/config/categories" \
     -d '{"key":"APC","value":"enable"}'
   ```
   Then tag the target VMs with `APC:enable`.
3. A **Mailtrap account** and an **API token** with sending access.
4. The **CPU model UUID** you want to pin (see the next section).

---

## Values to replace

The committed `playbook.json` contains placeholders. Replace them with your own
values (before import, or edit the playbook in Prism Central afterwards):

| Placeholder | Where | Replace with |
|---|---|---|
| `<PRISM_HOST>` | REST API action URLs + Mailtrap-independent calls | Your Prism Central FQDN or IP |
| `<PC_PASSWORD>` | "REST API — GET/PUT/list" actions (basic auth) | Prism Central password for your API user |
| `<MAILTRAP_API_TOKEN>` | Both Mailtrap actions (`token`, bearer auth) | Your Mailtrap API token |
| `<FROM@YOUR-VERIFIED-DOMAIN>` | Both Mailtrap actions `from.email` | A sender on a **verified** Mailtrap sending domain (or Mailtrap's demo domain `hello@demomailtrap.co` for a quick test) |
| `<RECIPIENT@EXAMPLE.COM>` | Both Mailtrap actions `to[0].email` | Who receives the notifications |
| `admin` | "REST API" actions `username` | Your Prism Central API user (default `admin`) |

Also review:

- **CPU model** (`cpu_model_reference.uuid` / `name`) in the "String Patch — enable APC" action.
- **Category key** in the "REST API — list VMs in category" action
  (`"filter": "category_name==APC"` — note: **key only**, not `key:value`).
- **Wait duration** (`wait_duration`, default `45` seconds) in the "Wait" action.
- **Maintenance window** text in the email bodies.

Secrets are **not** stored in this repo. In Prism Central the Mailtrap token and
the PC password live in the action's secret fields (masked as `********` when
read back).

---

## How to get the CPU model UUID (e.g. Intel Skylake)

APC pins a VM to a **CPU compatibility baseline**, identified by a fixed,
globally-unique `extId`/UUID. Nutanix does **not** expose an API to list these
CPU models (endpoints like `/api/vmm/v4.x/ahv/config/cpu-models` return `404`),
and a **name alone is rejected** ("CPU Model Name" invalid). You must supply the
UUID. To discover it:

1. In Prism Central, open a test VM → **Update** → set **CPU Model** to the
   baseline you want (e.g. *Intel Skylake*) → Save. (Leave APC enabled.)
2. Read the VM back and copy the `cpuModel.extId`:

   **VMM v4 (recommended):**
   ```bash
   VM=<vm-extId>
   curl -k -u admin:CHANGEME \
     "https://<PRISM_HOST>:9440/api/vmm/v4.1/ahv/config/vms/$VM" \
     | jq '.data.apcConfig.cpuModel'
   # -> { "extId": "315e63e4-d549-5a45-affd-c766032de984", "name": "Intel Skylake" }
   ```

   **v3:**
   ```bash
   curl -k -u admin:CHANGEME \
     "https://<PRISM_HOST>:9440/api/nutanix/v3/vms/$VM" \
     | jq '.spec.resources.apc_config.cpu_model_reference'
   ```

The UUID is stable across clusters, so the values below can be reused directly:

| CPU baseline | UUID |
|---|---|
| Intel Broadwell | `a2e1a75b-988c-50aa-88f3-b68e54d58958` |
| Intel Skylake | `315e63e4-d549-5a45-affd-c766032de984` |

Put the pair into the "String Patch — enable APC" action:

```json
{ "enabled": true,
  "cpu_model_reference": { "kind": "cpu_model",
                           "uuid": "<CPU_MODEL_UUID>",
                           "name": "<CPU_MODEL_NAME>" } }
```

---

## Import

```bash
PC=<prism-central-host>
curl -k -u admin:CHANGEME \
  -H 'Content-Type: application/json' \
  -X POST "https://$PC:9440/api/nutanix/v3/action_rules" \
  --data @playbook.json
```

Then, in Prism Central → **Operations → Playbooks**, open
*Enable APC (change processor) on VMs*, review/edit it, and enable it.

**Run it:** select the VM(s) — or filter by the `APC:enable` category — and click
**Run**. The playbook executes once per VM.

---

## Technical notes / gotchas

- **v3 VM update is a full-spec replace.** You must send the VM object back with
  its `spec` (and `metadata`/`api_version`), but the read-only top-level
  `status` key must be removed first (step 6) or the API returns
  `422 Additional properties are not allowed ('status')`.
- **String Patch uses JSON Pointer paths**, not JSONPath. Remove `status` with
  `/status` (not `$.status`); set the baseline at
  `/spec/resources/apc_config`.
- **Action outputs** are referenced as `{{action[N].<output>}}`, where `N` is the
  **0-based index of the action in the `action_list`**. The Branch condition must
  only reference actions earlier in its branch chain.
- **`cpuModel.name` alone is rejected** — always pass `extId`.
- **Completion email** embeds the category listing (`{{action[9].response_body}}`).
  The v3 list response is large JSON; if your X-Play build does not JSON-escape
  dynamic values, that email may fail (the playbook still completes because the
  Mailtrap actions are set to *continue on failure*). If so, replace it with a
  short scope summary (count + category) instead.
- **Power-off is ACPI + a fixed wait.** If the guest lacks tools or ignores ACPI,
  increase the wait or switch the mechanism to `hard`. The branch (step 3) makes
  the "not off" path explicit and safe.
- **Check delivery:** https://mailtrap.io/sending/email_logs
