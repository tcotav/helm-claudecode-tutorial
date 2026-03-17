# Exercise 02: Modify Per-Environment Settings

**Goal:** Make targeted changes to values files across environments, then validate the chart renders correctly. Learn how changes in one environment do not accidentally affect others.

You'll also be introduced to **skills** — reusable prompt workflows that encapsulate a sequence of steps into a single command. The `/helm-check` skill makes its first appearance in this exercise.

Skills are a Claude Code feature. You can read more about them in the [Claude Code documentation](https://docs.anthropic.com/en/docs/claude-code/slash-commands). To see exactly what `/helm-check` does, read the skill definition at `.claude/skills/helm-check/SKILL.md`.

---

## Why This Matters

Multi-environment Helm charts are only as reliable as their values split. The most common mistake is editing `values.yaml` when you meant to edit only a specific environment's file, or editing the wrong environment file entirely. Claude can help you make changes precisely — but you still need to verify what it rendered.

---

## Step 1: Increase replicas in prod

The team has decided that three replicas are not enough for production. Ask Claude to update it:

```
Increase the replica count in production to 5. Leave dev and staging unchanged.
```

**What to look for:** Claude should edit only `values-prod.yaml`. If it touches `values.yaml` or any other file, ask it why and whether that was intentional. After the change, ask Claude which file it modified and why it chose that file over the others.

---

## Step 2: Tighten dev resource limits

Dev currently allows up to 100m CPU and 128Mi memory. That's fine, but you've noticed dev environments sometimes run long-lived test jobs that consume more than expected. Ask Claude to lower the ceiling:

```
In dev, reduce the CPU limit to 50m and the memory limit to 64Mi. I want to make sure dev containers can't accumulate resources when idle.
```

**What to look for:** Claude should edit only `values-dev.yaml`. Importantly, it should not change `resources.requests` — just `resources.limits`. If it changes both, ask it to explain the difference between requests and limits and whether changing requests was appropriate here.

---

## Step 3: Validate with /helm-check

This step uses `/helm-check` — a Claude Code skill installed in `.claude/skills/helm-check/`. Instead of running `helm lint` and `helm template` as separate commands and interpreting the output yourself, `/helm-check` sequences them automatically and summarizes the result. You invoke it like a slash command directly in Claude Code.

Two commands run under the hood, and they catch different problems:

- **`helm lint`** validates the chart's structure and template syntax without rendering anything. It catches malformed YAML, missing required values, and references to undefined template helpers. It's fast and needs no cluster access.
- **`helm template`** performs the full render: merges values, executes all Go template logic, and outputs the final Kubernetes manifests. This is what ArgoCD will actually apply. Seeing the output is how you confirm your values changes landed where you intended.

Lint can pass while template fails (a values reference that's syntactically valid but undefined at render time). Template can succeed while producing manifests you didn't intend (wrong replica count, missing Ingress). Both together give you confidence. That's why `/helm-check` runs them in sequence.

```
/helm-check charts/myapp
```

**What to look for:** The skill should:
1. Run `helm lint` and report no errors
2. Render `helm template` with the default values
3. Offer to render with environment-specific values files

When it offers to render specific environments, ask it to render staging and prod so you can compare. Verify that:
- Staging shows `replicas: 2` and prod shows `replicas: 5`
- Dev resource limits are `50m` / `64Mi`

---

## Step 4: Ask Claude to explain the diff

After validation, use Claude to summarize what changed and why it matters:

```
Summarize what we changed in this exercise and explain how Helm merges values files so I can explain this to a teammate.
```

**What to look for:** Claude should explain that Helm performs a deep merge: the environment values file overrides only the keys it defines, and the rest come from `values.yaml`. This is why you can set `replicaCount: 5` in `values-prod.yaml` without touching the base file. It should also explain that arrays (like `ingress.hosts`) are replaced entirely, not merged — an important distinction that catches people off guard.

---

## Step 5: Ask about unintended side effects

```
Is there any way that changing values-prod.yaml could accidentally affect staging or dev?
```

**What to look for:** Claude should confirm that environment values files are independent — staging uses `values-staging.yaml`, prod uses `values-prod.yaml`, and they do not inherit from each other. They both inherit from `values.yaml`. The only way to accidentally affect all environments is to edit `values.yaml`. Claude should be able to trace this from the `helm template -f` flag semantics.

---

## Debrief

You made targeted, environment-scoped changes and validated them without deploying anything. The `/helm-check` skill is what you'll run after every set of changes — it catches template errors, missing values, and structural problems before the PR goes out.

Notice that the PR is the deployment trigger, not a manual command. You're preparing a correct chart; ArgoCD does the rest.

**Next:** Exercise 03 — add a Redis dependency.
