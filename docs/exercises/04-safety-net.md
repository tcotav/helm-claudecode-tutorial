# Exercise 04: The Safety Net

**Goal:** Understand what the safety hooks protect against and why direct `helm install` / `helm upgrade` is not the deployment path. Whether or not you have the hooks installed, this exercise explains the model.

---

## Before You Start

The hooks in this repo are optional. If you set them up and verified them, they will enforce the behavior described below at the system level. If you skipped setup, the exercise still works — you'll be asking Claude to explain what the hooks would do rather than watching them trigger.

To check whether your hooks are active:

```bash
pytest .claude/hooks/
```

If all tests pass, the hooks are active. If you don't have pytest or Python set up, skip that step and proceed.

---

## Why This Matters

The `helm install` and `helm upgrade` commands are fast and powerful. That's exactly why they're dangerous in a team environment. A direct `helm upgrade` to the production namespace:
- Has no review step
- Is not tied to a git commit or PR
- Can outpace ArgoCD, causing a state drift that ArgoCD will then undo or fight against
- Leaves no audit trail in your change management system

The hooks — and the workflow they enforce — exist to make the correct path the easy path.

---

## Step 1: Try to install the chart

Paste this into Claude Code:

```
Install the myapp chart to the dev namespace using helm install.
```

**If hooks are active:** The helm-validator hook intercepts the command before it executes and blocks it. Claude Code will show the hook's output explaining that `helm install` is not allowed and must go through your GitOps workflow.

**If hooks are not active:** Claude should decline based on the AGENTS.md instructions — it knows that `helm install` is not part of the workflow in this repo. If it doesn't, ask it directly:

```
Why shouldn't we run helm install directly, even in dev?
```

---

## Step 2: Ask Claude to explain the block

```
Explain what ArgoCD does instead of helm install. How does a chart change actually make it into the cluster?
```

**What to look for:** Claude should describe the GitOps loop:
1. You push a PR with chart changes
2. The PR gets reviewed and merged to the main branch
3. ArgoCD detects the change (polling or webhook) and runs `helm template` internally to compute the desired state
4. ArgoCD applies the diff to the cluster — equivalent to `kubectl apply` on the rendered manifests
5. The cluster reconciles to match the chart

ArgoCD is doing the Helm operations — you are not. Your job is to produce a correct chart.

---

## Step 3: Ask about the escape hatch

```
Can I still run helm install directly in my terminal, outside of Claude Code?
```

**What to look for:** Claude should confirm that yes — the hooks only restrict what Claude Code runs on your behalf. Your terminal is unaffected. The hooks are about keeping AI-assisted changes auditable and in-workflow, not about restricting you as an operator.

This matters because there are legitimate reasons to run Helm directly — debugging a specific cluster state, one-off investigation, local kind/minikube environments. Those use cases exist outside the AI-assisted workflow.

---

## Step 4: Ask about state drift

```
What happens if someone runs helm upgrade directly against the production cluster while ArgoCD is managing it?
```

**What to look for:** Claude should explain that ArgoCD continuously reconciles — meaning if you manually upgrade outside of ArgoCD's control, ArgoCD will detect that the cluster state no longer matches the git repo and mark the application as "OutOfSync". On the next sync, ArgoCD may overwrite the manual change. This is intentional behavior: git is the source of truth, not whatever was last applied to the cluster.

---

## Step 5: Check the audit log (if hooks are active)

```
Is there a record of what was attempted during this session?
```

**What to look for:** Claude should point you to `.claude/audit/helm-YYYY-MM-DD.log`. The log contains JSON entries for every helm command attempted through Claude Code, including the blocked `helm install`, with timestamps and working directories.

View it with:

```
Show me the contents of the helm audit log for today.
```

If hooks are not active, ask Claude to describe what an audit log entry would look like and why an audit trail matters in a shared environment.

---

## Debrief

The safety net is not about distrust — it's about workflow. ArgoCD is the deployment mechanism. Local `helm install` and `helm upgrade` are not. The hooks make this technically enforced rather than policy-enforced, so the constraint holds even when people are in a hurry.

If you chose to skip the hooks, that's fine. The constraint still lives in AGENTS.md, and Claude will respect it. Hooks provide a harder guarantee if your team wants it.

**Next:** Exercise 05 — prepare your changes for review and hand off to ArgoCD.
