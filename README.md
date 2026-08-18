# platform-xp-argo-appset

Crossplane XRD package that provides a self-service GitOps deployment golden path for the `urukube` platform. ArgoCD auto-discovers this repo via the `platform-custom-xrds` GitHub topic and deploys it to `crossplane-system` on the orchestrator cluster.

A BU submits one `UArgoAppSet` claim per target EKS cluster. The composition creates a single ArgoCD `ApplicationSet` on the orchestrator cluster that watches every repo in the GitHub org carrying `repoLabel`, deploys any branch matching `branchPattern` into a per-branch namespace, and targets the cluster named `<eksFriendlyName>-eks` — the same naming convention `platform-xp-eks` uses, which also registers that cluster with ArgoCD automatically. Deleting a branch (or it no longer matching `branchPattern`) prunes its Application and namespace.

`branchPattern` is optional (default `""`) — see [Example claims](#example-claims) below for why you'd leave it empty.

Optionally, a claim can also set `previewBranchPrefixes` (e.g. `["feat/", "fix/"]`) to deploy any branch starting with one of those prefixes as a preview environment — always from `dev/values.yaml` regardless of `environment`, into a namespace prefixed `preview-`. This is independent of, and additive to, the `branchPattern` deployment above. At least one of `branchPattern` / `previewBranchPrefixes` must be set, or the generator has no filters and matches nothing.

> **Don't let two claims set the same (or overlapping) `branchPattern` against the same `githubOrg` + `repoLabel`.** Each match produces its own ArgoCD `Application` targeting the same destination namespace — two claims both matching `main`, say, means two Applications both owning (and pruning) the same live resources, which flip-flops `OutOfSync`/`SharedResourceWarning` forever. If a claim just needs to *add* preview-branch coverage on top of an existing fixed-branch claim for the same repos, leave its `branchPattern` empty rather than repeating the other claim's pattern.

## Composition pipeline

1. **`resolve-environment`** — merges `org-defaults` with the BU-specific `EnvironmentConfig` selected by `buId`.
2. **`generate-resources`** — builds the single `ApplicationSet` as a `provider-kubernetes` `Object`, applied in-cluster via the static `kubernetes-incluster` `ProviderConfig`.

## What gets provisioned

Every `UArgoAppSet` claim creates one `argoproj.io/v1alpha1 ApplicationSet` resource, named after the claim, in the `argocd` namespace on the orchestrator cluster itself — the only namespace `provider-kubernetes`'s RBAC (`provider.yaml`) permits it to touch. It has exactly one **SCM Provider (GitHub)** generator, `allBranches: true`, watching every repo carrying the `repoLabel` topic, with one `filters` entry per branch class:

| Branch class | Filter (only present when...) | Deploys to | Values file |
|---|---|---|---|
| Fixed branch | `branchMatch: branchPattern` — when `branchPattern` is non-empty | `<eksFriendlyName>-eks`, namespace `<repo>-<branch-slug>` | `<environment>/values.yaml` |
| Preview branch | `branchMatch` built from each `previewBranchPrefixes` entry — one filter per prefix | `<eksFriendlyName>-eks`, namespace `preview-<repo>-<branch-slug>` | always `dev/values.yaml` |

Every match is pruned automatically when its branch is deleted or stops matching. Branch slugs are normalised as `{{ .branch | replace "/" "-" | lower }}` — e.g. `feat/add-stripe` → `feat-add-stripe`, giving namespace `payments-api-feat-add-stripe` (or `preview-payments-api-feat-add-stripe` for a preview-prefix match).

### One generator, one shared template — not a per-generator override

ArgoCD's Git generator has no branch-enumeration capability — it only reads files/directories at a fixed revision. The correct, working construct for "every branch matching a pattern, across every repo with a topic" is the **SCM Provider generator** itself with `allBranches: true` plus a `filters` list; each `filters[]` entry is OR'd, so one generator naturally covers every branch class at once (the ApplicationSet uses `spec.goTemplate: true` so `{{ .repository }}` / `{{ .branch }}` / `{{ .url }}` are available in the Application template).

An earlier version of this composition used a **second, independent** SCM Provider generator for preview branches, carrying its own per-generator `template` override for the preview namespace/values-file. That looked reasonable but doesn't work: the installed `ApplicationSet` CRD only declares a `template` field under `matrix`/`merge` generators, not on a bare `scmProvider` entry — every apply failed schema validation with `.spec.generators[1].template: field not declared in schema`.

The fix is one generator with a `filters` list covering every branch class, and the per-class differences (namespace, values file, the `preview` label) expressed **once** in the shared top-level `spec.template`, selected at ArgoCD render time via `regexMatch` against a single alternation regex the composition builds from `previewBranchPrefixes` (e.g. `["feat/", "fix/"]` → `^(feat/|fix/).*`). Adding more preview prefixes to a claim later needs no composition change — it's purely a claim-parameter edit.

## Parameters

| Field | Required | Default | Description |
|---|---|---|---|
| `spec.parameters.buId` | Yes | — | Business Unit ID (e.g. `BU001`) |
| `spec.parameters.eksFriendlyName` | Yes | — | `friendlyName` of the target `UEks` claim. Application destination is the cluster named `<eksFriendlyName>-eks`, which must already exist and be `Ready` (`platform-xp-eks` registers it with ArgoCD automatically) |
| `spec.parameters.environment` | Yes | — | Environment name (e.g. `dev`, `staging`, `prod`, or any custom name) — selects the values file at `<helmChartPath>/<environment>/values.yaml` in each matched repo |
| `spec.parameters.awsAccountId` | Yes | — | 12-digit AWS account ID the target cluster lives in. Not consumed by any resource here — the ApplicationSet is applied in-cluster on the orchestrator — carried only as a label |
| `spec.parameters.githubOrg` | Yes | — | GitHub organisation to scan for repos |
| `spec.parameters.repoLabel` | Yes | — | GitHub topic that marks repos eligible for deployment |
| `spec.parameters.branchPattern` | No | `""` | Regex matched against branch names (e.g. `^main$`, `^release/.*`) — every matching branch gets its own Application. Leave empty to disable this filter entirely, e.g. when another claim already owns the fixed-branch deployment for the same repos and this claim should be preview-only |
| `spec.parameters.previewBranchPrefixes` | No | `[]` | List of plain branch-name prefixes (not regex, e.g. `["feat/", "fix/"]`). Any branch starting with one of these gets its own Application too, always using `dev/values.yaml` (regardless of `environment`), in namespace `preview-<repo>-<branch-slug>`. Empty (default) disables preview-branch deployments |
| `spec.parameters.githubOrgPat` | Yes | — | GitHub PAT (repo read + read:org scopes) used by the SCM generator. Stored in a `Secret` the composition creates in `argocd` — **not** read from an externally-managed secret. See warning below. |
| `spec.parameters.helmChartPath` | No | `helm/` | Path to the Helm chart inside each repo |
| `spec.parameters.requeueSeconds` | No | `180` | How often the SCM generator polls GitHub for branch changes |

> **`githubOrgPat` is stored in plaintext in `spec.parameters`** — visible via `kubectl get uargoappset -o yaml`, and in git if the claim manifest is committed. Never commit a real PAT into a claim file; apply it via `kubectl apply`/CI secret injection instead, the same way you'd handle any other raw credential. This is a deliberate tradeoff versus the previous design (which referenced an ESO-managed secret and never put the token value in the claim at all) — the composition now owns creating the `Secret` itself rather than depending on an externally-populated one.

## Example claims

### Fixed branch only (no previews)

The common case — one claim owning the `main` (or `^release/.*`, etc.) deployment for a set of repos:

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
    environment: dev
    awsAccountId: "472506473117"
    githubOrg: urukube
    repoLabel: platform-preview-enabled
    branchPattern: "^main$"
    githubOrgPat: "<inject-at-apply-time, never commit>"
```

Deploys `main` → namespace `payments-api-main` on `bu001-dev-eks`. No `previewBranchPrefixes`, so no preview Applications are created.

### Preview branches only (no fixed branch)

Adds preview-environment coverage for the **same repos** a fixed-branch claim (like the one above) already covers, without re-declaring `branchPattern` and generating a duplicate `main` Application:

```yaml
apiVersion: gitops.platform.urukube.io/v1alpha1
kind: UArgoAppSet
metadata:
  name: bu001-payments-previews-gitops
  namespace: bu001
spec:
  parameters:
    buId: BU001
    eksFriendlyName: bu001-dev
    environment: dev
    awsAccountId: "472506473117"
    githubOrg: urukube
    repoLabel: platform-preview-enabled
    branchPattern: ""
    previewBranchPrefixes: ["feat/", "fix/"]
    githubOrgPat: "<inject-at-apply-time, never commit>"
```

Deploys any `feat/*` or `fix/*` branch → namespace `preview-payments-api-<branch-slug>` on `bu001-dev-eks`, always from `dev/values.yaml`. `main` is left alone — it's the first claim's responsibility.

### Both in one claim

If nothing else already covers the fixed branch for these repos, one claim can do both at once by setting `branchPattern` and `previewBranchPrefixes` together:

```yaml
    branchPattern: "^main$"
    previewBranchPrefixes: ["feat/", "fix/"]
```

This targets the cluster named `bu001-dev-eks` in every example above — that's `<eksFriendlyName>-eks`.

## What an app repo needs to participate

1. Add the GitHub topic matching `repoLabel` (e.g. `platform-preview-enabled`) on the repo settings page.
2. Have a Helm chart at `helmChartPath` (default `helm/`), with a values file at `helmChartPath/<environment>/values.yaml` for each environment it supports (e.g. `helm/dev/values.yaml`).
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

Cascading deletion (namespace + everything in it, not just the `Application` object) depends on the generated `Application` carrying the `resources-finalizer.argocd.argoproj.io` finalizer — without it, ArgoCD would delete only the `Application` CR and orphan the live resources. The composition sets this on the shared template, so it applies to every Application the generator produces, regardless of branch class.

If `previewBranchPrefixes` is set, the same generator also watches for any branch starting with one of those prefixes (e.g. `feat/add-stripe`) and runs this flow for it too, except: the values file is always `dev/values.yaml` (not `<environment>/values.yaml`), and the namespace is `preview-payments-api-feat-add-stripe`. It's pruned the same way — on branch delete or once it no longer starts with a configured prefix.

## In-cluster provider setup

Unlike the AWS providers used by other `platform-xp-*` repos, `provider-kubernetes` here talks to the orchestrator cluster's own API server — there is no cross-account role to assume. The `Provider`, its `provider-kubernetes-incluster` `DeploymentRuntimeConfig`, and the static `kubernetes-incluster` `ProviderConfig` are declared once in `platform-xp-crossplane-shared/provider-kubernetes.yaml` — shared with `platform-xp-eks`, which uses the identical in-cluster provider for its own purposes. Declaring them in both repos' own `provider.yaml` would make ArgoCD see two Applications independently owning the same objects, producing a permanent `SharedResourceWarning` and flip-flopping `OutOfSync` status (each Application's sync overwrites the other's tracking annotation on the live object) — which is exactly what happened before this was split out.

This repo's own `provider.yaml` declares only its scoped `Role`/`RoleBinding`, granting `create`, `update`, `patch`, `delete`, `get`, `list`, `watch` on `argoproj.io/applicationsets` **and** core `secrets` (for the per-claim GitHub token `Secret`) in the `argocd` namespace only, bound to the shared provider's `provider-kubernetes-incluster` service account — it cannot touch any other resource in the cluster. Because the `ProviderConfig` this repo references depends on CRDs the shared `Provider` registers, `platform-xp-crossplane-shared/provider-kubernetes.yaml` sync-waves its `Provider`/`DeploymentRuntimeConfig` ahead of its `ProviderConfig`, and marks the `ProviderConfig` `SkipDryRunOnMissingResource=true` — otherwise ArgoCD's pre-flight validation would abort that Application's entire sync on a cold cluster where the CRD isn't registered yet.

## Files

| File | Purpose |
|---|---|
| `provider.yaml` | Declares this repo's scoped `Role`/`RoleBinding` on `argocd` (`applicationsets` + `secrets`), bound to the shared `provider-kubernetes` install in `platform-xp-crossplane-shared` |
| `xrd.yaml` | Defines the `XUArgoAppSet` / `UArgoAppSet` API and parameter schema |
| `composition.yaml` | Maps a claim to the GitHub token `Secret` and the single `ApplicationSet` `Object` resource |
