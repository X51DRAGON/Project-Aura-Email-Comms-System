# Handoff Guide — Aurora Marketing Agent (Phase 1.2)

Purpose
-------
This document is a thorough, step-by-step handoff intended for someone with little or no prior knowledge of the project. Follow each numbered step exactly on a Windows machine with PowerShell (v5.1 or later). Do not run any networked services from this environment unless explicitly instructed.

Prerequisites
-------------
- Windows machine with PowerShell (v5.1) and Git installed.
- Optional: GitHub CLI (`gh`) for creating PRs.
- No credentials, tokens, or secrets are needed to run the local validation/export.
- Paths used in this guide assume the files live under `D:\VSC stuff` (original workspace) and `D:\vsc_tools` (helpers). Adjust paths if you use different locations.

Files in the workspace (what each file is for)
-------------------------------------------
- `D:\VSC stuff\The agent.json` — main n8n workflow JSON (import into n8n).
- `D:\VSC stuff\subworkflows_report.md` — auto-generated report listing `toolWorkflow` nodes and referenced workflow IDs.
- `D:\VSC stuff\README.md` — overview, verification commands, and short handoff notes.
- `D:\VSC stuff\HANDOFF.md` — this detailed handoff for a green operator.
- `D:\VSC stuff\rebuild_report.ps1` — regenerates `subworkflows_report.md` atomically.
- `D:\VSC stuff\validate_before_commit.ps1` — validates JSON, counts `toolWorkflow` nodes, and calls the rebuild script.
- `D:\VSC stuff\compress_export2.ps1` — creates `agent_export.zip` (non-interactive, .NET-based).
- `D:\VSC stuff\append_hash_to_readme.ps1` — computes SHA-256 and appends it to `README.md`.
- `D:\vsc_tools\phase12.ps1` & `d:\vsc_tools\run_phase12.bat` — consolidated runner that executes validation, rebuild, zip creation, and hash append from a path without spaces.
- `D:\aurora_agent_repo` — prepared repo scaffold (use `scripts\populate_repo.ps1` to copy files into it locally).

High-level workflow (what we will do)
------------------------------------
1. Validate the JSON and rebuild the subworkflows report.
2. Create an export zip containing only `The agent.json`, `subworkflows_report.md`, and `README.md`.
3. Compute a SHA-256 of the ZIP and append it to the README for audit.
4. Populate the clean local repo, commit to branch `phase1.2-export`, and create a private PR for review.
5. Send the zip to Saud (operator) by DM/email including the SHA-256.

Exact copy/paste steps (run in order)
------------------------------------
Open Windows PowerShell (as your normal user). Run these commands exactly.

1) Run the consolidated Phase 1.2 runner (one-shot)

```powershell
d:\vsc_tools\run_phase12.bat
```

What to expect (copy these lines into the PR comment or chat):
- JSON OK
- toolWorkflow node count: <number>
- Rebuilt report: D:\VSC stuff\subworkflows_report.md
- Created: d:\VSC stuff\agent_export.zip
- Export SHA256: <hash>
- Appended to: d:\VSC stuff\README.md

If the above command fails due to environment restrictions, run the steps manually:

2) Manual run — validation & rebuild

```powershell
# validate JSON and rebuild report
powershell -NoProfile -ExecutionPolicy Bypass -File "d:\VSC stuff\validate_before_commit.ps1"
```

3) Manual run — create zip

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "d:\VSC stuff\compress_export2.ps1"
```

4) Manual run — append hash

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "d:\VSC stuff\append_hash_to_readme.ps1"
```

Verify archive contents (important)
---------------------------------
Make a temporary folder and extract the archive to confirm it contains exactly the three files.

```powershell
# create temp folder
$temp = Join-Path $env:TEMP "agent_export_check"
Remove-Item -LiteralPath $temp -Recurse -Force -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Path $temp | Out-Null
Expand-Archive -LiteralPath "d:\VSC stuff\agent_export.zip" -DestinationPath $temp -Force
Get-ChildItem -Path $temp | Select-Object Name, Length
```

You should see only:
- The agent.json
- subworkflows_report.md
- README.md

Compute SHA-256 manually (if you'd like to double-check)

```powershell
Get-FileHash "d:\VSC stuff\agent_export.zip" -Algorithm SHA256 | Format-List
```

Repo preparation & PR (local-only)
---------------------------------
Use the prepared repo scaffold at `D:\aurora_agent_repo`.

1) Populate the repo with the actual files

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "d:\aurora_agent_repo\scripts\populate_repo.ps1"
```

2) Initialize, commit and set remote (edit script or pass -Remote)

```powershell
# edit the script if you need to change remote; otherwise pass it as parameter
powershell -NoProfile -ExecutionPolicy Bypass -File "d:\aurora_agent_repo\scripts\create_repo_and_push.ps1" -Remote "https://github.com/<your_github_username>/aurora-agent-vsc-prototype.git"
```

Notes:
- The script will create branch `phase1.2-export` and commit the prepared files.
- By default the script does NOT push to origin; uncomment the push lines after you confirm your remote and that you are authenticated.

Create PR
---------
If you have `gh` CLI, run (after pushing the branch):

```powershell
gh pr create --base main --head phase1.2-export --title "Aurora Agent – Phase 1.2 export" --body-file ".\docs\PR_NOTES.txt"
```

Otherwise open the browser to:

```
https://github.com/<your_github_username>/aurora-agent-vsc-prototype/compare/main...phase1.2-export?expand=1
```
and paste the contents of `.\docs\PR_NOTES.txt` as the PR body (remember to replace `<PASTE_HASH_HERE>` with the actual SHA-256).

How to send the ZIP to Saud (operator)
-------------------------------------
Email or DM the file (do not upload to GitHub). Suggested content (copy/paste):

Subject: Aurora Dev Agent — Phase 1.2 export (for staging import)

Body:

Hi Saud,

Attached is the Phase 1.2 export for the Aurora Marketing agent. Please import into the aura-neural staging instance under namespace `aurora_dev_agent`.

Archive: agent_export.zip
SHA-256: <paste the Export SHA256 value printed by the script>

Notes:
- Archive contains only workflow JSON, a subworkflows report, and README with the export hash.
- Dual-key normalization preserved (both `message ID` and `message_ID` are present where needed).
- No credentials or secrets are included.

Please confirm when the import completes and whether everything looks healthy.

Best,
[your name]

Troubleshooting (common issues & fixes)
-------------------------------------
- Problem: PowerShell command splits or prompts with `Path[..]` errors when run from the remote runner.
  Fix: Use the batch shim at `d:\vsc_tools\run_phase12.bat` (no spaces path), or run the individual scripts manually using the quoted `-File` form.

- Problem: `Add-Content : Stream was not readable` when regenerating the report.
  Fix: We replaced streaming writes with atomic temp-file writes in `rebuild_report.ps1`. Ensure the script is the updated version. If you still see this, ensure no other process has the file open and re-run.

- Problem: ZIP missing expected files.
  Fix: Check the compress script (`compress_export2.ps1`) and verify the source files exist at their expected paths before running. Use `Get-ChildItem` to confirm.

- Problem: `toolWorkflow` nodes not found or count mismatch.
  Fix: `validate_before_commit.ps1` counts literal occurrences of the node type. If you edited `The agent.json` manually, re-run the script. If nodes were intentionally added/removed, document the change.

Developer notes (for future maintainers)
--------------------------------------
- Dual-key normalization policy: Keep both `space` and `underscore` variants when mapping AI outputs to tool inputs (non-destructive approach). When adding new toolWorkflow nodes, include both variants.
- Do not change workflow IDs in `toolWorkflow` nodes without coordinating with the target n8n instance; those IDs are instance-specific.
- When editing `The agent.json`, always run `validate_before_commit.ps1` before committing.

Minimal acceptance checklist (operator follow these and reply in chat)
-------------------------------------------------------------------
1. Ran `d:\vsc_tools\run_phase12.bat`.
2. Confirmed console showed: `JSON OK`, `toolWorkflow node count: <n>`, `Rebuilt report: ...`, `Created: ...`, `Export SHA256: ...`.
3. Confirmed ZIP contents (exactly 3 files).
4. Confirmed SHA-256 appended to `README.md`.
5. Sent zip to Saud with the SHA-256 and got confirmation (reply here with Saud's confirmation).

If all the above are true, mark Phase 1.2 as complete and await Phase 1.3 instructions.

Contact & handover
------------------
If you get stuck, paste the exact console output here and I'll help debug. Keep the workspace local-only and do not post the zip publicly.

---

Last edited: 2025-11-07
