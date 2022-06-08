# Introduction

This is a playground that creates a monitoring environment by using gitOps principles.

We gonna create two repositories:

  * fluxcd-monitoring-lab
    this repository has all fluxcd system resources

  * monitoring-fleet
    this repository has all infrastructure monitoring resources (nginx ingress, prometheus, alert-manager, grafana)

## Features

  * k8s cluster running on docker via kind
  * deployment managed by fluxcd
  * multi-environment support
  * configmaps uses merge behavior
  * ingress supported out-of-box
  * grafana dashboard as a code (fluxcd and ingress controller)

## Installation

### 1. Install requirements

* [fluxcd](https://fluxcd.io/)
* [docker](https://www.docker.com/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [kind k8s](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)


### 2. Install k8s playground

```
kind create cluster --name monitoring  --config kind.yaml
kubectl config set-context kind-monitoring
```

### 3. Create a [github token](https://docs.github.com/en/enterprise-server@3.4/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)

```
export GITHUB_USER=<username>
export GITHUB_TOKEN=<token>
```

### 4. bootstrap fluxcd

```
flux bootstrap github --owner=$GITHUB_USER \
  --repository=fluxcd-monitoring-lab \
  --path=clusters/dev \
  --personal
```

### 5. Deploy monitoring stack

```
Requirements:

- 1. clone the fluxcd-monitoring-lab repository
- 2. jump into the fluxcd-monitoring-lab repository folder
- 3. fork the repository: https://github.com/eduardolmedeiros/monitoring-fleet
- 4. clone the monitoring-fleet (fork repository)
- 5. deploy monitoring resources

flux create source git monitoring-fleet \
  --url=https://github.com/<user>/monitoring-fleet \
  --branch=main \
  --interval=1m \
  --export > ./clusters/dev/monitoring-fleet-repo.yaml

flux create kustomization sources \
  --target-namespace=flux-system \
  --source=monitoring-fleet \
  --path="./sources" \
  --prune=true \
  --interval=1m \
  --export > ./clusters/dev/sources.yaml

flux create kustomization infrastructure \
  --target-namespace=flux-system \
  --source=monitoring-fleet \
  --path="./infrastructure/dev" \
  --prune=true \
  --interval=1m \
  --export > ./clusters/dev/infrastructure.yaml

flux create kustomization monitoring \
  --target-namespace=flux-system \
  --source=monitoring-fleet \
  --path="./monitoring/dev" \
  --depends-on=infrastructure \
  --prune=true \
  --interval=1m \
  --export > ./clusters/dev/monitoring.yaml

- 5. add & commit & push all files
```

### 6. update /etc/hosts file (required to access all ingress hosts)

```
# monitoring lab
127.0.0.1 grafana.k8s.local alert-manager.k8s.local prometheus.k8s.local
```

### 7. Access monitoring resources

```
http://grafana.k8s.local
http://prometheus.k8s.local
http://alert-manager.k8s.local
```

## fluxcd useful commands

```
flux get all # get all resources and statuses
flux reconcile kustomization infrastructure # force sync for infrastructure resources
flux reconcile kustomization monitoring # force sync for monitoring resources
flux get helmreleases --all-namespaces # get all helm releases status in all namespaces
flux get helmreleases -n monitoring # get helm release in monitoring namespace
```

## How to update/manage the monitoring resources?

Just update the yaml files from the monitoring-fleet (fork repository)

## How to uninstall?

```
kind delete cluster --name monitoring
```
