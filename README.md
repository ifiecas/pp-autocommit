# pp-autocommit

**Every deploy becomes a commit — and every commit can become a deploy.** Automated source control for Copilot Studio solutions via Power Platform Pipelines extensibility, with a restore pipeline for rollbacks and edit-in-code workflows.

When a solution is deployed to Test through Power Platform Pipelines, this setup automatically exports it from dev, unpacks it into readable files, and commits it to an Azure DevOps repo — zero clicks past the deployment approval. Your repo becomes a running history of every successful deploy, one commit each, with the solution unpacked so git diffs show exactly what changed in your agents, flows, and apps.

```
Maker deploys in Copilot Studio
        │
        ▼
Power Platform Pipeline runs (with approval gate, if configured)
        │
        ▼  deployment succeeds
OnDeploymentCompleted event fires in the host environment
        │
        ▼
Power Automate flow checks the stage → queues an Azure DevOps pipeline
        │
        ▼
Pipeline: export solution from dev → unpack → git commit → push
```

Fully dynamic: **any** solution deployed through the pipeline gets committed. Add new agents, never touch this setup.

---

## Contents

- [Why this matters](#why-this-matters)
- [What's in this repo](#whats-in-this-repo)
- [Prerequisites](#prerequisites)
- [Setup — Azure DevOps](#part-1--azure-devops-setup) (~10 min)
- [Setup — Power Automate flow](#part-2--the-power-automate-flow) (~5 min)
- [Test it end to end](#part-3--test-it-end-to-end)
- [Going the other way — edit in code, restore to dev](#part-4--going-the-other-way-edit-in-code-restore-to-dev)
- [Troubleshooting](#troubleshooting)
- [Production notes](#production-notes)

---

## The problem, in plain English

You know how Word keeps version history? You can open any document, click back through every save, see who changed what, and restore last Tuesday's version if today's edit went wrong. Nobody set that up. It's just there, quietly, every time you hit save.

**Copilot Studio agents don't have that.** When someone updates an agent — rewords a topic, changes a trigger, tweaks a flow — the old version is simply *gone*. There's no "see what changed," no "who did this," no "take me back to when it worked."

So when an agent starts giving odd answers, the conversation goes like this:

> "It was fine last week. What changed?"
> "…I think Sam edited something? Or was that the week before?"

Nobody's at fault — the tool just never kept receipts.

**This project gives your agents that version history.** Every time an agent is deployed for testing, a snapshot is saved automatically — what changed, who deployed it, when, and whether it was approved. Like Word's version history, nobody has to remember to do anything. You build, you deploy, and the receipts write themselves.

---

## Why this matters

Copilot Studio agents are low-code, but they're still software — and in most orgs, the only record of what an agent looked like last month is whatever's in the environment right now. No history, no diffs, no rollback, no answer to "what changed?"

This setup fixes that:

- **"What changed?" becomes answerable.** Agent behaving differently after a deploy? Open the commit diff and see the exact topic text, trigger phrase, or flow logic that changed — instead of comparing screenshots from memory.
- **Audit trail for free.** Every deploy is one commit: timestamped, tied to a build number, tied to an approved deployment. "Who changed this agent, when, and was it approved?" becomes a git log, not an investigation. Big deal in regulated industries.
- **Rollback becomes real.** The previous known-good state sits in git, unpacked and re-importable — not in someone's memory.
- **It closes the pro-code gap.** Every serious dev team has source control; most Copilot Studio teams have none. This brings maker work up to the standard the rest of engineering already holds.
- **Zero behaviour change for makers.** This is what makes it stick: makers keep deploying exactly as they do today. Nobody learns git, nobody runs a command. Governance that depends on humans remembering steps fails — governance that's a side effect of deploying doesn't.

Low-code doesn't have to mean no-history.

---

## What's in this repo

| File | What it is |
|---|---|
| `README.md` | This guide |
| `commit-solution.yml` | The Azure DevOps pipeline — copy into your repo as-is, change one line |
| `restore-solution.yml` | The reverse pipeline: pack the solution from git and import it back into dev — for rollbacks and edit-in-code workflows |
| `sample-trigger-payload.json` | What `OnDeploymentCompleted` actually sends — so you can see where the useful fields live before you build the flow |

---

## Prerequisites

- Power Platform Pipelines with a dev → test stage (custom host recommended)
- An Azure DevOps organization with a project + repo for solution source
- A service principal (Entra app registration) with a client secret
- **[Power Platform Build Tools](https://marketplace.visualstudio.com/items?itemName=microsoft-IsvExpTools.PowerPlatform-BuildTools)** extension installed in your DevOps org (free)
- A licensed account for the flow's Azure DevOps connection — this connector requires interactive sign-in; use a dedicated service account for production

---

## Part 1 — Azure DevOps setup

### 1.1 Service connection

Project Settings → Service connections → New → **Power Platform**:

| Field | Value |
|---|---|
| Server URL | Your **dev** environment URL, e.g. `https://yourorg.crm.dynamics.com` (no trailing slash) |
| Tenant ID | App registration → Overview page in Entra |
| Application ID | Same page |
| Client secret | Certificates & secrets (create a new one if the original isn't saved — old values can't be viewed) |
| Connection name | e.g. `pp-dev-serviceprincipal` — **the YAML references this exact name** |

Tick **Grant access permission to all pipelines**.

> Find your environment URL: Power Platform admin center → your dev environment → Environment URL.

### 1.2 Service principal permissions

The SP needs an **application user** in your **dev** environment with **System Customizer** (or higher) — that's where it exports from.

Admin center → dev environment → Settings → Users + permissions → Application users → New app user.

### 1.3 Let the pipeline push to git

Project Settings → Repositories → your repo → Security → **`<Project> Build Service`** → **Contribute: Allow**.

⚠️ This is the step everyone forgets. Without it, the final git push fails.

### 1.4 Add the pipeline YAML

Commit [`commit-solution.yml`](commit-solution.yml) to the root of your repo. One line to change:

```yaml
PowerPlatformSPN: 'pp-dev-serviceprincipal'   # ← your service connection name from 1.1
```

> **Why `$(solutionName)` as a variable and not a YAML parameter?** The Power Automate Azure DevOps connector sends queue-time **variables**, not parameters. Declaring `parameters:` in the YAML instead makes queuing from the flow fail with `You can't set the following variables (solutionName)`. Variables + the "settable at queue time" checkbox (1.6) is the combination that works.

### 1.5 Create the pipeline

Pipelines → New pipeline → Azure Repos Git → your repo → **Existing Azure Pipelines YAML file** → `/commit-solution.yml` → **Save** (dropdown next to Run — don't run yet).

Note the pipeline's ID from the URL: `_build?definitionId=N`. The flow needs that **number** — not the pipeline name, and not a run's `buildId`.

### 1.6 Make the variable settable at queue time

Edit the pipeline → **Variables** → New variable:

- Name: `solutionName`
- Value: any default
- ✅ **Let users override this value when running this pipeline** ← required, or queuing from the flow fails validation

---

## Part 2 — The Power Automate flow

Build this in the **host environment** — the one where the Power Platform Pipelines app is installed. Pipeline-extension flows must live there; if the trigger's Catalog dropdown doesn't show the Pipelines catalog, you're in the wrong environment.

Create an **automated cloud flow**, inside a solution for portability.

### 2.1 Trigger

**Microsoft Dataverse → "When an action is performed"**

| Field | Value |
|---|---|
| Catalog | `Microsoft Power Platform Pipelines` (some tenants show `Microsoft Dataverse Common` with category `Power Platform Pipelines` — same thing) |
| Category | `Custom` / `Power Platform Pipelines` |
| Table name | `(none)` |
| Action name | `OnDeploymentCompleted` |

`OnDeploymentCompleted` fires only on **successful** deployments — failed or rejected deploys never trigger a commit. No extra logic needed.

### 2.2 Condition

⚠️ **The gotcha that costs the most time:** the useful fields live under **`OutputParameters`**, not `InputParameters`. See [`sample-trigger-payload.json`](sample-trigger-payload.json) for the real structure.

Add a **Condition** with one row. Left side via the **Expression (fx)** tab — not Dynamic content:

```
triggerOutputs()?['body/OutputParameters/DeploymentStageName']
```

→ **is equal to** → your stage's exact display name, e.g. `Dev to UAT` (spacing and case matter).

The stage filter stops the flow committing when you later deploy test → prod. Deliberately **no** solution-name check — that's what makes this work for every solution automatically.

Optional hardening rows:
- `triggerOutputs()?['body/InputParameters/DeploymentStatus']` equals `200000002` (success)
- `triggerOutputs()?['body/OutputParameters/DeploymentPipelineName']` equals your pipeline's name — add this if multiple pipelines share one host

> **Tip:** to see your actual payload, run one deployment with the flow live and no condition, then read the trigger's raw Outputs in run history. Copy exact key names from there — never guess.

### 2.3 If yes → Queue a new build

**Azure DevOps → "Queue a new build"** (sign in with a licensed account when prompted):

| Field | Value |
|---|---|
| Organization Name | your DevOps org (*Enter custom value* if the dropdown shows "No items") |
| Project Name | your project |
| Build Definition Id | the **numeric ID** from 1.5 |
| Source Branch | `main` |
| Parameters | ↓ |

```json
{"solutionName": "@{triggerOutputs()?['body/OutputParameters/ArtifactName']}"}
```

Leave **If no** empty. Save, turn the flow on.

---

## Part 3 — Test it end to end

1. Make a small edit to a solution in dev (tweak a topic message in your agent)
2. Deploy to Test via your pipeline — approve if you have a gate
3. Follow the chain, in order:
   - **Flow run history** — triggered? condition true? Queue a new build green?
   - **DevOps → Pipelines** — build ran, all steps green?
   - **Repo** — commit `Deploy to Test: <solution> (build …)` with a `solutions/<solution>/` folder?
4. Open the commit's diff — your exact change, in plain text. That's your agent under source control. 🎉

A run ending in `No changes to commit` is fine: the repo already matches dev exactly (e.g. a redeploy with no edits). Reruns are idempotent by design.

---

## Part 4 — Going the other way: edit in code, restore to dev

The commit pipeline makes git a *history* of your agent. Adding [`restore-solution.yml`](restore-solution.yml) makes it a *source of truth*: edit the files in the repo (or check out an old version), run the restore pipeline, and the change lands back in Copilot Studio. Same machinery in reverse — **pack → import → publish** instead of export → unpack → commit.

Two things this unlocks:

- **Rollback for real.** Restore the solution folder from an old commit, run the pipeline — dev is back to that version. Then redeploy through the normal pipeline, approval and auto-commit included. Full circle.
- **Edit-in-code.** Fix a typo across ten topics with find-and-replace, make PR-reviewed changes to agent text, flip a settings boolean — all from the repo, no portal clicking.

### 4.1 Setup (~3 min — mirrors 1.4–1.6)

1. Commit [`restore-solution.yml`](restore-solution.yml) to the repo root; change the two `PowerPlatformSPN` values to your service connection name
2. Pipelines → New pipeline → Existing YAML → `/restore-solution.yml` → **Save**
3. Edit the pipeline → Variables → add `solutionName` (any default) → ✅ **Let users override this value when running this pipeline**

No flow this time — restoring is deliberately manual.

### 4.2 Editing agent files safely

Your agent's content lives under `solutions/<name>/botcomponents/`. Each `*.topic.*` folder is one topic; the message text sits in a YAML block like:

```yaml
- kind: SendActivity
  activity:
    text:
      - Hello, how can I help you today?
```

Fastest way to find the right file: use the repo **search** with a distinctive phrase your agent says.

**Safe to edit in git:** human-readable text (topic messages, `speak` lines), boolean settings in `configuration.json` (e.g. `aISettings` toggles).
**Do in the portal instead:** structural changes — adding components, touching IDs/GUIDs/`kind:` values, rewiring references, channels. Keep indentation exactly as-is (spaces, not tabs).

### 4.3 Running a restore

Pipelines → `restore-solution` → **Run pipeline** → set `solutionName` → Run. When it's green, open the agent in Copilot Studio — hard-refresh if the portal was already open (it caches). If `publishOnImport` is true in the solution's `configuration.json`, the import auto-publishes; otherwise the pipeline's publish step covers it.

If the **pack step fails after an edit**, it's almost always broken YAML formatting — revert the file in git and make a smaller change.

### 4.4 Safety rules

- **Only ever restore into dev.** Changes reach test/prod through the normal deployment pipeline — with its approval gate and auto-commit. Importing straight into downstream environments bypasses both and creates two sources of drift.
- **The import overwrites dev.** Anything someone changed in the portal since that commit is gone. In shared dev environments, treat restore as a deliberate act — consider a DevOps environment approval on this pipeline.
- **Keep `trigger: none`.** Auto-restoring on merge to `main` risks a loop with the auto-commit pipeline (the `[skip ci]` in commit messages is the guard, but manual is the safer default).

---

## Troubleshooting

Every row in this table is a mistake made while building this — so you don't have to.

| Symptom | Cause | Fix |
|---|---|---|
| Trigger's Catalog dropdown missing the Pipelines catalog | Building the flow in the wrong environment | Switch to the host environment |
| Queue a new build **Skipped** | Condition false — usually pointing at `InputParameters` instead of `OutputParameters`, or stage name mismatch | Check the trigger's raw Outputs; fix the expression path and exact stage string |
| `You can't set the following variables (solutionName)` | YAML declares `parameters:`, or the variable isn't marked settable | Use the `$(solutionName)` variable + tick "Let users override" (1.6) |
| Export step 403 / auth failure | SP has no app user or insufficient role in dev | Add app user with System Customizer (1.2) |
| Git push fails with a permissions error | Build Service lacks Contribute on the repo | 1.3 |
| "Definition not found" when queuing | Pipeline name or a run's `buildId` used instead of the definition ID | Use the number from `_build?definitionId=N` |
| Org/Build dropdowns show "No items" | Connector enumeration quirk, missing access, or the pipeline doesn't exist yet | *Enter custom value*; check the signed-in account can open the org; create the pipeline first |

---

## Production notes

- **Connections:** run the Azure DevOps connection as a dedicated licensed service account, not an individual — personal connections silently break on password changes, MFA re-prompts, or staff changes. The Dataverse trigger connection and the pipeline's service connection run as the service principal.
- **Flow ownership:** assign the flow to the service account or a team, so the whole flow — not just its connections — survives personnel changes. No individual should be load-bearing.
- **Least privilege:** the DevOps account needs only Basic access + Queue builds on this one pipeline. The SP needs only System Customizer in dev (plus whatever your deployment setup already grants).
- **Drift caveat:** the pipeline exports the *current* dev state, which could differ from the deployed artifact if someone edits the solution in the seconds between deploy and export. Negligible in practice; if you need exact-artifact fidelity, the deployment stage run record in the host stores the actual exported zip.

---

## Pairs well with

**[pp-pipelineapproval](https://github.com/ifiecas/pp-pipelineapproval)** — a pre-deployment approval flow on `OnPreDeploymentStarted`. Together they bookend the deployment: approval gates the deploy going in, auto-commit records the evidence coming out.
