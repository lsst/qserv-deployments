# qserv-deployments — guide for AI coding agents

This repo is the gitops source of truth for deployed Qserv instances at USDF. It holds
**no application code**: just one directory per deployment under `deployments/`, each
with an Argo CD `Application` (`application.yaml`) and Helm values overrides
(`values.yaml`). A root "app of apps" (`bootstrap/app-of-apps.yaml`) auto-syncs
the child Application CRs into the cluster on every push to `main`, so changes to
`application.yaml` (e.g. `targetRevision` bumps) take effect without manual
`kubectl apply`. However, **workload synchronization is intentionally manual**: the
child apps have no auto-sync, so merging here changes the *desired* state but a human
still reviews the diff in Argo CD and syncs the actual pods/services.

Qserv itself — including the Helm chart (`deploy/helm/`) — lives in
github.com/lsst/qserv, conventionally checked out as a sibling (`../qserv/`). See
`../qserv/doc/architecture/deployment.md` for the full pipeline (images → chart →
this repo → Argo CD) and cluster topology.

## ⚠️ PVC safety — read before every change

The chart's StatefulSets create PVCs via `volumeClaimTemplates` (`worker-data-*`,
`czar-data-*`, `repl-data-*`). The volumes hold catalog data that takes **days to
weeks** to re-ingest; their loss is the single largest operational risk of running
Qserv on Kubernetes, and manual sync exists precisely as a guard. For every change,
state explicitly in the PR what happens to existing PVCs. Danger patterns:

- A `targetRevision` whose chart renames the chart or its StatefulSets changes the
  generated PVC names → new empty volumes bind; the old ones are stranded or
  reclaimed. STS/PVC names derive from the **chart** name, so Application/release
  renames keep PVC names but still force an STS recreate (immutable selector labels);
  a namespace change abandons every PVC in the old namespace.
- Overriding `<component>.storage.class`/`.size` (chart values; not currently set
  here): `volumeClaimTemplates` fields are mostly immutable — changes can force an
  STS delete/recreate.
- Reducing `workers.replicas` leaves the surplus PVCs behind (safe, but they must not
  be "cleaned up" casually — scaling back up rebinds them).
- Never enable auto-sync or prune **on the child (per-environment) Applications**.
  The root app-of-apps has auto-sync enabled, but it only manages Application CRs in
  the `argocd` namespace — it does not touch workload resources. Never suggest
  `helm uninstall`, `kubectl delete sts`, or namespace deletion without a human
  explicitly confirming the fate of each PVC by name.

## Making a deployment change

1. Edit the deployment's `values.yaml` (replica counts, node tiers, ingest
   enablement, external service) and/or `application.yaml`
   (`spec.sources[0].targetRevision` pins the chart version from
   `ghcr.io/lsst/charts`).
2. Each chart release pins matching image tags in its default values, so an upgrade
   is normally just a `targetRevision` bump (these files deliberately stopped
   overriding images). A one-off `qservImageName` override must stay on the same
   release line — the qserv CI job summary prints the exact names.
3. Sanity-render before pushing, from a qserv checkout:
   `helm template ../qserv/deploy/helm -f deployments/usdf-qserv-<env>/values.yaml`
   and diff against the pre-change render, scrutinizing anything under
   `volumeClaimTemplates`, StatefulSet names, or labels/selectors. (For an exact
   render, pull the pinned chart: `helm template oci://ghcr.io/lsst/charts/qserv
   --version <targetRevision> -f ...`.)
4. PR into `main`, stating the PVC impact. After merge, the root app-of-apps
   auto-syncs Application CR changes (e.g. `targetRevision` bumps) into the
   cluster. A human then reviews the resulting workload diff in Argo CD and syncs
   manually. Rollout runs smig schema-migration Jobs before the StatefulSets
   restart.

## Environments

`usdf-qserv-dev`, `usdf-qserv-int`, `usdf-qserv-prod` — separate namespaces on the
USDF cluster; worker/czar/replication pods are pinned to node tiers via the
`qserv.lsst.io/tier` label, and the czar is exposed through a MetalLB LoadBalancer
with an allowlist of source ranges. Promote changes dev → int → prod; prod values
changes deserve the most scrutiny (35 workers in int and prod, 70 in dev as of
2026-08, each with a 10 Ti PVC by chart default).
