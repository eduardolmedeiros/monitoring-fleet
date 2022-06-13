# Introduction

This is a playground that creates a monitoring environment by using gitOps principles.

## Features

  * k8s cluster running on docker via kind
  * deployment managed by fluxcd
  * multi-environment support
  * configmaps uses merge behavior
  * ingress supported out-of-box
  * grafana dashboard as a code (fluxcd and ingress controller)

## Installation

For this playground, we gonna create two repositories:

| Name                       | Description                                                                                                   |
| -------------------------- | ------------------------------------------------------------------------------------------------------------- |
| **fluxcd-monitoring-lab**  | contains fluxcd system manifests files                                                                        |
| **monitoring-fleet**       | contains all infrastructure & monitoring (nginx ingress, prometheus, alert-manager, grafana) manifests files  |
    

### 1. Install requirements

* [fluxcd](https://fluxcd.io/)
* [docker](https://www.docker.com/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [kind k8s](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)


### 2. Install k8s playground

```
curl -o kind.yaml https://raw.githubusercontent.com/eduardolmedeiros/monitoring-fleet/main/kind.yaml
kind create cluster --name monitoring --config kind.yaml
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

#### Deployment

Fork the repository [monitoring-fleet](https://github.com/eduardolmedeiros/monitoring-fleet)

```
gh repo fork --remote https://github.com/eduardolmedeiros/monitoring-fleet
```

Clone the fluxcd-monitoring-lab repository (previously created by the bootstrap command)

```
git clone https://github.com/$GITHUB_USER/fluxcd-monitoring-lab
```


Create all infrastructure & monitoring resources

```
cd fluxcd-monitoring-lab

flux create source git monitoring-fleet \
  --url=https://github.com/$GITHUB_USER/monitoring-fleet \
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
```

### 5 Add & commit & push all files

```
cd fluxcd-monitoring-lab
git add .
git commit -m "setup monitoring environment" -a
git pull
git push
```

### 6. Update /etc/hosts file (required to access all ingress hosts)

```
echo "# monitoring lab
127.0.0.1 grafana.k8s.local alert-manager.k8s.local prometheus.k8s.local" >> /etc/hosts
```

### 7. Access monitoring resources

```
http://grafana.k8s.local
http://prometheus.k8s.local
http://alert-manager.k8s.local
```

## fluxcd useful commands

```
# get all resources and statuses
flux get all 

# force sync for infrastructure resources
flux reconcile kustomization infrastructure

# force sync for monitoring resources
flux reconcile kustomization monitoring

# get all helm releases status in all namespaces
flux get helmreleases --all-namespaces

# get helm release in monitoring namespace
flux get helmreleases -n monitoring 
```

## How to update/manage the monitoring resources?

Just update the yaml files from the monitoring-fleet (fork repository)

Example:

Update the kube-prometheus chart version to the [latest version](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack).


```
cd monitoring-fleet
vim monitoring/dev/prometheus/release-patch.yaml
git commit -m "update kube-prometheus chart version" -a
git pull ; git push
```

Push the changes and check the fluxcd.

```
flux logs
flux get helmreleases -n monitoring 
```

## How to uninstall?

```
kind delete cluster --name monitoring
```
