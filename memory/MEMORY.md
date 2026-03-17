# helmtutorial Project Memory

## Project Purpose
Interactive tutorial for using Claude Code with Helm charts. Mirrors tftutorial structure but for Helm/GKE/ArgoCD.

## Key Design Decisions
- Hooks are OPTIONAL (unlike tftutorial where they were required) - README and AGENTS.md are explicit about this
- No terraform anything - settings.json is helm-only
- GitOps deployment model via ArgoCD - helm install/upgrade are never the path
- Multi-environment chart: dev / staging / prod with values-<env>.yaml pattern
- Redis subchart is NOT in the initial chart - adding it is Exercise 03

## Repository Structure
- charts/myapp/ - nginx app, 4 templates, 4 values files (base + 3 envs)
- .claude/hooks/ - copied from ccode_infra_starter (helm-validator, helm-logger, hook_utils, conftest, test_helm_validator)
- .claude/skills/helm-check/ - copied from ccode_infra_starter
- .claude/settings.json - helm hooks only (no terraform hooks, no Edit/Write hook)
- docs/exercises/ - 5 exercises

## Exercise Flow
1. 01-orient - read the chart, understand multi-env structure, ArgoCD explanation
2. 02-modify - change replica counts and resource limits per env, /helm-check validation
3. 03-add-resource - add Bitnami redis subchart (conditioned per env via redis.enabled)
4. 04-safety-net - understand why helm install is blocked, ArgoCD as deployment mechanism
5. 05-pr-handoff - commit message, PR description, what ArgoCD does on merge

## Source Repos
- tftutorial: /Users/tcotav/code/github/tcotav/tftutorial (pattern source)
- ccode_infra_starter: /Users/tcotav/code/github/tcotav/ccode_infra_starter (hooks/skills source)
