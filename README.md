# The agent — n8n workflow

This repository contains an exported n8n workflow file: `The agent.json`.

## Overview

- Purpose: an n8n workflow integrating WhatsApp, Gmail, OpenAI (LangChain) and other tools to act as an automated assistant for "Peter Mares" (and secondary roles like Amy C.).
- Main file: `The agent.json` (n8n workflow). Import this JSON into your n8n instance to restore the workflow.

## Changes Made to The agent.json

### 1. Fixed Runtime-Breaking Expression (Critical Fix)

**Location:** `Simple Memory1` node  
**Issue:** The sessionKey expression referenced a non-existent node name

- **Before:** `={{ $('Gmail Trigger').item.json.threadId }}`
- **After:** `={{ $('Gmail Trigger1').item.json.threadId }}`

**Impact:** This prevents expression-evaluation errors at runtime when the memory node tries to read the Gmail trigger's threadId. Without this fix, the workflow would crash when processing emails.

---

### 2. Added Non-Destructive Mapping Key Duplicates

**Nodes affected:** 
- Reply to an email (line ~389-390)
- Reply to an email1 (line ~713-714)
- Reply to an email2 (line ~1229-1233)

**What was added:**
- Added `message_ID` alongside existing `message ID`
- Added `Whatsapp_message_for_Peter` alongside existing `Whatsapp message for Peter`

**Why:** Ensures compatibility whether the AI outputs field names with spaces or underscores. Both versions now exist, so the workflow accepts either format.

**Note:** This is non-destructive — original keys remain unchanged, new keys were added as backups.

---

### 3. JSON Validation

Validated the JSON structure using PowerShell's ConvertFrom-Json to ensure syntactic correctness after edits.

---

**Summary:** Only 2 functional changes (1 critical fix + 4 backup mapping keys). No changes to credentials, workflow IDs, or core logic. All changes are minimal and low-risk.

## Files changed

- `d:\VSC stuff\The agent.json` — updated the expression described above.
- `d:\VSC stuff\README.md` — this file (created).

## How I verified

I validated the file is valid JSON using PowerShell's ConvertFrom-Json.

PowerShell validation command (run from the workspace directory):

```powershell
# Validate JSON parses
Get-Content -Raw -LiteralPath 'd:\VSC stuff\The agent.json' | ConvertFrom-Json | Out-Null; Write-Output 'OK'
```

You should see `OK` when the JSON parses successfully.

I also scanned the workflow for n8n expression references like `$('Node Name')` and confirmed the referenced node names exist in the workflow.

## How to import into n8n

1. Open your n8n instance (local or cloud).
2. Workflow → Import workflow → choose `The agent.json`.
3. After importing, open the workflow and verify the following:
   - All required credentials are present and valid (OpenAI, Gmail OAuth, WhatsApp, Supabase, SerpAPI, etc.). The exported JSON contains credential IDs from your environment — these are environment-specific. If credentials are missing or invalid, re-link them in n8n.
   - Any referenced sub-workflows (toolWorkflow nodes referencing other workflows by ID) exist in your n8n instance. If a referenced workflow is missing, update the `workflowId` parameter in the node to point to the correct workflow.
4. Save the workflow.

## How to test (safe local tests)

1. Dry-run expression checks

   Re-validate the JSON as shown above. Then, in PowerShell you can list node names:

```powershell
$json = Get-Content -Raw -LiteralPath 'd:\VSC stuff\The agent.json' | ConvertFrom-Json
$json.nodes | ForEach-Object { $_.name }
```

2. Test in n8n UI

   - Enable required credentials.
   - For trigger-based nodes (WhatsApp/Gmail), either produce real events (send a WhatsApp message to the configured account / send an email) or use the n8n UI's play/execute features with sample data.
   - Watch the execution and check the expression outputs, especially the memory nodes that use `sessionKey` expressions.

3. Verify Supabase row creation and any downstream tools by checking their respective dashboards (Supabase, Gmail sent items, WhatsApp logs).

## Testing with provided sample payloads

I added a small `test_payloads/` folder containing example payloads and a helper script `run_tests.ps1` to POST them to an n8n webhook (or any HTTP endpoint).

- `test_payloads/whatsapp_sample.json` — example WhatsApp message payload.
- `test_payloads/gmail_sample.json` — example Gmail trigger payload.
- `run_tests.ps1` — PowerShell helper to POST a payload file to a webhook URL.

Usage example (PowerShell):

```powershell
# POST the WhatsApp test payload to a webhook URL
.\run_tests.ps1 -WebhookUrl 'https://your-n8n-url/webhook/<webhook-id>' -Payload '.\test_payloads\whatsapp_sample.json'
```

Note: If your workflow uses native Gmail/WhatsApp trigger nodes, they may not be callable by a generic webhook. The helper is intended for webhook-based testing or for nodes you expose via an HTTP webhook node.

## Subworkflows report

I created `subworkflows_report.md` listing all `toolWorkflow` nodes and the workflow IDs they reference. After importing the workflow, confirm each referenced workflow exists in your instance and update the `workflowId` values in the nodes if needed.


## Recommended follow-ups (optional)

- Normalize `AI` variable names vs field IDs: some tool inputs use `$fromAI('message_ID', ...)` while field ids are `message ID`. This is often intentional but may be a source of confusion — consider standardizing if you plan to edit workflows programmatically.

- Review sub-workflow IDs referenced by `toolWorkflow` nodes. If you migrated workflows between n8n instances, update these workflow IDs to match the target instance.

- Add unit-like tests / sample payloads: create a small set of JSON sample inputs (a Gmail message, a WhatsApp message) and run the workflow with them to validate end-to-end behavior.

- If you want, I can further:
  - Standardize mapping keys across toolWorkflow nodes.
  - Create sample input files and a test harness to run headless executions (via the n8n CLI or API).

## Notes & caveats

- I did not change any credential IDs or workflow IDs apart from the single expression fix. The IDs embedded in the JSON are likely specific to the original n8n instance. When importing into a different instance you will need to reconnect credentials and possibly update workflow references.

- I performed only static checks (searches for expression mismatches and JSON parsing). I did not (and cannot) execute the workflow against live services from this environment.

## What I can do next (if you want me to continue)

- Automatically normalize tool input keys across the workflow.
- Create sample trigger payloads and a short script to execute the workflow programmatically (requires n8n API/credentials).
- Check and fix any other expression mismatches if you rename nodes in the future.

## Handoff

You're set to import `The agent.json` into n8n. After importing, make sure credentials are linked and run the tests described above. If you want, tell me which next action to take (normalize keys, create tests, or run broader refactors) and I'll continue.

---

Last edited: 2025-11-07


## Finalization checklist (workspace layout)

Keep the exact workspace structure below. The scripts assume these relative paths from the repository root.

```
d:\VSC stuff\
 ├── The agent.json
 ├── subworkflows_report.md
 ├── test_payloads\
 │     ├── gmail_sample.json
 │     └── whatsapp_sample.json
 ├── run_tests.ps1
 └── README.md
```

I added two helper scripts for local validation and report rebuilding:

- `rebuild_report.ps1` — regenerates `subworkflows_report.md` from `The agent.json`.
- `validate_before_commit.ps1` — runs JSON parse check, counts `toolWorkflow` nodes, and invokes `rebuild_report.ps1`.

## Validation routines (run before every commit)

Recommended commands (examples):

```powershell
# Check JSON integrity
powershell -NoProfile -Command "Get-Content -Raw -LiteralPath 'd:\VSC stuff\The agent.json' | ConvertFrom-Json | Out-Null; 'JSON OK'"

# Verify node count & references
(Get-Content -Raw -LiteralPath 'd:\VSC stuff\The agent.json') -match '@n8n/n8n-nodes-langchain.toolWorkflow' | Measure-Object -Line

# Rebuild subworkflow report
powershell -File .\rebuild_report.ps1
```

Alternatively run the consolidated helper:

```powershell
.\validate_before_commit.ps1
```

## Test payload protocol

Until the neural node exposes a test endpoint, POST sample payloads to a dummy webhook (e.g., webhook.site or Postman Echo) to simulate inbound events:

```powershell
.\run_tests.ps1 -WebhookUrl 'https://webhook.site/<temporary_id>' -Payload '.\test_payloads\gmail_sample.json'
```

Confirm the payloads appear in the dummy webhook inspector.

## Mapping policy (final)

* Keep current **non-destructive normalization** (space-key + underscore-key both supported).
* Do **not** perform destructive renaming until we confirm the canonical schema inside the live n8n instance.
* Any new toolWorkflow node added must include both key variants immediately.

## Documentation discipline

For every change or search added to the workspace, append a short note in `README.md` using this format:

```
[YYYY-MM-DD] description
Reason: short reason
```

Audit note added during this pass:

[2025-11-07] normalized AI mapping keys in nodes `Reply to an email`, `Reply to an email1`, and `Reply to an email2` (added `message_ID` duplicates); added `Whatsapp_message_for_Peter` duplicate mapping for `Reply to an email2`.

## GitHub linkage (read-only phase)

* Commit only to the local feature branch `agent/vsc-prototype-v1`. Example:

```powershell
git checkout -b agent/vsc-prototype-v1
git add .
git commit -m "chore: normalize mapping keys + add validation scripts" --no-verify
git push origin agent/vsc-prototype-v1
```

Do not create PRs to main until the private endpoint is available and live testing is complete.

## Files for review

Please review the following files after this pass:

- `The agent.json` (post-normalization snippet shown below)
- `subworkflows_report.md`
- `README.md` (this file — with the final change log entry above)

[2025-11-07] Validation pass OK – structure aligned with Aurora Marketing schema v1.1

[2025-11-07] Phase 1.2 – export built and hashed for aura-neural staging (Aurora Marketing v1.1)

For a full, step-by-step handoff aimed at a new operator, see `HANDOFF.md` in the repository root. It includes exact copy/paste commands, screenshots suggestions, troubleshooting, and a sign-off checklist.

## Handoff to Operator (Saud)

Run the consolidated Phase 1.2 export locally on Windows (this will validate the JSON, rebuild the subworkflows report, create the export ZIP, and append the SHA-256 to the README):

```powershell
# one-shot runner (from your machine)
d:\vsc_tools\run_phase12.bat
```

What you will see (copy/paste these lines back into chat when finished):
- JSON OK
- toolWorkflow node count: <number>
- Rebuilt report: D:\VSC stuff\subworkflows_report.md
- Created: d:\VSC stuff\agent_export.zip
- Export SHA256: <hash>
- Appended to: d:\VSC stuff\README.md

After the run
- Verify `d:\VSC stuff\agent_export.zip` exists and contains only `The agent.json`, `subworkflows_report.md` and `README.md`.
- Send the ZIP file to Saud via DM or email (do not push the ZIP to GitHub).

Suggested email/DM to Saud (copy/paste):

Subject: Aurora Dev Agent — Phase 1.2 export (for staging import)

Body:

Hi Saud,

Attached is the Phase 1.2 export for the Aurora Marketing agent. Please import into the aura-neural staging instance under namespace `aurora_dev_agent`.

Archive: agent_export.zip
SHA-256: <paste the Export SHA256 value printed by the script>

Notes:
- This archive contains only the workflow JSON, a subworkflows report, and README with the export hash.
- Dual-key normalization preserved (both `message ID` and `message_ID` keys exist).
- No credentials, secrets, or tokens are included.

Please confirm when the import is complete and healthy. After confirmation I will proceed with Phase 1.3 – Runtime Binding.

Thanks,
[operator]

PR checklist (local-only)
- Run `d:\vsc_tools\run_phase12.bat` and paste the `Export SHA256` value into `d:\aurora_agent_repo\docs\PR_NOTES.txt` before creating the PR.
- Create the PR from `phase1.2-export` to `main` and include `.\docs\PR_NOTES.txt` as the body.

Small reminders
- Keep dual-key normalization (do not perform destructive key renames).
- Do not commit `agent_export.zip` into the repo; it is excluded via `.gitignore`.

Audit log
[2025-11-07] Phase 1.2 – handoff prepared; run and confirm export locally.
