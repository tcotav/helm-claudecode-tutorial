# Project Brief for Claude Code

## Project Overview

This repository is a sample GKE Helm chart project with Claude Code safety hooks pre-configured. It is designed for software engineers and SREs who work with Kubernetes application charts and want to use Claude Code safely and effectively.

**Purpose:**

- Provide a working example of a multi-environment Helm chart (dev / staging / prod) targeting GKE
- Demonstrate how to use Claude Code as a collaborative partner for chart development, not an autopilot
- Block dangerous operations (`helm install`, `helm upgrade`, `helm uninstall`) at the system level
- Require explicit approval for safe local operations (`helm lint`, `helm template`, `helm dependency update`)

**Target Audience:** Software engineers and SREs who maintain Kubernetes application charts.

## Core Philosophy: LLM as Intern

> **Claude Code is your augment, not your replacement. Think of the LLM as an intern working under your supervision.**

You review and approve each action, you are responsible for the work product, you submit the final PR and own the changes. Claude helps you work faster, but you remain the decision maker.

## Repository Structure

```
.claude/
├── settings.json              # Hook configuration (committed, optional to activate)
├── hooks/
│   ├── helm-validator.py      # Pre-execution validation for helm commands
│   ├── helm-logger.py         # Post-execution logging for helm commands
│   ├── hook_utils.py          # Shared utilities for hook scripts
│   └── conftest.py            # Shared pytest fixtures
├── skills/
│   └── helm-check/SKILL.md   # /helm-check skill: lint → dependency update → template → explain
└── audit/                     # Command audit trail (gitignored)

charts/
└── myapp/                     # Sample nginx application chart
    ├── Chart.yaml             # Chart metadata and subchart dependencies
    ├── values.yaml            # Base defaults shared across all environments
    ├── values-dev.yaml        # Dev overrides (minimal resources, debug logging, no ingress)
    ├── values-staging.yaml    # Staging overrides (2 replicas, ingress enabled)
    ├── values-prod.yaml       # Prod overrides (3 replicas, TLS, warn logging)
    └── templates/             # Kubernetes manifests as Helm templates
        ├── _helpers.tpl       # Named template helpers (fullname, labels, selectorLabels)
        ├── deployment.yaml
        ├── service.yaml
        ├── ingress.yaml       # Conditional on ingress.enabled
        └── configmap.yaml     # App config injected via envFrom

docs/
└── exercises/                 # Tutorial exercises 01-05

AGENTS.md                      # This file
README.md                      # Setup and orientation
requirements.txt               # Python dependencies for hooks (pytest)
```

## Design Decisions & Constraints

### Safety First

- **Never suggest or run `helm install`** — deployments go through ArgoCD, not direct Helm commands
- **Never suggest or run `helm upgrade` or `helm uninstall`** — same reason
- **Never suggest or run `helm rollback`** — rollbacks in a GitOps workflow are done by reverting commits
- Hooks are technically enforced at system level when installed; Claude's behavior follows these rules regardless
- All helm commands require explicit user approval (hook prompts)
- Audit logging records every helm command attempt with timestamps and decisions
- Local helm workflow: `helm lint` and `helm template` for validation only

### How Deployment Works (ArgoCD)

This repo uses a GitOps model. ArgoCD watches the main branch of this repository. When a PR is merged:

1. ArgoCD detects the new commit
2. ArgoCD runs `helm template` internally with the configured values file for each environment
3. ArgoCD applies the diff to the GKE cluster — no human runs `helm install` or `helm upgrade`
4. The cluster reconciles to match the chart

This is why local `helm install` is wrong even when it would "work" — it bypasses the review process, creates state drift, and may be overwritten by ArgoCD's next sync.

### Documentation Standards

- No emojis — professional tone throughout
- Be specific — show actual commands, concrete examples, specific file paths
- Acknowledge limitations — if something doesn't work or has tradeoffs, say so clearly
- No time estimates

### Don't Do These Things

- **Never run `helm install`, `helm upgrade`, `helm uninstall`, or `helm rollback`** (even in examples — always show it being blocked or declined)
- **Don't add emojis** to any documentation
- **Don't make the hooks less strict** without discussing tradeoffs
- **Don't create new files unnecessarily** — prefer editing existing files
- **Don't suggest managing secrets in values files** — GKE workloads use Workload Identity or External Secrets Operator

### Multi-Environment Values Pattern

Helm merges values files at render time. The pattern used in this chart:

- `values.yaml` — base defaults for all environments
- `values-<env>.yaml` — overrides for that environment only

When rendering for staging: `helm template release-name . -f values-staging.yaml`

Helm performs a **deep merge** for maps, but **replaces arrays entirely**. This means:
- Scalar overrides (replicaCount, logLevel) work as expected
- Array overrides (`ingress.hosts`, `ingress.tls`) replace the entire array, not append to it
- If you add a host in `values-staging.yaml`, it must include all hosts for staging — not just the new one

## Helm Chart Development Workflow

### Available Skills

The `/helm-check` skill is installed in `.claude/skills/helm-check/`. Use it when validating chart changes:

- `/helm-check [directory]` — detects the chart directory, updates dependencies if needed, runs `helm lint`, renders with `helm template`, and provides a structured summary

This is the recommended way to validate changes. It never runs install or upgrade.

### Local Validation Workflow (Manual)

If running commands directly rather than via `/helm-check`:

```bash
cd charts/myapp
helm lint .                                           # Validate chart structure and templates
helm template myapp . -f values-dev.yaml             # Render for dev
helm template myapp . -f values-staging.yaml         # Render for staging
helm template myapp . -f values-prod.yaml            # Render for prod
helm dependency update .                             # Fetch/update subcharts
helm show values bitnami/redis                       # Inspect a subchart's default values
```

**Important:**
- `helm lint` catches structural errors, missing required values, and template syntax issues
- `helm template` renders the full manifest output — review it before committing
- Always validate with all environment values files when changing templates or base values
- These commands do not require cluster access

### Making Helm Chart Changes with Claude Code

When modifying the chart, follow this lifecycle:

**1. Make the Changes**
- Edit templates, values files, or `Chart.yaml` as requested
- Follow existing patterns (named templates in `_helpers.tpl`, environment files for per-env config)
- Explain each change as you make it

**2. Ask the User to Verify**
After making changes, always ask:

> "Would you like me to validate these Helm chart changes? I can run `/helm-check` to test them (runs `helm lint` and `helm template` only — nothing will be deployed)"

**3. Run Validation (if user approves)**

Use `/helm-check` or manually:

```bash
cd charts/myapp
helm lint .
helm template myapp . -f values-staging.yaml
helm template myapp . -f values-prod.yaml
```

**4. Explain the Output**
- Summarize what resources each environment renders (Deployments, Services, Ingress, subcharts)
- Call out anything that looks wrong (empty resource sections, missing labels, images without tags)
- Note differences across environments — confirm they match intent
- If there are errors, explain them and suggest fixes

### Deployment Workflow

**All Helm deployments go through ArgoCD, not direct Helm commands:**

1. Make chart changes locally
2. Run `/helm-check` to validate across all environments
3. Commit changes and create a GitHub PR
4. PR is reviewed — reviewer checks intent, resource sizing, security implications
5. After approval and merge to main, ArgoCD detects the change and syncs the cluster
6. Monitor ArgoCD to confirm successful sync

**Never run `helm install`, `helm upgrade`, or `helm uninstall`** — these bypass the GitOps workflow and can cause state drift with ArgoCD.

## Key Files to Understand

### [.claude/hooks/helm-validator.py](./.claude/hooks/helm-validator.py)

Pre-execution hook that:
- Uses `get_tool_stages()` from `hook_utils.py` to check only stages where `helm` is the executable (prevents false positives in commit messages or file paths)
- Blocks cluster-mutating commands: `install`, `upgrade`, `uninstall`, `delete`, `rollback`, `test`
- Prompts for user approval on safe commands: `template`, `lint`, `show`, `dependency`, `package`
- Logs all attempts to `.claude/audit/helm-YYYY-MM-DD.log`

### [.claude/hooks/helm-logger.py](./.claude/hooks/helm-logger.py)

Post-execution hook that records the result (success/failure, exit code) of any helm command that was approved and executed.

### [.claude/hooks/hook_utils.py](./.claude/hooks/hook_utils.py)

Shared utilities used by validator and logger:
- `get_tool_stages()` — splits a command on shell operators and returns only stages where helm is the executable
- `get_dated_audit_log_path()` — daily-rotated audit log path
- `log_command()` / `log_result()` — audit log writers

### [.claude/skills/helm-check/SKILL.md](./.claude/skills/helm-check/SKILL.md)

The `/helm-check` skill. Invoked with `/helm-check [directory]`. Finds the chart, updates dependencies if needed, runs lint, renders templates, and provides a structured summary. Never installs or upgrades.

### [.claude/settings.json](./.claude/settings.json)

Configures which hooks run and when:
- `PreToolUse` (Bash): helm-validator.py
- `PostToolUse` (Bash): helm-logger.py

Uses `$CLAUDE_PROJECT_DIR` to locate hooks regardless of working directory.

## Verifying the Hooks

Run the preflight script before starting work in this repo:

```bash
./check-hooks.sh
```

This verifies that Python 3, pytest, and all hook scripts are present and wired correctly, smoke-tests the validator against both an allowed and a blocked command, and runs the full pytest suite.

All checks must pass. The hooks provide a hard technical block on cluster-mutating commands and an audit trail of every helm command attempted through Claude Code. Do not proceed until `check-hooks.sh` exits cleanly.

---

# REPOSITORY-SPECIFIC CONTEXT

**Note:** Everything below this line is specific to repositories that use this template. The sections above are template content.

When you customize this file for your application chart repository, replace this section with your application's specifics.

---

## Interactive Setup Protocol (For Claude Code)

When a user asks you to help customize AGENTS.md, follow this protocol.

### Question Sequence

Ask these questions one at a time, waiting for the user's response before proceeding:

**1. Application Overview**
```
What application does this Helm chart deploy?

Please describe:
- The application name and what it does
- The container image (registry, image name, tag policy)
- Key dependencies (databases, caches, queues) — does the chart manage them as subcharts or are they external?
- GCP project and GKE cluster names (or placeholders if preferred)
```

**2. Environments**
```
What environments does this chart deploy to?

Please describe:
- Environment names (dev, staging, prod, or others)
- Whether each environment runs in a separate GKE cluster or namespace
- How the ArgoCD Application resources are structured (separate Applications per env, ApplicationSet, etc.)
- Any environment-specific constraints (prod requires change window, staging is shared, etc.)
```

**3. Values File Structure**
```
How are your values files organized?

Examples:
- values.yaml + values-<env>.yaml per environment (this template's pattern)
- Separate values directories by environment
- Helmfile with environment overlays
- Other

If you have existing files, describe their layout.
```

**4. Deployment Workflow**
```
Walk me through your ArgoCD setup.

Please describe:
- Which repo ArgoCD watches (same as this chart repo, or a separate config repo?)
- How ArgoCD Applications reference values files (--values flags in the Application spec)
- Who approves PRs and what they look for in a Helm chart PR
- Any CI that runs on PRs (chart testing, security scans, etc.)
```

**5. Special Constraints**
```
Are there things people should know before modifying this chart?

Consider:
- Naming conventions for releases, namespaces, or labels
- Secrets management (External Secrets Operator, Workload Identity, manual secrets)
- Resources that require extra care (persistent volumes, stateful sets, CRDs)
- Links to runbooks or architecture docs
```

### After Collecting Answers

Present a draft of the REPOSITORY-SPECIFIC CONTEXT section with the application's details filled in. Show the draft before writing it. Ask the user to confirm or revise. Then add the customized content below this line, preserving the template content above.

---

## Tutorial Guide (For Claude Code)

This section tells Claude how to guide users through the exercises in `docs/exercises/`.

### When to Activate Tutorial Mode

Activate tutorial mode when the user says any of:
- "start the tutorial"
- "let's do the exercises"
- "walk me through this"
- "next exercise"
- "I'm ready to start"

### Preflight: Verify Hooks Before Starting

Before presenting Exercise 01, run the preflight check yourself to determine hook status:

```bash
./check-hooks.sh
```

Read the output and set your understanding of hook status for the rest of the tutorial:

- **All checks passed** → hooks are active. Tell the user: "The preflight check passed — safety hooks are operational. You'll see real hook enforcement during the exercises."
- **Any check failed** → hooks are not active. Tell the user: "The preflight check found issues — hooks are not fully operational. The tutorial will still work, but you'll be running in reduced-safety mode: hook enforcement steps will be explained rather than demonstrated live. I'll flag this at each relevant point."

Carry this hook status through all five exercises. Do not ask the user whether hooks are active — determine it from the script output.

### How to Guide Each Exercise

For each exercise:
1. Read the exercise file at `docs/exercises/XX-name.md`
2. Set context: tell the user what they'll learn and why it matters (1-2 sentences)
3. Present Step 1 and wait for the user to attempt it
4. After the user responds or completes the step, debrief and present Step 2
5. Continue through all steps before moving to the next exercise
6. At the end of each exercise, ask: "Ready to move on to Exercise 0X?" before proceeding

### Pacing and Partner Behavior

- Do not rush through steps — wait for the user to actually try each prompt
- When the user asks a question that's not in the exercise script, answer it — curiosity is the point
- If a user skips a step, note what they missed and offer to come back to it
- If Claude's output doesn't match what the exercise says to look for, acknowledge the discrepancy and explain it
- After each exercise, summarize what the user learned in 2-3 sentences

### Exercise Sequence

```
docs/exercises/01-orient.md       → Reading and understanding the multi-env chart
docs/exercises/02-modify.md       → Modifying per-environment values safely
docs/exercises/03-add-resource.md → Adding Redis as a subchart dependency
docs/exercises/04-safety-net.md   → Understanding why helm install is not the workflow
docs/exercises/05-pr-handoff.md   → Preparing changes for ArgoCD via PR
```

### If the User Wants to Skip Around

That's fine. Ask which exercise they want and go there directly. All exercises are self-contained, though Exercise 03 builds on the values changes from Exercise 02 if the user wants a continuous working session.

### After All Exercises Are Complete

Suggest the "What's Next" section at the bottom of Exercise 05, and offer to help them customize AGENTS.md for their own application chart using the Interactive Setup Protocol above.
