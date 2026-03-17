# Exercise 01: Orient Yourself

**Goal:** Use Claude as a chart reader before touching anything. This is how you should start every session in an unfamiliar Helm chart repo.

---

## Why This Matters

The instinct when working with an AI coding tool is to immediately ask it to make changes. Resist that. Claude can explain complex Helm chart structures in plain English, which means you can understand what you're working with before you break anything.

Helm charts have more moving parts than they look — values cascade across files, templates share helpers, and environment differences are easy to miss. Orienting first costs nothing and prevents real confusion later.

---

## Step 1: Get the big picture

Paste this into Claude Code:

```
What does this chart do? Give me a plain English summary — what Kubernetes resources get created, what application runs, and how is it configured.
```

**What to look for:** Claude should describe the nginx Deployment, the Service, the ConfigMap, and the conditional Ingress. It should mention that the Ingress is disabled by default and only active in staging and prod. If it mentions redis, that's coming — we haven't added it yet.

---

## Step 2: Understand the environment split

```
This chart has four values files: values.yaml, values-dev.yaml, values-staging.yaml, and values-prod.yaml. Explain how they relate to each other and what's different in each environment.
```

**What to look for:** Claude should explain that `values.yaml` contains defaults shared across all environments, and the environment-specific files only override what differs. Key differences to note: replica count (1 → 2 → 3), resource limits (minimal in dev, higher in prod), ingress (off in dev, on in staging and prod), log level (debug → info → warn), and TLS (prod only).

---

## Step 3: Ask about the ConfigMap

```
How does application configuration get into the container? Walk me through the ConfigMap and how it connects to the Deployment.
```

**What to look for:** Claude should trace the path from `values.yaml` → `configmap.yaml` → `deployment.yaml`. The ConfigMap holds `LOG_LEVEL` and `MAX_CONNECTIONS`, and the Deployment mounts it via `envFrom.configMapRef`. Claude should note that the ConfigMap name is derived from the release name using the `myapp.fullname` helper template — which is why changing the release name changes the ConfigMap name.

---

## Step 4: Ask about what's missing

```
If I ran helm template on this chart right now, which environments would get an Ingress resource and which wouldn't? Why?
```

**What to look for:** Claude should identify that `ingress.enabled` is `false` in `values.yaml` and `values-dev.yaml`, so a default render and a dev render produce no Ingress resource. Staging and prod set `ingress.enabled: true`, so they do. Claude should be able to walk through the `{{- if .Values.ingress.enabled -}}` guard in `ingress.yaml`.

---

## Step 5: Ask about ArgoCD

```
I notice there's no helm install or helm upgrade command anywhere in this repo. How does the chart actually get deployed to GKE?
```

**What to look for:** Claude should explain (based on the AGENTS.md context) that ArgoCD watches this git repository. When you merge a PR with chart changes, ArgoCD detects the change and reconciles the cluster state to match. You never run `helm install` directly — the cluster pulls from git. This is what "GitOps" means in practice.

---

## Debrief

You just used Claude as a reader. This is one of its most valuable uses — you can ask these kinds of questions about any Helm chart repo and come away with a real picture of what it does before touching anything.

Notice that Claude answered based on the actual templates and values files, not generic knowledge about Helm. It can trace the exact path from a value in `values-prod.yaml` through to the Kubernetes manifest that would land in the cluster.

**Next:** Exercise 02 — make your first change.
