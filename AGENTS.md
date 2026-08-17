# qserv-deployments — guide for AI coding agents

This repo is the gitops source of truth for deployed Qserv instances at USDF. It holds
**no application code**: just one directory per deployment under `deployments/`, each
with an Argo CD `Application` (`application.yaml`) and Helm values overrides
(`values.yaml`). Argo CD watches `main` and reconciles — but **synchronization is
intentionally manual**: merging here changes the *desired* state; a human reviews the
diff in Argo CD and syncs.

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

- Changing the chart `targetRevision` to a version that renames the chart or its
  StatefulSets → generated PVC names change → new empty volumes bind and the old ones
  are stranded or reclaimed. (STS and PVC names derive from the **chart** name, so
  renaming an Application or Helm release keeps PVC names — but it changes immutable
  selector labels and forces an STS delete/recreate.) Changing the target namespace
  abandons every PVC in the old namespace.
- Overriding `<component>.storage.class`/`.size` (chart values — these deployments
  currently don't set them): most `volumeClaimTemplates` fields are immutable, so
  this can force StatefulSet delete/recreate.
- Reducing `workers.replicas` leaves the surplus PVCs behind (safe, but they must not
  be "cleaned up" casually — scaling back up rebinds them).
- Never enable auto-sync or prune. Never suggest `helm uninstall`, `kubectl delete
  sts`, or namespace deletion without a human explicitly confirming the fate of each
  PVC by name.

## Making a deployment change

1. Edit the deployment's `values.yaml` (replica counts, node tiers, ingest
   enablement, external service) and/or `application.yaml`
   (`spec.sources[0].targetRevision` pins the chart version from
   `ghcr.io/lsst/charts`).
2. Image names come from the chart itself: each chart release pins matching image
   tags in its default values, so upgrading a deployment is normally just a
   `targetRevision` bump (these values files deliberately stopped overriding images —
   see the "Remove image values" commit). If you do override `qservImageName` etc.
   for a one-off, take the names from the qserv CI job summary and keep them on the
   same release line as `targetRevision`.
3. Sanity-render before pushing, from a qserv checkout:
   `helm template ../qserv/deploy/helm -f deployments/usdf-qserv-<env>/values.yaml`
   and diff against the pre-change render, scrutinizing anything under
   `volumeClaimTemplates`, StatefulSet names, or labels/selectors. (For an exact
   render, pull the pinned chart: `helm template oci://ghcr.io/lsst/charts/qserv
   --version <targetRevision> -f ...`.)
4. PR into `main`, stating the PVC impact. After merge, a human syncs in Argo CD.
   Rollout runs smig schema-migration Jobs before the StatefulSets restart.

## Environments

`usdf-qserv-dev`, `usdf-qserv-int`, `usdf-qserv-prod` — separate namespaces on the
USDF cluster; worker/czar/replication pods are pinned to node tiers via the
`qserv.lsst.io/tier` label, and the czar is exposed through a MetalLB LoadBalancer
with an allowlist of source ranges. Promote changes dev → int → prod; prod values
changes deserve the most scrutiny (35 workers in int and prod, 70 in dev as of
2026-08, each with a 10 Ti PVC by chart default).
