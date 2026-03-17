# Helm Tutorial for Claude Code

A sample multi-environment Helm chart for a GKE application, with Claude Code safety hooks pre-configured. Use this to learn how to work with Helm charts using Claude Code as a collaborative partner.

## What This Is

This repo contains:

- A working Helm chart for a simple nginx-based application (`charts/myapp/`)
- Three environment values files: dev, staging, and prod
- Claude Code safety hooks that block `helm install`, `helm upgrade`, and `helm uninstall`
- A `/helm-check` skill for linting and rendering charts locally
- Five guided exercises that walk through a complete chart development workflow

The chart targets GKE. The deployment model is GitOps: ArgoCD watches this repo and applies changes after PRs merge. You never run `helm install` directly.

## Prerequisites

- Helm 3.x installed (`helm version`)
- Claude Code installed and running in this repo

## Setup

### 1. Clone and open in Claude Code

```bash
git clone <this-repo>
cd helmtutorial
```

Open Claude Code in this directory. It will read `AGENTS.md` automatically.

### 2. (Optional) Activate the safety hooks

The hooks are optional. They provide a hard technical block on `helm install` / `helm upgrade` and an audit trail of all helm commands run through Claude Code.

To activate them, you need Python and pytest:

```bash
pip install pytest
pytest .claude/hooks/
```

All tests must pass before the hooks will behave correctly. If you skip this step, Claude's behavior is still governed by the rules in `AGENTS.md` — you just won't have the system-level enforcement layer.

### 3. Verify the chart

```bash
cd charts/myapp
helm lint .
helm template myapp . -f values-staging.yaml
```

Both commands should succeed with no errors.

## What Gets Blocked vs. Prompted

| Command | Behavior |
|---|---|
| `helm install` | Blocked (not allowed under any circumstances) |
| `helm upgrade` | Blocked |
| `helm uninstall` | Blocked |
| `helm rollback` | Blocked |
| `helm lint` | Prompts for approval, then runs |
| `helm template` | Prompts for approval, then runs |
| `helm dependency update` | Prompts for approval, then runs |
| `helm show values` | Prompts for approval, then runs |

Deployment to GKE goes through ArgoCD after PR merge — not through direct Helm commands.

## Guided Exercises

The exercises are in `docs/exercises/`. To start, open Claude Code and say:

```
start the tutorial
```

Claude will guide you through each exercise step by step.

| Exercise | What You'll Learn |
|---|---|
| 01-orient | Read and understand a multi-env chart before touching anything |
| 02-modify | Make targeted changes to values files without affecting other environments |
| 03-add-resource | Add Redis as a subchart dependency, conditioned per environment |
| 04-safety-net | Understand why direct helm install is not the deployment path |
| 05-pr-handoff | Prepare changes for review and hand off to ArgoCD |

## Chart Structure

```
charts/myapp/
├── Chart.yaml             # Chart metadata (add subchart dependencies here)
├── values.yaml            # Base defaults for all environments
├── values-dev.yaml        # Dev: minimal resources, debug logging, no ingress
├── values-staging.yaml    # Staging: 2 replicas, ingress enabled
├── values-prod.yaml       # Prod: 3 replicas, TLS, warn logging
└── templates/
    ├── _helpers.tpl        # Named templates: fullname, labels, selectorLabels
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml        # Conditional: only rendered when ingress.enabled=true
    └── configmap.yaml      # App config: LOG_LEVEL and MAX_CONNECTIONS
```

## Using /helm-check

The primary way to validate chart changes:

```
/helm-check charts/myapp
```

This runs `helm lint`, updates dependencies if needed, renders `helm template`, and gives you a structured summary of what would be deployed. It never installs or upgrades anything.

## Audit Log

If hooks are active, every helm command attempted through Claude Code is logged to:

```
.claude/audit/helm-YYYY-MM-DD.log
```

View today's log:

```bash
cat .claude/audit/helm-$(date +%Y-%m-%d).log | python3 -m json.tool
```

Or filter to see only blocked commands:

```bash
grep '"decision": "BLOCKED"' .claude/audit/helm-$(date +%Y-%m-%d).log
```

## After the Tutorial

To apply this workflow to a real application chart, ask Claude:

```
Help me customize AGENTS.md for my actual application.
```

Claude will ask you about your application, environments, ArgoCD setup, and team conventions, then help you fill in the REPOSITORY-SPECIFIC CONTEXT section.
