# Exercise 05: Prepare for Review

**Goal:** Turn your chart changes into a PR that a reviewer can meaningfully evaluate and that ArgoCD can act on after merge. Learn how to write change summaries that give reviewers what they need without making them re-read the chart from scratch.

---

## Why This Matters

A Helm chart PR without context is hard to review. The reviewer sees diffs in YAML files — values numbers changed, a dependency block appeared, template conditionals were added. Without explanation, they have to reconstruct your intent from the diff.

Claude can help you write a PR description that answers the questions reviewers actually ask: what changed, in which environments, what are the risks, and what did you validate.

---

## Step 1: Get a change summary

Ask Claude to summarize everything you've changed across the exercises:

```
Summarize all the changes we made to the chart during this tutorial. Include which files changed, what changed in each, and which environments are affected.
```

**What to look for:** Claude should produce a summary covering:
- `values-prod.yaml`: replica count increased from 3 to 5
- `values-dev.yaml`: CPU and memory limits reduced
- `Chart.yaml`: Redis subchart dependency added
- `values-staging.yaml` and `values-prod.yaml`: Redis enabled with per-environment config
- `values-dev.yaml`: Redis explicitly disabled

If any changes from earlier exercises were reversed or if you didn't complete all exercises, Claude should reflect the actual state of the files, not what the exercises said to do.

---

## Step 2: Ask Claude to draft a commit message

```
Write a commit message for these changes. Follow the convention: imperative mood, under 72 characters for the subject line, a body that explains the why.
```

**What to look for:** A good commit message for this work might look like:

```
Scale prod replicas and add Redis subchart dependency

Increase prod replica count to 5 to address observed traffic headroom
issues. Add Bitnami Redis 19.x as a conditional subchart dependency,
enabled in staging and prod with environment-appropriate configuration.
Redis is disabled in dev to keep local environments lightweight.

Validated with helm lint and helm template across all three environments.
```

If Claude's draft is vague or doesn't include the "why", ask it to revise with more context.

---

## Step 3: Draft the PR description

```
Write a PR description that a reviewer could use to evaluate these changes. Include: what changed, which environments are affected, what the deployment risk is, and what validation was done.
```

**What to look for:** A complete PR description should include:

**Summary** — what this PR does in 2-3 sentences

**Changes by environment** — a brief table or list:
- Dev: tighter resource limits, no Redis
- Staging: Redis enabled (standalone, no persistence), 2 replicas
- Prod: Redis enabled (with GKE persistent disk), 5 replicas, TLS

**Deployment risk** — something like: "Prod replica count change is low risk (scale-up, no downtime). Redis addition in staging is new infrastructure — first deployment will create a Redis pod. Prod Redis is similarly new; PVC will be created on first sync."

**Validation** — what commands were run: `helm lint` passed, `helm template` rendered correctly for all three environments.

---

## Step 4: Ask what ArgoCD will do

```
When this PR merges to main, what exactly does ArgoCD do? Walk me through the sequence.
```

**What to look for:** Claude should describe:
1. ArgoCD detects the commit on the tracked branch (via polling or webhook)
2. ArgoCD runs `helm template` internally with the appropriate values file for each environment's Application resource
3. ArgoCD computes the diff between the rendered manifests and the current cluster state
4. ArgoCD applies the diff — new resources are created, changed resources are updated, removed resources are deleted
5. ArgoCD reports the sync status; if something fails to apply, the Application enters a Degraded state and the team is alerted

Crucially: ArgoCD acts on each environment's Application independently. Staging and prod are separate ArgoCD Applications, each tracking the same chart but with different values files.

---

## Step 5: Ask what a reviewer should focus on

```
If you were reviewing this PR, what would you check that helm lint and helm template can't catch?
```

**What to look for:** Claude should surface the things automated checks miss:
- **Resource sizing**: Are 5 prod replicas appropriate? Do the resource limits match what the app actually uses?
- **Redis configuration**: Is `auth.enabled: false` acceptable for staging? Should prod have auth enabled?
- **PVC storage class**: Is the default GKE storage class appropriate for Redis persistence?
- **Selector label stability**: Did any changes touch `selectorLabels`? (Changing those forces pod recreation)
- **ArgoCD Application config**: Does the ArgoCD Application resource for each environment reference the right values file? (That config lives in a separate repo — this PR doesn't touch it)

This is the kind of judgment that Claude can flag but that a human reviewer must decide.

---

## Debrief

You've completed the full cycle: read the chart, made targeted changes, added a dependency, understood the safety constraints, and prepared changes for review. The PR description you drafted gives reviewers the context they need to approve quickly and confidently.

ArgoCD takes it from there. Your job ends at the merge.

---

## What's Next

If you want to apply this workflow to a real chart:

1. Ask Claude to help you customize `AGENTS.md` for your specific application using the Interactive Setup Protocol — it will ask you about your cluster setup, environments, team, and ArgoCD configuration.

2. Explore how your ArgoCD Applications are configured. Understanding which values file each Application references is important for the multi-environment pattern to work correctly.

3. If your team hasn't set up hooks yet, consider whether the audit trail would be useful. The hooks are optional, but the log can help you reconstruct what was run in a debugging session.
