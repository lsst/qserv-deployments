# Qserv Deployments

This repository contains Argo CD deployment configurations for [Qserv](https://github.com/lsst/qserv), the distributed database system for the Vera C. Rubin Observatory.

Qserv itself, including its Helm chart, is maintained in the Qserv repository. This repository contains the configuration describing particular deployed instances of Qserv.

## Repository structure

Each deployment has its own directory containing an Argo CD `Application` declaration and the Helm values specific to that deployment. For example:

```text
deployments/
├── usdf-qserv-int/
│   ├── application.yaml
│   └── values.yaml
└── usdf-qserv-prod/
    ├── application.yaml
    └── values.yaml
```

The `application.yaml` file identifies the Qserv Helm chart and version to deploy, the target Kubernetes namespace, and the associated values file. The `values.yaml` file contains deployment-specific overrides of the defaults supplied by the Qserv Helm chart.

## Deployment model

Deployments are managed by Argo CD. The desired state of each Qserv deployment is recorded in this repository and reconciled by Argo CD with the corresponding Kubernetes resources.

The Qserv Helm chart is published separately from this repository. An Argo CD Application therefore uses multiple sources: the published Qserv Helm chart and the deployment-specific values maintained here.

Changes to a deployment should normally be made by modifying its configuration in this repository rather than by directly modifying resources in the cluster.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).

