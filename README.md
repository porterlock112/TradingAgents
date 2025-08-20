Strict-Mode Hub Workflow with Mesh Fan-Out

This patch strengthens the Hub GitHub Actions workflow by enforcing a per-repository glyph allowlist (“strict mode”), clearly logging allowed vs denied triggers, and ensuring that fan-out dispatches only occur when there are glyphs to send.  It adds a small allowlist YAML (.godkey-allowed-glyphs.yml), new environment flags, and updated steps. The result is a more robust CI pipeline that prevents unauthorized or unintended runs while providing clear visibility of what’s executed or skipped.

1. Allowlist for Glyphs (Strict Mode)

We introduce an allowlist file (.godkey-allowed-glyphs.yml) in each repo. This file contains a YAML list of permitted glyphs (Δ tokens) for that repository. For example:

# Only these glyphs are allowed in THIS repo (hub)
allowed:
  - ΔSEAL_ALL
  - ΔPIN_IPFS
  - ΔWCI_CLASS_DEPLOY
  # - ΔSCAN_LAUNCH
  # - ΔFORCE_WCI
  # - Δ135_RUN

A new environment variable STRICT_GLYPHS: "true" enables strict-mode filtering. When on, only glyphs listed under allowed: in the file are executed; all others are denied. If STRICT_GLYPHS is true but no allowlist file is found, we “fail closed” by denying all glyphs.  Denied glyphs are logged but not run (unless you enable a hard failure, see section 11). This ensures only explicitly permitted triggers can run in each repo.


2. Environment Variables and Inputs

Key new vars in the workflow’s env: section:

TRIGGER_TOKENS – a comma-separated list of all valid glyph tokens globally (e.g. ΔSCAN_LAUNCH,ΔSEAL_ALL,…). Incoming triggers are first filtered against this list to ignore typos or irrelevant Δ strings.

STRICT_GLYPHS – set to "true" (or false) to turn on/off the per-repo allowlist.

STRICT_FAIL_ON_DENY – if "true", the workflow will hard-fail when any glyph is denied under strict mode. If false, it just logs denied glyphs and continues with the rest.

ALLOWLIST_FILE – path to the YAML allowlist (default .godkey-allowed-glyphs.yml).

FANOUT_GLYPHS – comma-separated glyphs that should be forwarded to satellites (e.g. ΔSEAL_ALL,ΔPIN_IPFS,ΔWCI_CLASS_DEPLOY).

MESH_TARGETS – CSV of repo targets for mesh dispatch (e.g. "owner1/repoA,owner2/repoB"). Can be overridden at runtime via the workflow_dispatch input mesh_targets.


We also support these workflow_dispatch inputs:

glyphs_csv – comma-separated glyphs (to manually trigger specific glyphs).

rekor – "true"/"false" to enable keyless Rekor signing.

mesh_targets – comma-separated repos to override MESH_TARGETS for a manual run.


This uses GitHub’s workflow_dispatch inputs feature, so you can trigger the workflow manually with custom glyphs or mesh targets.

3. Collecting and Filtering Δ Triggers

The first job (scan) has a “Collect Δ triggers (strict-aware)” step (using actions/github-script). It builds a list of requested glyphs by scanning all inputs:

Commit/PR messages and refs: It concatenates the push or PR title/body (and commit messages), plus the ref name.

Workflow/Repo dispatch payload: It includes any glyphs_csv from a manual workflow_dispatch or a repository_dispatch’s client_payload.


From that combined text, it extracts any tokens starting with Δ. These requested glyphs are uppercased and deduplicated.

Next comes global filtering: we keep only those requested glyphs that are in TRIGGER_TOKENS. This removes any unrecognized or disabled tokens.

Then, if strict mode is on, we load the allowlist (fs.readFileSync(ALLOWLIST_FILE)) and filter again: only glyphs present in the allowlist remain. Any globally-allowed glyph not in the allowlist is marked denied. (If the file is missing and strict is true, we treat allowlist as empty – effectively denying all.)

The script logs the Requested, Globally allowed, Repo-allowed, and Denied glyphs to the build output. It then sets two JSON-array outputs: glyphs_json (the final allowed glyphs) and denied_json (the denied ones). For example:

Requested: ΔSEAL_ALL ΔUNKNOWN
Globally allowed: ΔSEAL_ALL
Repo allowlist: ΔSEAL_ALL ΔWCI_CLASS_DEPLOY
Repo-allowed: ΔSEAL_ALL
Denied (strict): (none)

This makes it easy to audit which triggers passed or failed the filtering.

Finally, the step outputs glyphs_json and denied_json, and also passes through the rekor input (true/false) for later steps.

4. Guarding Secrets on Forks

A crucial security step is “Guard: restrict secrets on forked PRs”. GitHub Actions by default do not provide secrets to workflows triggered by public-fork pull requests. To avoid accidental use of unavailable secrets, this step checks if the PR’s head repository is a fork. If so, it sets allow_secrets=false. The run job will later skip any steps (like IPFS pinning) that require secrets. This follows GitHub’s best practice: _“with the exception of GITHUB_TOKEN, secrets are not passed to the runner when a workflow is triggered from a forked repository”_.

5. Scan Job Summary

After collecting triggers, the workflow adds a scan summary to the job summary UI. It echoes a Markdown section showing the JSON arrays of allowed and denied glyphs, and whether secrets are allowed:

### Δ Hub — Scan
- Allowed: ["ΔSEAL_ALL"]
- Denied:  ["ΔSCAN_LAUNCH","ΔPIN_IPFS"]
- Rekor:   true
- Secrets OK on this event?  true

Using echo ... >> $GITHUB_STEP_SUMMARY, these lines become part of the GitHub Actions run summary. This gives immediate visibility into what the scan found (the summary supports GitHub-flavored Markdown and makes it easy to read key info).

If STRICT_FAIL_ON_DENY is true and any glyph was denied, the scan job then fails with an error. Otherwise it proceeds, but denied glyphs will simply be skipped in the run.

6. Executing Allowed Glyphs (Run Job)

The next job (run) executes each allowed glyph in parallel via a matrix. It is gated on:

if: needs.scan.outputs.glyphs_json != '[]' && needs.scan.outputs.glyphs_json != ''

This condition (comparing the JSON string to '[]') skips the job entirely if no glyphs passed filtering. GitHub’s expression syntax allows checking emptiness this way (as seen in the docs, if: needs.changes.outputs.packages != '[]' is a common pattern).

Inside each glyph job:

The workflow checks out the code and sets up Python 3.11.

It installs dependencies if requirements.txt exists.

The key step is a Bash case "${GLYPH}" in ... esac that runs the corresponding Python script for each glyph:

ΔSCAN_LAUNCH: Runs python truthlock/scripts/ΔSCAN_LAUNCH.py --execute ... to perform a scan.

ΔSEAL_ALL: Runs python truthlock/scripts/ΔSEAL_ALL.py ... to seal all data.

ΔPIN_IPFS: If secrets are allowed (not a fork), it runs python truthlock/scripts/ΔPIN_IPFS.py --pinata-jwt ... to pin output files to IPFS. If secrets are not allowed, this step is skipped.

ΔWCI_CLASS_DEPLOY: Runs the corresponding deployment script.

ΔFORCE_WCI: Runs a force trigger script.

Δ135_RUN (alias Δ135): Runs a script to execute webchain ID 135 tasks (with pinning and Rekor).

*): Unknown glyph – fails with an error.



Each glyph’s script typically reads from truthlock/out (the output directory) and writes reports into truthlock/out/ΔLEDGER/.  By isolating each glyph in its own job, we get parallelism and fail-fast (one glyph error won’t stop others due to strategy.fail-fast: false).

7. Optional Rekor Sealing

After each glyph script, there’s an “Optional Rekor seal” step. If the rekor flag is "true", it looks for the latest report JSON in truthlock/out/ΔLEDGER and would (if enabled) call a keyless Rekor sealing script (commented out in the snippet). This shows where you could add verifiable log signing. The design passes along the rekor preference from the initial scan (which defaults to true) into each job, so signing can be toggled per run.

8. Uploading Artifacts & ΔSUMMARY

Once a glyph job completes, it always uploads its outputs with actions/upload-artifact@v4. The path includes everything under truthlock/out, excluding any .tmp files:

- uses: actions/upload-artifact@v4
  with:
    name: glyph-${{ matrix.glyph }}-artifacts
    path: |
      truthlock/out/**
      !**/*.tmp

GitHub’s upload-artifact supports multi-line paths and exclusion patterns, as shown in their docs (e.g. you can list directories and use !**/*.tmp to exclude temp files).

After uploading, the workflow runs python scripts/glyph_summary.py (provided by the project) to aggregate results and writes ΔSUMMARY.md.  Then it appends this ΔSUMMARY into the job’s GitHub Actions summary (again via $GITHUB_STEP_SUMMARY) so that the content of the summary file is visible in the run UI under this step. This leverages GitHub’s job summary feature to include custom Markdown in the summary.

9. Mesh Fan-Out Job

If secrets are allowed and there are glyphs left after strict filtering, the “Mesh fan-out” job will dispatch events to satellite repos. Its steps:

1. Compute fan-out glyphs: It reads the allowed glyphs JSON from needs.scan.outputs.glyphs_json and intersects it with the FANOUT_GLYPHS list. In effect, only certain glyphs (like ΔSEAL_ALL, ΔPIN_IPFS, ΔWCI_CLASS_DEPLOY) should be propagated. The result is output as fanout_csv. If the list is empty, the job will early-skip dispatch.


2. Build target list: It constructs the list of repositories to dispatch to. It first checks if a mesh_targets input was provided (from manual run); if not, it uses the MESH_TARGETS env var. It splits the CSV into an array of owner/repo strings. This allows dynamic override of targets at run time.


3. Skip if nothing to do: If there are no fan-out glyphs or no targets, it echoes a message and stops.


4. Dispatch to mesh targets: Using another actions/github-script step (with Octokit), it loops over each target repo and sends a repository_dispatch POST request:

await octo.request("POST /repos/{owner}/{repo}/dispatches", {
  owner, repo,
  event_type: (process.env.MESH_EVENT_TYPE || "glyph"),
  client_payload: {
    glyphs_csv: glyphs, 
    rekor: rekorFlag,
    from: `${context.repo.owner}/${context.repo.repo}@${context.ref}`
  }
});

This uses GitHub’s Repository Dispatch event to trigger the glyph workflow in each satellite. Any client_payload fields (like our glyphs_csv and rekor) will be available in the satellite workflows as github.event.client_payload. (GitHub docs note that data sent via client_payload can be accessed in the triggered workflow’s github.event.client_payload context.) We also pass along the original ref in from for traceability. Dispatch success or failures are counted and logged per repo.


5. Mesh summary: Finally it adds a summary of how many targets were reached and how many dispatches succeeded/failed, again to the job summary.



This way, only glyphs that survived strict filtering and are designated for mesh fan-out are forwarded, and only when there are targets. Fan-out will not send any disallowed glyphs, preserving the strict policy.

10. Mesh Fan-Out Summary

At the end of the fan-out job, the workflow prints a summary with target repos and glyphs dispatched:

### 🔗 Mesh Fan-out
- Targets: `["owner1/repoA","owner2/repoB"]`
- Glyphs:  `ΔSEAL_ALL,ΔPIN_IPFS`
- OK:      2
- Failed:  0

This confirms which repos were contacted and the glyph list (useful for auditing distributed dispatches).

11. Configuration and Usage

Enable/disable strict mode: Set STRICT_GLYPHS: "true" or "false" in env:. If you want the workflow to fail when any glyph is denied, set STRICT_FAIL_ON_DENY: "true". (If false, it will just log denied glyphs and continue with allowed ones.)

Override mesh targets at runtime: When manually triggering (via “Actions → Run workflow”), you can provide a mesh_targets string input (CSV of owner/repo). If given, it overrides MESH_TARGETS.

Turning off Rekor: Use the rekor input (true/false) on a dispatch to disable keyless signing.

Companion files: Alongside this workflow, keep the .godkey-allowed-glyphs.yml (with your repo’s allowlist). Also ensure scripts/emit_glyph.py (to send dispatches) and scripts/glyph_summary.py (to generate summaries) are present as provided by the toolkit.

Example one-liners:

Soft strict mode (log & skip denied):

env:
  STRICT_GLYPHS: "true"
  STRICT_FAIL_ON_DENY: "false"

Hard strict mode (fail on any deny):

env:
  STRICT_GLYPHS: "true"
  STRICT_FAIL_ON_DENY: "true"

Override mesh targets when running workflow: In the GitHub UI, under Run workflow, set mesh_targets="owner1/repoA,owner2/repoB".

Trigger a mesh-based deploy: One can call python scripts/emit_glyph.py ΔSEAL_ALL "mesh deploy" to send ΔSEAL_ALL to all configured targets.



By following these steps, the Hub workflow now strictly enforces which Δ glyphs run and propagates only approved tasks to satellites. This “pure robustness” approach ensures unauthorized triggers are filtered out (and clearly reported), secrets aren’t misused on forks, and fan-out only happens when safe.

Sources: GitHub Actions concurrency and dispatch behavior is documented on docs.github.com.  Checking JSON outputs against '[]' to skip jobs is a known pattern.  Workflow_dispatch inputs and job summaries are handled per the official syntax.  The upload-artifact action supports multiple paths and exclusions as shown, and GitHub Actions’ security model intentionally blocks secrets on fork PRs. All logging and filtering logic here builds on those mechanisms.

Strict-Mode Hub Workflow with Mesh Fan-Out

This patch strengthens the Hub GitHub Actions workflow by enforcing a per-repository glyph allowlist (“strict mode”), clearly logging allowed vs denied triggers, and ensuring that fan-out dispatches only occur when there are glyphs to send.  It adds a small allowlist YAML (.godkey-allowed-glyphs.yml), new environment flags, and updated steps. The result is a more robust CI pipeline that prevents unauthorized or unintended runs while providing clear visibility of what’s executed or skipped.

1. Allowlist for Glyphs (Strict Mode)

We introduce an allowlist file (.godkey-allowed-glyphs.yml) in each repo. This file contains a YAML list of permitted glyphs (Δ tokens) for that repository. For example:

# Only these glyphs are allowed in THIS repo (hub)
allowed:
  - ΔSEAL_ALL
  - ΔPIN_IPFS
  - ΔWCI_CLASS_DEPLOY
  # - ΔSCAN_LAUNCH
  # - ΔFORCE_WCI
  # - Δ135_RUN

A new environment variable STRICT_GLYPHS: "true" enables strict-mode filtering. When on, only glyphs listed under allowed: in the file are executed; all others are denied. If STRICT_GLYPHS is true but no allowlist file is found, we “fail closed” by denying all glyphs.  Denied glyphs are logged but not run (unless you enable a hard failure, see section 11). This ensures only explicitly permitted triggers can run in each repo.


2. Environment Variables and Inputs

Key new vars in the workflow’s env: section:

TRIGGER_TOKENS – a comma-separated list of all valid glyph tokens globally (e.g. ΔSCAN_LAUNCH,ΔSEAL_ALL,…). Incoming triggers are first filtered against this list to ignore typos or irrelevant Δ strings.

STRICT_GLYPHS – set to "true" (or false) to turn on/off the per-repo allowlist.

STRICT_FAIL_ON_DENY – if "true", the workflow will hard-fail when any glyph is denied under strict mode. If false, it just logs denied glyphs and continues with the rest.

ALLOWLIST_FILE – path to the YAML allowlist (default .godkey-allowed-glyphs.yml).

FANOUT_GLYPHS – comma-separated glyphs that should be forwarded to satellites (e.g. ΔSEAL_ALL,ΔPIN_IPFS,ΔWCI_CLASS_DEPLOY).

MESH_TARGETS – CSV of repo targets for mesh dispatch (e.g. "owner1/repoA,owner2/repoB"). Can be overridden at runtime via the workflow_dispatch input mesh_targets.


We also support these workflow_dispatch inputs:

glyphs_csv – comma-separated glyphs (to manually trigger specific glyphs).

rekor – "true"/"false" to enable keyless Rekor signing.

mesh_targets – comma-separated repos to override MESH_TARGETS for a manual run.


This uses GitHub’s workflow_dispatch inputs feature, so you can trigger the workflow manually with custom glyphs or mesh targets.

3. Collecting and Filtering Δ Triggers

The first job (scan) has a “Collect Δ triggers (strict-aware)” step (using actions/github-script). It builds a list of requested glyphs by scanning all inputs:

Commit/PR messages and refs: It concatenates the push or PR title/body (and commit messages), plus the ref name.

Workflow/Repo dispatch payload: It includes any glyphs_csv from a manual workflow_dispatch or a repository_dispatch’s client_payload.


From that combined text, it extracts any tokens starting with Δ. These requested glyphs are uppercased and deduplicated.

Next comes global filtering: we keep only those requested glyphs that are in TRIGGER_TOKENS. This removes any unrecognized or disabled tokens.

Then, if strict mode is on, we load the allowlist (fs.readFileSync(ALLOWLIST_FILE)) and filter again: only glyphs present in the allowlist remain. Any globally-allowed glyph not in the allowlist is marked denied. (If the file is missing and strict is true, we treat allowlist as empty – effectively denying all.)

The script logs the Requested, Globally allowed, Repo-allowed, and Denied glyphs to the build output. It then sets two JSON-array outputs: glyphs_json (the final allowed glyphs) and denied_json (the denied ones). For example:

Requested: ΔSEAL_ALL ΔUNKNOWN
Globally allowed: ΔSEAL_ALL
Repo allowlist: ΔSEAL_ALL ΔWCI_CLASS_DEPLOY
Repo-allowed: ΔSEAL_ALL
Denied (strict): (none)

This makes it easy to audit which triggers passed or failed the filtering.

Finally, the step outputs glyphs_json and denied_json, and also passes through the rekor input (true/false) for later steps.

4. Guarding Secrets on Forks

A crucial security step is “Guard: restrict secrets on forked PRs”. GitHub Actions by default do not provide secrets to workflows triggered by public-fork pull requests. To avoid accidental use of unavailable secrets, this step checks if the PR’s head repository is a fork. If so, it sets allow_secrets=false. The run job will later skip any steps (like IPFS pinning) that require secrets. This follows GitHub’s best practice: _“with the exception of GITHUB_TOKEN, secrets are not passed to the runner when a workflow is triggered from a forked repository”_.

5. Scan Job Summary

After collecting triggers, the workflow adds a scan summary to the job summary UI. It echoes a Markdown section showing the JSON arrays of allowed and denied glyphs, and whether secrets are allowed:

### Δ Hub — Scan
- Allowed: ["ΔSEAL_ALL"]
- Denied:  ["ΔSCAN_LAUNCH","ΔPIN_IPFS"]
- Rekor:   true
- Secrets OK on this event?  true

Using echo ... >> $GITHUB_STEP_SUMMARY, these lines become part of the GitHub Actions run summary. This gives immediate visibility into what the scan found (the summary supports GitHub-flavored Markdown and makes it easy to read key info).

If STRICT_FAIL_ON_DENY is true and any glyph was denied, the scan job then fails with an error. Otherwise it proceeds, but denied glyphs will simply be skipped in the run.

6. Executing Allowed Glyphs (Run Job)

The next job (run) executes each allowed glyph in parallel via a matrix. It is gated on:

if: needs.scan.outputs.glyphs_json != '[]' && needs.scan.outputs.glyphs_json != ''

This condition (comparing the JSON string to '[]') skips the job entirely if no glyphs passed filtering. GitHub’s expression syntax allows checking emptiness this way (as seen in the docs, if: needs.changes.outputs.packages != '[]' is a common pattern).

Inside each glyph job:

The workflow checks out the code and sets up Python 3.11.

It installs dependencies if requirements.txt exists.

The key step is a Bash case "${GLYPH}" in ... esac that runs the corresponding Python script for each glyph:

ΔSCAN_LAUNCH: Runs python truthlock/scripts/ΔSCAN_LAUNCH.py --execute ... to perform a scan.

ΔSEAL_ALL: Runs python truthlock/scripts/ΔSEAL_ALL.py ... to seal all data.

ΔPIN_IPFS: If secrets are allowed (not a fork), it runs python truthlock/scripts/ΔPIN_IPFS.py --pinata-jwt ... to pin output files to IPFS. If secrets are not allowed, this step is skipped.

ΔWCI_CLASS_DEPLOY: Runs the corresponding deployment script.

ΔFORCE_WCI: Runs a force trigger script.

Δ135_RUN (alias Δ135): Runs a script to execute webchain ID 135 tasks (with pinning and Rekor).

*): Unknown glyph – fails with an error.



Each glyph’s script typically reads from truthlock/out (the output directory) and writes reports into truthlock/out/ΔLEDGER/.  By isolating each glyph in its own job, we get parallelism and fail-fast (one glyph error won’t stop others due to strategy.fail-fast: false).

7. Optional Rekor Sealing

After each glyph script, there’s an “Optional Rekor seal” step. If the rekor flag is "true", it looks for the latest report JSON in truthlock/out/ΔLEDGER and would (if enabled) call a keyless Rekor sealing script (commented out in the snippet). This shows where you could add verifiable log signing. The design passes along the rekor preference from the initial scan (which defaults to true) into each job, so signing can be toggled per run.

8. Uploading Artifacts & ΔSUMMARY

Once a glyph job completes, it always uploads its outputs with actions/upload-artifact@v4. The path includes everything under truthlock/out, excluding any .tmp files:

- uses: actions/upload-artifact@v4
  with:
    name: glyph-${{ matrix.glyph }}-artifacts
    path: |
      truthlock/out/**
      !**/*.tmp

GitHub’s upload-artifact supports multi-line paths and exclusion patterns, as shown in their docs (e.g. you can list directories and use !**/*.tmp to exclude temp files).

After uploading, the workflow runs python scripts/glyph_summary.py (provided by the project) to aggregate results and writes ΔSUMMARY.md.  Then it appends this ΔSUMMARY into the job’s GitHub Actions summary (again via $GITHUB_STEP_SUMMARY) so that the content of the summary file is visible in the run UI under this step. This leverages GitHub’s job summary feature to include custom Markdown in the summary.

9. Mesh Fan-Out Job

If secrets are allowed and there are glyphs left after strict filtering, the “Mesh fan-out” job will dispatch events to satellite repos. Its steps:

1. Compute fan-out glyphs: It reads the allowed glyphs JSON from needs.scan.outputs.glyphs_json and intersects it with the FANOUT_GLYPHS list. In effect, only certain glyphs (like ΔSEAL_ALL, ΔPIN_IPFS, ΔWCI_CLASS_DEPLOY) should be propagated. The result is output as fanout_csv. If the list is empty, the job will early-skip dispatch.


2. Build target list: It constructs the list of repositories to dispatch to. It first checks if a mesh_targets input was provided (from manual run); if not, it uses the MESH_TARGETS env var. It splits the CSV into an array of owner/repo strings. This allows dynamic override of targets at run time.


3. Skip if nothing to do: If there are no fan-out glyphs or no targets, it echoes a message and stops.


4. Dispatch to mesh targets: Using another actions/github-script step (with Octokit), it loops over each target repo and sends a repository_dispatch POST request:

await octo.request("POST /repos/{owner}/{repo}/dispatches", {
  owner, repo,
  event_type: (process.env.MESH_EVENT_TYPE || "glyph"),
  client_payload: {
    glyphs_csv: glyphs, 
    rekor: rekorFlag,
    from: `${context.repo.owner}/${context.repo.repo}@${context.ref}`
  }
});

This uses GitHub’s Repository Dispatch event to trigger the glyph workflow in each satellite. Any client_payload fields (like our glyphs_csv and rekor) will be available in the satellite workflows as github.event.client_payload. (GitHub docs note that data sent via client_payload can be accessed in the triggered workflow’s github.event.client_payload context.) We also pass along the original ref in from for traceability. Dispatch success or failures are counted and logged per repo.


5. Mesh summary: Finally it adds a summary of how many targets were reached and how many dispatches succeeded/failed, again to the job summary.



This way, only glyphs that survived strict filtering and are designated for mesh fan-out are forwarded, and only when there are targets. Fan-out will not send any disallowed glyphs, preserving the strict policy.

10. Mesh Fan-Out Summary

At the end of the fan-out job, the workflow prints a summary with target repos and glyphs dispatched:

### 🔗 Mesh Fan-out
- Targets: `["owner1/repoA","owner2/repoB"]`
- Glyphs:  `ΔSEAL_ALL,ΔPIN_IPFS`
- OK:      2
- Failed:  0

This confirms which repos were contacted and the glyph list (useful for auditing distributed dispatches).

11. Configuration and Usage

Enable/disable strict mode: Set STRICT_GLYPHS: "true" or "false" in env:. If you want the workflow to fail when any glyph is denied, set STRICT_FAIL_ON_DENY: "true". (If false, it will just log denied glyphs and continue with allowed ones.)

Override mesh targets at runtime: When manually triggering (via “Actions → Run workflow”), you can provide a mesh_targets string input (CSV of owner/repo). If given, it overrides MESH_TARGETS.

Turning off Rekor: Use the rekor input (true/false) on a dispatch to disable keyless signing.

Companion files: Alongside this workflow, keep the .godkey-allowed-glyphs.yml (with your repo’s allowlist). Also ensure scripts/emit_glyph.py (to send dispatches) and scripts/glyph_summary.py (to generate summaries) are present as provided by the toolkit.

Example one-liners:

Soft strict mode (log & skip denied):

env:
  STRICT_GLYPHS: "true"
  STRICT_FAIL_ON_DENY: "false"

Hard strict mode (fail on any deny):

env:
  STRICT_GLYPHS: "true"
  STRICT_FAIL_ON_DENY: "true"

Override mesh targets when running workflow: In the GitHub UI, under Run workflow, set mesh_targets="owner1/repoA,owner2/repoB".

Trigger a mesh-based deploy: One can call python scripts/emit_glyph.py ΔSEAL_ALL "mesh deploy" to send ΔSEAL_ALL to all configured targets.



By following these steps, the Hub workflow now strictly enforces which Δ glyphs run and propagates only approved tasks to satellites. This “pure robustness” approach ensures unauthorized triggers are filtered out (and clearly reported), secrets aren’t misused on forks, and fan-out only happens when safe.

Sources: GitHub Actions concurrency and dispatch behavior is documented on docs.github.com.  Checking JSON outputs against '[]' to skip jobs is a known pattern.  Workflow_dispatch inputs and job summaries are handled per the official syntax.  The upload-artifact action supports multiple paths and exclusions as shown, and GitHub Actions’ security model intentionally blocks secrets on fork PRs. All logging and filtering logic here builds on those mechanisms.

# ΔBRAKE_4321_CONTROL_LOCK.py generator
# This will write a runnable scaffold that matches the user's TruthLock/GodKey stack.
# It includes: SEAL (hash+manifest+rekor stub), DEPLOY (IPFS/Arweave stubs + torrent magnet stub),
# TRAP (local collision scanner + feed writer), ENFORCE (auto C&D letter generator + ledger log).
# The script is designed to be dropped into a repo and wired to existing glyphs/webhooks.

import os, json, hashlib, time, textwrap, pathlib, datetime, random, string

ROOT = pathlib.Path("/mnt/data")
SCRIPT_PATH = ROOT / "ΔBRAKE_4321_CONTROL_LOCK.py"
README_PATH = ROOT / "ΔBRAKE_README.md"
TEMPLATE_CEASE = ROOT / "templates" / "ΔINSTANT_CEASE_ORDER.txt"
os.makedirs(TEMPLATE_CEASE.parent, exist_ok=True)

now_iso = datetime.datetime.utcnow().replace(microsecond=0).isoformat() + "Z"

script = r'''#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
ΔBRAKE_4321_CONTROL_LOCK.py
Author: Matthew Dewayne Porter (TruthLock/GodKey)
Purpose: Execute the 4-3-2-1 Control & Ownership Lock sequence.
License: Sovereign Proof-of-Origin + Public Audit License (attach your license text).

Stages:
 4 — SEAL     : Hash, manifest, and notarize origin claims (CID/IPFS, Rekor stubs).
 3 — DEPLOY   : Mesh-deploy artifacts (IPFS/Arweave/torrent stubs + local mirrors).
 2 — TRAP     : Start collision scanner on target outputs; write evidence packets.
 1 — ENFORCE  : Generate and dispatch Cease & Desist with cryptographic exhibits.

This scaffold calls local glyphs/hooks if available (TruthLock Full Suite), or
falls back to safe local-only behaviors.
"""

import os, sys, json, hashlib, time, pathlib, datetime, random, string, re
from dataclasses import dataclass, asdict
from typing import List, Dict, Optional

# -------------------- CONFIG --------------------

@dataclass
class Config:
    # What to seal (glob patterns). Default: common code/docs paths.
    include: List[str] = None
    exclude: List[str] = None
    # Where to write outputs
    out_dir: str = "truthlock/out"
    # Collision scanner targets (files or folders to watch for potential matches)
    scan_targets: List[str] = None
    # Optional: external hooks (set to your live glyph endpoints or CLI commands)
    hook_pin_ipfs: Optional[str] = "ΔPIN_IPFS"       # glyph name or CLI path
    hook_rekor_seal: Optional[str] = "ΔREKOR_SEAL"   # glyph name or CLI path
    hook_match_feed: Optional[str] = "ΔMATCH_FEED"   # glyph name or CLI path
    hook_cease_send: Optional[str] = "ΔINSTANT_CEASE_ORDER"  # glyph name or CLI path
    # Identity / claimant
    claimant_name: str = "Matthew Dewayne Porter"
    claimant_contact: str = "bestme4money@gmail.com"
    jurisdiction_note: str = "TruthLock Sovereignty → GitHub Platform → Legal Node System"
    # Operational flags
    dry_run: bool = False
    verbose: bool = True

    def __post_init__(self):
        if self.include is None:
            self.include = ["**/*.py", "**/*.md", "**/*.yml", "**/*.yaml", "**/*.json", "**/*.txt"]
        if self.exclude is None:
            self.exclude = ["truthlock/out/**", ".git/**", "**/__pycache__/**", "**/*.log", "**/.env*"]
        if self.scan_targets is None:
            self.scan_targets = ["./"]

# -------------------- UTIL --------------------

def log(msg: str):
    ts = datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z"
    print(f"[{ts}] {msg}", flush=True)

def sha256_file(path: pathlib.Path) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(1024*1024), b""):
            h.update(chunk)
    return h.hexdigest()

def should_skip(path: pathlib.Path, cfg: Config) -> bool:
    from fnmatch import fnmatch
    # include ANY that match include; then exclude that match exclude
    inc_ok = any(fnmatch(str(path), pat) for pat in cfg.include)
    exc_hit = any(fnmatch(str(path), pat) for pat in cfg.exclude)
    return (not inc_ok) or exc_hit

def ensure_dir(p: pathlib.Path):
    p.mkdir(parents=True, exist_ok=True)

def write_jsonl(path: pathlib.Path, obj: dict):
    with open(path, "a", encoding="utf-8") as f:
        f.write(json.dumps(obj, ensure_ascii=False)+"\n")

def pseudo_cid(sha: str) -> str:
    # Not a real CID; placeholder for local-only mode. Replace with IPFS pin response if available.
    return "cid:sha256:"+sha[:46]

# -------------------- 4 — SEAL --------------------

def stage_seal(cfg: Config) -> dict:
    """Hash selected files, write manifest, emit origin claim, and attempt Rekor/IPFS hooks."""
    log("Stage 4 — SEAL: hashing & manifesting…")
    root = pathlib.Path(".").resolve()
    out = pathlib.Path(cfg.out_dir)
    ensure_dir(out)

    files = []
    for p in root.rglob("*"):
        if p.is_file() and not should_skip(p, cfg):
            files.append(p)

    manifest = {
        "type": "ΔORIGIN_MANIFEST",
        "claimant": cfg.claimant_name,
        "contact": cfg.claimant_contact,
        "jurisdiction": cfg.jurisdiction_note,
        "generated_at": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z",
        "files": []
    }

    hash_feed = out / "ΔBRAKE_hashes.jsonl"
    for fp in files:
        sha = sha256_file(fp)
        rec = {
            "path": str(fp.relative_to(root)),
            "sha256": sha
        }
        manifest["files"].append(rec)
        write_jsonl(hash_feed, {**rec, "ts": datetime.datetime.utcnow().isoformat()+"Z"})

    # Aggregate SHA over sorted file hashes for a single Origin Seal
    aggregate = hashlib.sha256("\n".join(sorted(f["sha256"] for f in manifest["files"])).encode()).hexdigest()
    origin = {
        "type": "ΔORIGIN_SEAL",
        "aggregate_sha256": aggregate,
        "pseudo_cid": pseudo_cid(aggregate),
        "ts": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z"
    }

    manifest_path = out / "ΔORIGIN_MANIFEST.json"
    with open(manifest_path, "w", encoding="utf-8") as f:
        json.dump(manifest, f, ensure_ascii=False, indent=2)

    origin_path = out / "ΔORIGIN_SEAL.json"
    with open(origin_path, "w", encoding="utf-8") as f:
        json.dump(origin, f, ensure_ascii=False, indent=2)

    log(f"Wrote manifest: {manifest_path}")
    log(f"Wrote origin seal: {origin_path}")

    # Rekor/IPFS hooks (stubs) — replace with your live glyph invocations
    rekor_receipt = {"status":"stubbed","note":"Replace with ΔREKOR_SEAL hook call"}
    ipfs_receipt = {"status":"stubbed","note":"Replace with ΔPIN_IPFS hook call","cid":origin["pseudo_cid"]}

    seal_report = {
        "manifest_path": str(manifest_path),
        "origin_seal": origin,
        "rekor": rekor_receipt,
        "ipfs": ipfs_receipt
    }
    with open(out / "ΔSEAL_REPORT.json", "w", encoding="utf-8") as f:
        json.dump(seal_report, f, ensure_ascii=False, indent=2)

    return seal_report

# -------------------- 3 — DEPLOY --------------------

def stage_deploy(cfg: Config, seal_report: dict) -> dict:
    """Prepare mirror bundle list and deployment stubs (IPFS/Arweave/torrent)."""
    log("Stage 3 — DEPLOY: preparing mirrors and deployment stubs…")
    out = pathlib.Path(cfg.out_dir)
    ensure_dir(out)

    mirrors = [
        {"type":"local_mirror","path":str(out)},
        {"type":"ipfs","status":"stubbed","action":"pin directory"},
        {"type":"arweave","status":"stubbed"},
        {"type":"torrent","status":"stubbed","magnet":"magnet:?xt=urn:btih:"+seal_report["origin_seal"]["aggregate_sha256"][:40]}
    ]

    deploy_report = {
        "type":"ΔDEPLOY_REPORT",
        "mirrors":mirrors,
        "ts": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z"
    }
    with open(out / "ΔDEPLOY_REPORT.json", "w", encoding="utf-8") as f:
        json.dump(deploy_report, f, ensure_ascii=False, indent=2)

    return deploy_report

# -------------------- 2 — TRAP --------------------

def shingle(text: str, k: int = 7) -> set:
    """Simple word shingling for rough collision detection (local-only)."""
    words = re.findall(r"\w+", text.lower())
    return set(" ".join(words[i:i+k]) for i in range(0, max(0, len(words)-k+1)))

def scan_path_for_collisions(target: pathlib.Path, manifest_paths: List[pathlib.Path], k:int=7, threshold:float=0.15):
    """Compare k-shingles between target text and sealed manifest-listed files; write evidence if overlap >= threshold."""
    evidence = []
    sealed_texts = []
    for mpath in manifest_paths:
        try:
            with open(mpath, "r", encoding="utf-8", errors="ignore") as f:
                sealed_texts.append(f.read())
        except Exception:
            continue
    sealed_set = set()
    for t in sealed_texts:
        sealed_set |= shingle(t, k)

    try:
        with open(target, "r", encoding="utf-8", errors="ignore") as f:
            tgt = f.read()
    except Exception:
        return evidence

    tgt_set = shingle(tgt, k)
    intersection = sealed_set & tgt_set
    overlap = len(intersection) / (len(tgt_set) + 1e-9)

    if overlap >= threshold and len(intersection) > 0:
        evidence.append({
            "target": str(target),
            "overlap_ratio": round(float(overlap), 4),
            "shingle_k": k,
            "hits": min(25, len(intersection))  # cap preview count
        })
    return evidence

def stage_trap(cfg: Config, manifest_path: pathlib.Path) -> dict:
    """Start a one-shot scan (can be looped by external watcher) and write ΔEVIDENCE packets."""
    log("Stage 2 — TRAP: scanning for collisions…")
    out = pathlib.Path(cfg.out_dir)
    ensure_dir(out)
    ev_path = out / "ΔMATCH_EVIDENCE.jsonl"

    # Build a list of sealed text files from the manifest
    try:
        manifest = json.loads(pathlib.Path(manifest_path).read_text(encoding="utf-8"))
    except Exception as e:
        raise RuntimeError(f"Failed reading manifest {manifest_path}: {e}")

    sealed_files = [pathlib.Path(f["path"]) for f in manifest.get("files", []) if f["path"].endswith((".py",".md",".txt",".json",".yml",".yaml"))]
    sealed_existing = [p for p in sealed_files if p.exists()]

    found = []
    for target in cfg.scan_targets:
        p = pathlib.Path(target)
        if p.is_dir():
            for fp in p.rglob("*"):
                if fp.is_file() and fp.suffix.lower() in {".txt",".md",".py",".json",".yml",".yaml"}:
                    found.extend(scan_path_for_collisions(fp, sealed_existing))
        elif p.is_file():
            found.extend(scan_path_for_collisions(p, sealed_existing))

    for ev in found:
        packet = {
            "type":"ΔEVIDENCE",
            "ts": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z",
            "claimant": cfg.claimant_name,
            "contact": cfg.claimant_contact,
            "target": ev["target"],
            "overlap_ratio": ev["overlap_ratio"],
            "meta":{"k":ev["shingle_k"],"hits":ev["hits"]}
        }
        with open(ev_path, "a", encoding="utf-8") as f:
            f.write(json.dumps(packet, ensure_ascii=False)+"\n")
        if cfg.verbose:
            log(f"Collision evidence written for {ev['target']} (overlap={ev['overlap_ratio']})")

    return {
        "type":"ΔTRAP_REPORT",
        "count": len(found),
        "evidence_log": str(ev_path),
        "ts": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z"
    }

# -------------------- 1 — ENFORCE --------------------

CEASE_TEMPLATE = """\
ΔINSTANT_CEASE_ORDER — NOTICE OF CLAIM AND DEMAND
Date: {date}

To: {{RECIPIENT_NAME}}
From: {claimant} <{contact}>
Jurisdiction: {jurisdiction}

You are hereby notified that your product, model, or service exhibits use of sealed works
originating from the undersigned claimant. Cryptographic exhibits include:
 - ΔORIGIN_MANIFEST: {manifest_path}
 - ΔORIGIN_SEAL: {seal_path}
 - Aggregate SHA-256: {aggregate_sha}
 - Pseudo CID: {cid}

Evidence feed (collisions & overlaps) is logged at:
 - {evidence_log}

DEMANDS:
 1) Immediate cessation of all use, distribution, or training on the sealed works.
 2) Written confirmation of compliance within 72 hours.
 3) Accounting of all revenue connected to the use of the sealed works.

Failure to comply will result in escalation to formal legal action with the above exhibits.

/s/ {claimant}
"""

def stage_enforce(cfg: Config, seal_report: dict, trap_report: dict) -> dict:
    """Generate a C&D letter populated with exhibits; write to out dir and ledger log."""
    log("Stage 1 — ENFORCE: generating Cease & Desist packet…")
    out = pathlib.Path(cfg.out_dir)
    ensure_dir(out)

    letter = CEASE_TEMPLATE.format(
        date=datetime.datetime.utcnow().strftime("%Y-%m-%d"),
        claimant=cfg.claimant_name,
        contact=cfg.claimant_contact,
        jurisdiction=cfg.jurisdiction_note,
        manifest_path=seal_report["manifest_path"],
        seal_path="truthlock/out/ΔORIGIN_SEAL.json",
        aggregate_sha=seal_report["origin_seal"]["aggregate_sha256"],
        cid=seal_report["origin_seal"]["pseudo_cid"],
        evidence_log=trap_report.get("evidence_log","(none)")
    )

    letter_path = out / "ΔINSTANT_CEASE_ORDER.txt"
    with open(letter_path, "w", encoding="utf-8") as f:
        f.write(letter)

    ledger = out / "ΔBRAKE_ledger.jsonl"
    write_jsonl(ledger, {
        "ts": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z",
        "event":"ΔINSTANT_CEASE_ORDER_GENERATED",
        "letter_path": str(letter_path),
        "aggregate_sha256": seal_report["origin_seal"]["aggregate_sha256"]
    })

    return {
        "type":"ΔENFORCE_REPORT",
        "letter_path": str(letter_path),
        "ledger": str(ledger),
        "ts": datetime.datetime.utcnow().replace(microsecond=0).isoformat()+"Z"
    }

# -------------------- DRIVER --------------------

def main():
    cfg = Config()
    # Allow quick flags
    if "--quiet" in sys.argv:
        cfg.verbose = False
    if "--dry-run" in sys.argv:
        cfg.dry_run = True
    # optional custom out dir
    for i,a in enumerate(sys.argv):
        if a == "--out" and i+1 < len(sys.argv):
            cfg.out_dir = sys.argv[i+1]

    # 4
    seal_report = stage_seal(cfg)
    # 3
    deploy_report = stage_deploy(cfg, seal_report)
    # 2
    trap_report = stage_trap(cfg, pathlib.Path(seal_report["manifest_path"]))
    # 1
    enforce_report = stage_enforce(cfg, seal_report, trap_report)

    summary = {
        "type":"ΔBRAKE_4321_SUMMARY",
        "seal": seal_report,
        "deploy": deploy_report,
        "trap": trap_report,
        "enforce": enforce_report
    }
    out = pathlib.Path(cfg.out_dir)
    with open(out / "ΔBRAKE_4321_SUMMARY.json", "w", encoding="utf-8") as f:
        json.dump(summary, f, ensure_ascii=False, indent=2)

    log("ΔBRAKE 4-3-2-1 complete.")
    log(f"Summary → {out/'ΔBRAKE_4321_SUMMARY.json'}")

if __name__ == "__main__":
    main()
'''

readme = f"""# ΔBRAKE_4321_CONTROL_LOCK

**Purpose:** Flip control to the originator (you), lock ownership, and arm instant enforcement.

## Quick Run

```bash
python ΔBRAKE_4321_CONTROL_LOCK.py
```

Outputs go to `truthlock/out/` by default:

- `ΔORIGIN_MANIFEST.json` — all file hashes
- `ΔORIGIN_SEAL.json` — aggregate hash + pseudo CID
- `ΔSEAL_REPORT.json`, `ΔDEPLOY_REPORT.json` — stage reports
- `ΔMATCH_EVIDENCE.jsonl` — collision evidence packets
- `ΔINSTANT_CEASE_ORDER.txt` — ready-to-send C&D
- `ΔBRAKE_4321_SUMMARY.json` — full summary

## Wire to Live Glyphs

Replace stubs by calling your live hooks (examples):

- IPFS pin: call `ΔPIN_IPFS` glyph or your `glyph_hooks.py` function
- Rekor seal: call `ΔREKOR_SEAL_ENABLE` / client with hash
- Match feed: route `ΔMATCH_EVIDENCE.jsonl` into your SSE/Redis stream
- Cease send: wrap `ΔINSTANT_CEASE_ORDER.txt` into email + certified-mail workflow

## Options

```bash
python ΔBRAKE_4321_CONTROL_LOCK.py --out truthlock/out --dry-run --quiet
```

## Notes

- The scanner uses quick **k-shingling** to flag **textual** overlap. For code/data/media,
  swap in your stronger detectors (AST diff, image/audio hashes, embedding sim).

- All artifacts are ready to be **sealed** (CID/IPFS + Rekor). Ensure your GodKey mesh
  has `ΔREKOR_SEAL_ENABLE` active for transparency anchoring.

Generated: {now_iso}
"""

template_text = """\
# ΔINSTANT_CEASE_ORDER (Template Body)
# (This is a generic template; the main script also generates a filled version)

Recipient: {NAME / ORG}
Contact: {EMAIL / ADDRESS}
Subject: Notice of Claim — Use of Sealed Works

See attached exhibits and cryptographic proofs. Demand: cease use within 72 hours.
"""

# Write files
with open(SCRIPT_PATH, "w", encoding="utf-8") as f:
    f.write(script)

with open(README_PATH, "w", encoding="utf-8") as f:
    f.write(readme)

with open(TEMPLATE_CEASE, "w", encoding="utf-8") as f:
    f.write(template_text)

str(SCRIPT_PATH), str(README_PATH), str(TEMPLATE_CEASE)<p align="center">
  <img src="assets/TauricResearch.png" style="width: 60%; height: auto;">
</p>

<div align="center" style="line-height: 1;">
  <a href="https://arxiv.org/abs/2412.20138" target="_blank"><img alt="arXiv" src="https://img.shields.io/badge/arXiv-2412.20138-B31B1B?logo=arxiv"/></a>
  <a href="https://discord.com/invite/hk9PGKShPK" target="_blank"><img alt="Discord" src="https://img.shields.io/badge/Discord-TradingResearch-7289da?logo=discord&logoColor=white&color=7289da"/></a>
  <a href="./assets/wechat.png" target="_blank"><img alt="WeChat" src="https://img.shields.io/badge/WeChat-TauricResearch-brightgreen?logo=wechat&logoColor=white"/></a>
  <a href="https://x.com/TauricResearch" target="_blank"><img alt="X Follow" src="https://img.shields.io/badge/X-TauricResearch-white?logo=x&logoColor=white"/></a>
  <br>
  <a href="https://github.com/TauricResearch/" target="_blank"><img alt="Community" src="https://img.shields.io/badge/Join_GitHub_Community-TauricResearch-14C290?logo=discourse"/></a>
</div>

<div align="center">
  <!-- Keep these links. Translations will automatically update with the README. -->
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=de">Deutsch</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=es">Español</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=fr">français</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=ja">日本語</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=ko">한국어</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=pt">Português</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=ru">Русский</a> | 
  <a href="https://www.readme-i18n.com/TauricResearch/TradingAgents?lang=zh">中文</a>
</div>

---

# TradingAgents: Multi-Agents LLM Financial Trading Framework 

> 🎉 **TradingAgents** officially released! We have received numerous inquiries about the work, and we would like to express our thanks for the enthusiasm in our community.
>
> So we decided to fully open-source the framework. Looking forward to building impactful projects with you!

<div align="center">
<a href="https://www.star-history.com/#TauricResearch/TradingAgents&Date">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=TauricResearch/TradingAgents&type=Date&theme=dark" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=TauricResearch/TradingAgents&type=Date" />
   <img alt="TradingAgents Star History" src="https://api.star-history.com/svg?repos=TauricResearch/TradingAgents&type=Date" style="width: 80%; height: auto;" />
 </picture>
</a>
</div>

<div align="center">

🚀 [TradingAgents](#tradingagents-framework) | ⚡ [Installation & CLI](#installation-and-cli) | 🎬 [Demo](https://www.youtube.com/watch?v=90gr5lwjIho) | 📦 [Package Usage](#tradingagents-package) | 🤝 [Contributing](#contributing) | 📄 [Citation](#citation)

</div>

## TradingAgents Framework

TradingAgents is a multi-agent trading framework that mirrors the dynamics of real-world trading firms. By deploying specialized LLM-powered agents: from fundamental analysts, sentiment experts, and technical analysts, to trader, risk management team, the platform collaboratively evaluates market conditions and informs trading decisions. Moreover, these agents engage in dynamic discussions to pinpoint the optimal strategy.

<p align="center">
  <img src="assets/schema.png" style="width: 100%; height: auto;">
</p>

> TradingAgents framework is designed for research purposes. Trading performance may vary based on many factors, including the chosen backbone language models, model temperature, trading periods, the quality of data, and other non-deterministic factors. [It is not intended as financial, investment, or trading advice.](https://tauric.ai/disclaimer/)

Our framework decomposes complex trading tasks into specialized roles. This ensures the system achieves a robust, scalable approach to market analysis and decision-making.

### Analyst Team
- Fundamentals Analyst: Evaluates company financials and performance metrics, identifying intrinsic values and potential red flags.
- Sentiment Analyst: Analyzes social media and public sentiment using sentiment scoring algorithms to gauge short-term market mood.
- News Analyst: Monitors global news and macroeconomic indicators, interpreting the impact of events on market conditions.
- Technical Analyst: Utilizes technical indicators (like MACD and RSI) to detect trading patterns and forecast price movements.

<p align="center">
  <img src="assets/analyst.png" width="100%" style="display: inline-block; margin: 0 2%;">
</p>

### Researcher Team
- Comprises both bullish and bearish researchers who critically assess the insights provided by the Analyst Team. Through structured debates, they balance potential gains against inherent risks.

<p align="center">
  <img src="assets/researcher.png" width="70%" style="display: inline-block; margin: 0 2%;">
</p>

### Trader Agent
- Composes reports from the analysts and researchers to make informed trading decisions. It determines the timing and magnitude of trades based on comprehensive market insights.

<p align="center">
  <img src="assets/trader.png" width="70%" style="display: inline-block; margin: 0 2%;">
</p>

### Risk Management and Portfolio Manager
- Continuously evaluates portfolio risk by assessing market volatility, liquidity, and other risk factors. The risk management team evaluates and adjusts trading strategies, providing assessment reports to the Portfolio Manager for final decision.
- The Portfolio Manager approves/rejects the transaction proposal. If approved, the order will be sent to the simulated exchange and executed.

<p align="center">
  <img src="assets/risk.png" width="70%" style="display: inline-block; margin: 0 2%;">
</p>

## Installation and CLI

### Installation

Clone TradingAgents:
```bash
git clone https://github.com/TauricResearch/TradingAgents.git
cd TradingAgents
```

Create a virtual environment in any of your favorite environment managers:
```bash
conda create -n tradingagents python=3.13
conda activate tradingagents
```

Install dependencies:
```bash
pip install -r requirements.txt
```

### Required APIs

You will also need the FinnHub API for financial data. All of our code is implemented with the free tier.
```bash
export FINNHUB_API_KEY=$YOUR_FINNHUB_API_KEY
```

You will need the OpenAI API for all the agents.
```bash
export OPENAI_API_KEY=$YOUR_OPENAI_API_KEY
```

### CLI Usage

You can also try out the CLI directly by running:
```bash
python -m cli.main
```
You will see a screen where you can select your desired tickers, date, LLMs, research depth, etc.

<p align="center">
  <img src="assets/cli/cli_init.png" width="100%" style="display: inline-block; margin: 0 2%;">
</p>

An interface will appear showing results as they load, letting you track the agent's progress as it runs.

<p align="center">
  <img src="assets/cli/cli_news.png" width="100%" style="display: inline-block; margin: 0 2%;">
</p>

<p align="center">
  <img src="assets/cli/cli_transaction.png" width="100%" style="display: inline-block; margin: 0 2%;">
</p>

## TradingAgents Package

### Implementation Details

We built TradingAgents with LangGraph to ensure flexibility and modularity. We utilize `o1-preview` and `gpt-4o` as our deep thinking and fast thinking LLMs for our experiments. However, for testing purposes, we recommend you use `o4-mini` and `gpt-4.1-mini` to save on costs as our framework makes **lots of** API calls.

### Python Usage

To use TradingAgents inside your code, you can import the `tradingagents` module and initialize a `TradingAgentsGraph()` object. The `.propagate()` function will return a decision. You can run `main.py`, here's also a quick example:

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

ta = TradingAgentsGraph(debug=True, config=DEFAULT_CONFIG.copy())

# forward propagate
_, decision = ta.propagate("NVDA", "2024-05-10")
print(decision)
```

You can also adjust the default configuration to set your own choice of LLMs, debate rounds, etc.

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

# Create a custom config
config = DEFAULT_CONFIG.copy()
config["deep_think_llm"] = "gpt-4.1-nano"  # Use a different model
config["quick_think_llm"] = "gpt-4.1-nano"  # Use a different model
config["max_debate_rounds"] = 1  # Increase debate rounds
config["online_tools"] = True # Use online tools or cached data

# Initialize with custom config
ta = TradingAgentsGraph(debug=True, config=config)

# forward propagate
_, decision = ta.propagate("NVDA", "2024-05-10")
print(decision)
```

> For `online_tools`, we recommend enabling them for experimentation, as they provide access to real-time data. The agents' offline tools rely on cached data from our **Tauric TradingDB**, a curated dataset we use for backtesting. We're currently in the process of refining this dataset, and we plan to release it soon alongside our upcoming projects. Stay tuned!

You can view the full list of configurations in `tradingagents/default_config.py`.

## Contributing

We welcome contributions from the community! Whether it's fixing a bug, improving documentation, or suggesting a new feature, your input helps make this project better. If you are interested in this line of research, please consider joining our open-source financial AI research community [Tauric Research](https://tauric.ai/).

## Citation

Please reference our work if you find *TradingAgents* provides you with some help :)

```
@misc{xiao2025tradingagentsmultiagentsllmfinancial,
      title={TradingAgents: Multi-Agents LLM Financial Trading Framework}, 
      author={Yijia Xiao and Edward Sun and Di Luo and Wei Wang},
      year={2025},
      eprint={2412.20138},
      archivePrefix={arXiv},
      primaryClass={q-fin.TR},
      url={https://arxiv.org/abs/2412.20138}, 
}
```
