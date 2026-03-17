# Exercise 03: Add a Redis Dependency

**Goal:** Add Redis as a Helm subchart dependency, wire it into the values files per environment, and verify the rendered output. Learn how Helm manages external chart dependencies and how to keep per-environment configuration intentional.

---

## Why This Matters

Real applications rarely stand alone. Redis is a common dependency for session storage, caching, and queuing. Helm's subchart mechanism lets you declare an external chart as a dependency and configure it alongside your own values — without managing its manifests manually.

The key skill here is intentionality: every environment should get exactly the Redis configuration it needs, no more.

---

## Step 1: Ask Claude what adding Redis involves

Before making any changes, ask Claude to explain the process:

```
I want to add Redis to this chart as a subchart dependency using the Bitnami chart. What steps are involved? Don't make any changes yet — just explain the process.
```

**What to look for:** Claude should describe three things:
1. Adding a `dependencies` block to `Chart.yaml` with the Bitnami repo and version
2. Running `helm dependency update` to download the subchart into `charts/`
3. Configuring Redis values in `values.yaml` (and per-environment overrides)

If Claude skips step 2 or doesn't mention that `charts/` needs to be populated, ask it to clarify — the chart will fail to render without the dependency downloaded.

---

## Step 2: Add the dependency to Chart.yaml

```
Add Redis as a subchart dependency. Use the Bitnami redis chart, version 19.x. Condition it on a redis.enabled flag so we can turn it on or off per environment.
```

**What to look for:** Claude should add a `dependencies` block to `Chart.yaml` like this:

```yaml
dependencies:
  - name: redis
    version: "19.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

The `condition` field is important — it means Redis only gets deployed when the values file sets `redis.enabled: true`. Without it, Redis would deploy in every environment unconditionally.

---

## Step 3: Configure Redis in the values files

```
Update the values files so that Redis is disabled in dev and enabled in staging and prod. In staging, use a single replica with no persistence. In prod, use a replica count of 1 with persistence enabled. Add the redis.enabled: false key to values.yaml as the default.
```

**What to look for:** Claude should:
- Add `redis.enabled: false` to `values.yaml` (the safe default)
- Add `redis.enabled: false` to `values-dev.yaml` explicitly (belt and suspenders)
- Add `redis.enabled: true` with minimal config to `values-staging.yaml`
- Add `redis.enabled: true` with persistence enabled to `values-prod.yaml`

For staging:
```yaml
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: false
  master:
    persistence:
      enabled: false
```

For prod, persistence should be enabled. Ask Claude what the appropriate `storageClass` would be for GKE standard storage if it doesn't mention it.

---

## Step 4: Download the dependency

Before rendering, the subchart needs to be fetched:

```
Run helm dependency update to fetch the Redis subchart.
```

**What to look for:** Claude should run:

```bash
cd charts/myapp && helm dependency update .
```

The hook will prompt for approval before running this. The command downloads the Redis chart archive into `charts/myapp/charts/`. After it completes, ask Claude to confirm the file is present.

Note: `charts/myapp/charts/` is gitignored — the subchart is fetched from the repository at render time, not committed. This is intentional. Teams that want reproducibility pin the version in `Chart.yaml` and fetch during CI.

---

## Step 5: Validate the chart

```
/helm-check charts/myapp
```

Ask Claude to render with staging and prod values specifically:

```
Render the chart with values-staging.yaml and summarize the Redis resources it produces.
```

**What to look for:** The rendered output for staging should include Redis Deployment and Service resources. The prod render should include those plus a PersistentVolumeClaim for Redis storage. The dev render should have no Redis resources at all.

If `helm lint` warns about the Redis subchart configuration, ask Claude to explain each warning and whether it should be addressed before the PR.

---

## Step 6: Ask about the tradeoffs

```
What are the tradeoffs of using a Bitnami subchart versus managing Redis separately — for example, deploying it from a different chart in a different repo?
```

**What to look for:** Claude should give you a balanced answer:
- Subchart pros: Redis config lives with the app, simpler onboarding, easy per-release configuration
- Subchart cons: Redis lifecycle is coupled to the app release, harder to share Redis across apps, subchart updates require your chart version bump

Neither approach is universally right. The answer depends on whether Redis is shared across services or app-specific.

---

## Debrief

You added an external dependency, conditioned it per environment, and verified the rendered output — all without deploying anything. This is the full local validation cycle for dependency changes.

The subchart approach is one of the more complex Helm patterns. If the rendering looked right across all three environments, you have a solid PR ready to go.

**Next:** Exercise 04 — experience the safety hooks.
