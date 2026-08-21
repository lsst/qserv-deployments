# Qserv Deployments

This repository contains Argo CD deployment configurations for [Qserv](https://github.com/lsst/qserv), the distributed database system for the Vera C. Rubin Observatory.

Qserv itself, including its Helm chart, is maintained in the Qserv repository. This repository contains the configuration describing particular deployed instances of Qserv.

## Repository structure

```text
bootstrap/
└── app-of-apps.yaml
deployments/
├── usdf-qserv-dev/
│   ├── application.yaml
│   └── values.yaml
├── usdf-qserv-int/
│   ├── application.yaml
│   └── values.yaml
└── usdf-qserv-prod/
    ├── application.yaml
    └── values.yaml
```

Each deployment has its own directory containing an Argo CD `Application` declaration and the Helm values specific to that deployment. The `application.yaml` file identifies the Qserv Helm chart and version to deploy, the target Kubernetes namespace, and the associated values file. The `values.yaml` file contains deployment-specific overrides of the defaults supplied by the Qserv Helm chart.

The `bootstrap/` directory contains the root "app of apps" Application that manages the child Application CRs (see below).

## Deployment model

Deployments are managed by Argo CD. The desired state of each Qserv deployment is recorded in this repository and reconciled by Argo CD with the corresponding Kubernetes resources.

The Qserv Helm chart is published separately from this repository. An Argo CD Application therefore uses multiple sources: the published Qserv Helm chart and the deployment-specific values maintained here.

Changes to a deployment should normally be made by modifying its configuration in this repository rather than by directly modifying resources in the cluster.

## App of apps

A root Argo CD Application (`bootstrap/app-of-apps.yaml`, named "qserv" in the Argo CD UI) watches the `deployments/` directory and auto-syncs the child Application CRs (`qserv-dev`, `qserv-int`, `qserv-prod`) into the cluster whenever changes are pushed to `main`. This means that changes to `application.yaml` files (e.g. chart version bumps) take effect automatically without a manual `kubectl apply`.

The child Applications themselves do **not** have auto-sync enabled — actual workload changes still require a human to review the diff in Argo CD and sync manually.

### Bootstrap

The root app must be applied once to set up this mechanism:

```
kubectl apply -f bootstrap/app-of-apps.yaml
```

After that, it is self-sustaining for all child Application CR changes.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).

