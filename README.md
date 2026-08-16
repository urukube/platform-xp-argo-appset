# platform-xp-argo-appset

Crossplane XRD package that provides a self-service GitOps deployment golden path for the `urukube` platform. ArgoCD auto-discovers this repo via the `platform-custom-xrds` GitHub topic and deploys it to `crossplane-system` on the orchestrator cluster.

A BU submits one `UArgoAppSet` claim per target EKS cluster. The composition creates a single ArgoCD `ApplicationSet` on the orchestrator cluster that watches every repo in the GitHub org carrying `repoLabel`, deploys any branch matching `branchPattern` into a per-branch namespace, and targets the cluster named `<eksFriendlyName>-eks` — the same naming convention `platform-xp-eks` uses, which also registers that cluster with ArgoCD automatically. Deleting a branch (or it no longer matching `branchPattern`) prunes its Application and namespace.

## Composition pipeline

1. **`resolve-environment`** — merges `org-defaults` with the BU-specific `EnvironmentConfig` selected by `buId`.
2. **`generate-resources`** — builds the single `ApplicationSet` as a `provider-kubernetes` `Object`, applied in-cluster via the static `kubernetes-incluster` `ProviderConfig`.

## What gets provisioned

Every `UArgoAppSet` claim creates one `argoproj.io/v1alpha1 ApplicationSet` resource, named after the claim, in the `argocd` namespace on the orchestrator cluster itself — the only namespace `provider-kubernetes`'s RBAC (`provider.yaml`) permits it to touch:

| Generator | Deploys to | Lifecycle |
|---|---|---|
| SCM Provider (GitHub), `allBranches: true`, filtered by `repoLabel` topic + `branchMatch: branchPattern` | `<eksFriendlyName>-eks`, namespace `<repo>-<branch-slug>` | Pruned automatically when the branch is deleted or stops matching `branchPattern` |

Branch slugs are normalised as `{{ .branch | replace "/" "-" | lower }}` — e.g. `feat/add-stripe` → `feat-add-stripe`, giving namespace `payments-api-feat-add-stripe`.

### Why a single SCM Provider generator instead of a Matrix

ArgoCD's Git generator has no branch-enumeration capability — it only reads files/directories at a fixed revision. The correct, working construct for "every branch matching a pattern, across every repo with a topic" is the **SCM Provider generator** itself with `allBranches: true` plus a `branchMatch` filter; combining it with a second generator via `matrix` would be redundant. The ApplicationSet uses `spec.goTemplate: true` so `{{ .repository }}` / `{{ .branch }}` / `{{ .url }}` are available in the Application template.

## Parameters

| Field | Required | Default | Description |
|---|---|---|---|
| `spec.parameters.buId` | Yes | — | Business Unit ID (e.g. `BU001`) |
| `spec.parameters.eksFriendlyName` | Yes | — | `friendlyName` of the target `UEks` claim. Application destination is the cluster named `<eksFriendlyName>-eks`, which must already exist and be `Ready` (`platform-xp-eks` registers it with ArgoCD automatically) |
| `spec.parameters.awsAccountId` | Yes | — | 12-digit AWS account ID the target cluster lives in. Not consumed by any resource here — the ApplicationSet is applied in-cluster on the orchestrator — carried only as a label |
| `spec.parameters.githubOrg` | Yes | — | GitHub organisation to scan for repos |
| `spec.parameters.repoLabel` | Yes | — | GitHub topic that marks repos eligible for deployment |
| `spec.parameters.branchPattern` | Yes | — | Regex matched against branch names (e.g. `^main$`, `^release/.*`) — every matching branch gets its own Application |
| `spec.parameters.githubOrgPat` | Yes | — | GitHub PAT (repo read + read:org scopes) used by the SCM generator. Stored in a `Secret` the composition creates in `argocd` — **not** read from an externally-managed secret. See warning below. |
| `spec.parameters.helmChartPath` | No | `helm/` | Path to the Helm chart inside each repo |
| `spec.parameters.helmValuesPath` | No | `helm/values.yaml` | Path to the base values file, relative to the repo root |
| `spec.parameters.requeueSeconds` | No | `180` | How often the SCM generator polls GitHub for branch changes |

> **`githubOrgPat` is stored in plaintext in `spec.parameters`** — visible via `kubectl get uargoappset -o yaml`, and in git if the claim manifest is committed. Never commit a real PAT into a claim file; apply it via `kubectl apply`/CI secret injection instead, the same way you'd handle any other raw credential. This is a deliberate tradeoff versus the previous design (which referenced an ESO-managed secret and never put the token value in the claim at all) — the composition now owns creating the `Secret` itself rather than depending on an externally-populated one.

## Example claim

```yaml
apiVersion: gitops.platform.urukube.io/v1alpha1
kind: UArgoAppSet
metadata:
  name: bu001-payments-gitops
  namespace: bu001
spec:
  parameters:
    buId: BU001
    eksFriendlyName: bu001-dev
    awsAccountId: "472506473117"
    githubOrg: urukube
    repoLabel: platform-preview-enabled
    branchPattern: "^main$"
    githubOrgPat: "<inject-at-apply-time, never commit>"
```

This targets the cluster named `bu001-dev-eks`.

## What an app repo needs to participate

1. Add the GitHub topic matching `repoLabel` (e.g. `platform-preview-enabled`) on the repo settings page.
2. Have a Helm chart at `helmChartPath` (default `helm/`).
3. Push to a branch matching `branchPattern`.

## Full flow

```
Developer pushes to a branch matching branchPattern in payments-api repo (topic: platform-preview-enabled)
  │
  ▼
ApplicationSet's SCM generator detects the branch (next poll cycle, ≤ requeueSeconds)
  │
  ▼
ArgoCD creates Application: <claim>-payments-api-<branch-slug>
Deploys helm chart from that branch → namespace payments-api-<branch-slug> on <eksFriendlyName>-eks
  │
Branch is deleted, or a later push no longer matches branchPattern
  │
  ▼
Application deleted → namespace + all resources pruned
```

## In-cluster provider setup

Unlike the AWS providers used by other `platform-xp-*` repos, `provider-kubernetes` here talks to the orchestrator cluster's own API server — there is no cross-account role to assume. `provider.yaml` grants it a dedicated service account (`provider-kubernetes-incluster`) via `DeploymentRuntimeConfig`, scoped by a `Role`/`RoleBinding` to `create`, `update`, `patch`, `delete`, `get`, `list`, `watch` on `argoproj.io/applicationsets` **and** core `secrets` (for the per-claim GitHub token `Secret`) in the `argocd` namespace only — it cannot touch any other resource in the cluster.

The `kubernetes-incluster` `ProviderConfig` (also in `provider.yaml`) is static — one per cluster, shared by every `UArgoAppSet` claim (and, by design, with `platform-xp-eks`'s identically-named `provider-kubernetes` install — see that repo's `provider.yaml` comments), referenced directly in `composition.yaml`.

Because `provider-kubernetes`'s `ProviderConfig` depends on CRDs its own `Provider` registers, `provider.yaml` sync-waves the `Provider`/`DeploymentRuntimeConfig`/`Role`/`RoleBinding` ahead of the `ProviderConfig`, and marks the `ProviderConfig` `SkipDryRunOnMissingResource=true` — otherwise ArgoCD's pre-flight validation aborts the entire Application sync on a cold cluster where the CRD isn't registered yet.

## Files

| File | Purpose |
|---|---|
| `provider.yaml` | Installs `provider-kubernetes:v1.2.6`; declares the `provider-kubernetes-incluster` `DeploymentRuntimeConfig`, its scoped `Role`/`RoleBinding` on the `argocd` namespace, and the static `kubernetes-incluster` `ProviderConfig` |
| `xrd.yaml` | Defines the `XUArgoAppSet` / `UArgoAppSet` API and parameter schema |
| `composition.yaml` | Maps a claim to the GitHub token `Secret` and the single `ApplicationSet` `Object` resource |
