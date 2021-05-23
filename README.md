# flux2-helm-ecr-cookiecutter

This example assumes a scenario with two environments: staging and production. It also assumes a single
private aplicational Helm chart in-repository, the flux Kustomize configs are kept as simple as possible.

The end goal is to have only the bare minimum to make Flux track the chart and inject diferent values based on the
environment, Helm is intended as the source of truth.


## Prerequisites

You will need a Kubernetes cluster version 1.16 or newer and kubectl version 1.18.
For a quick local test, you can use [Kubernetes kind](https://kind.sigs.k8s.io/docs/user/quick-start/).
Any other Kubernetes setup will work as well though.

In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

Install the Flux CLI on MacOS and Linux using Homebrew:

```sh
brew install fluxcd/tap/flux
```

Or install the CLI by downloading precompiled binaries using a Bash script:

```sh
curl -s https://toolkit.fluxcd.io/install.sh | sudo bash
```

## Repository structure

The Git repository contains the following top directories:

- **flux/app** dir contains Helm releases with a custom configuration per cluster
- **flux/infrastructure** dir contains common infra tools such as NGINX ingress controller and Helm repository definitions
- **flux/clusters** dir contains the Flux configuration per cluster

```
./flux/
├── app
│   ├── base
│   ├── production
│   └── staging
├── clusters
│   ├── production
│   └── staging
└── infrastructure
    └── sources
```

The apps configuration is structured into:

- **flux/app/base/** dir contains the main namespace and the Helm release definition for the app
- **flux/app/production/** dir contains the production Helm release values and suffixes the namespace with `-production`
- **flux/app/staging/** dir contains the production Helm release values and suffixes the namespace with `-staging`

```
./apps/
├── base
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── release.yaml
├── production
│   ├── app-values.yaml
│   └── kustomization.yaml
└── staging
    ├── app-values.yaml
    └── kustomization.yaml
```

In **apps/base/** dir we have a HelmRelease with common values for both clusters:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: app
spec:
  releaseName: mychart
  chart:
    spec:
      chart: helm/
      sourceRef:
        kind: GitRepository
        name: app
        namespace: flux-system
      version: "*"
  interval: 10s
  install:
    remediation:
      retries: 3
  # Place default values here
  values:
    service:
      type: NodePort
```

In **apps/staging/** dir we have a Kustomize patch with the staging specific values and in
**apps/production/** dir we have a Kustomize patch with the production specific values.


Infrastructure:

```
./infrastructure/
├── nginx
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── release.yaml
├── redis
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── release.yaml
└── sources
    ├── bitnami.yaml
    ├── kustomization.yaml
    └── podinfo.yaml
```

In **infrastructure/sources/** dir we have the GitRepository definition:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: app
spec:
  interval: 5m0s
  url: ssh://git@github.com/kita99/flux2-kustomize-helm-minimal-boilerplate
  ref: 
    branch: master
```

Note that with ` interval: 5m` we configure Flux to pull the Helm repository index every five minutes.
If the index contains a new chart version that matches a `HelmRelease` semver range, Flux will upgrade the release.

## Bootstrap staging and production

The clusters dir contains the Flux configuration:

```
.flux/clusters/
├── production
│   ├── apps.yaml
│   └── infrastructure.yaml
└── staging
    ├── apps.yaml
    └── infrastructure.yaml
```

In **flux/clusters/staging/** dir we have the Kustomization definitions:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 15s
  dependsOn:
    - name: infrastructure
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./flux/app/staging
  prune: true
  validation: client
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v1beta1
      kind: HelmRelease
      name: app
      namespace: app-staging
```

Note that with `path: .flux/app/staging` we configure Flux to sync the staging Kustomize overlay and 
with `dependsOn` we tell Flux to create the infrastructure items before deploying the apps.

Fork this repository on your personal GitHub account and export your GitHub access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Verify that your staging cluster satisfies the prerequisites with:

```sh
flux check --pre
```

Set the kubectl context to your staging cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=staging \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=master \
    --personal \
    --path=flux/clusters/staging
```

The bootstrap command commits the manifests for the Flux components in `clusters/staging/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Watch for the Helm releases being install on staging:

```console
$ watch flux get helmreleases --all-namespaces 
NAMESPACE	  NAME	REVISION	SUSPENDED	READY	MESSAGE                          
app    	    app   0.1.0  	  False    	True 	release reconciliation succeeded	
```

Bootstrap Flux on production by setting the context and path to your production cluster:

```sh
flux bootstrap github \
    --context=production \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=master \
    --personal \
    --path=flux/clusters/production
```

Watch the production reconciliation:

```console
$ watch flux get kustomizations
NAME          	REVISION                                        READY
apps          	master/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
flux-system   	master/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
infrastructure	master/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
```

## Encrypt Kubernetes secrets

In order to store secrets safely in a Git repository, you can setup Sealed
Secrets in your cluster and use the kubeseal CLI utility to encrypt Kubernetes
secrets.

Check out [this](https://github.com/bitnami-labs/sealed-secrets#installation) installation
guide for more information.

To check if your setup is working try to fetch the public certificate you'll need to encrypt secrets:

```sh
kubeseal --controller-name=my-sealed-secrets-controller --controller-namespace=my-sealed-secrets-namespace --fetch-cert
```

Generate a Kubernetes secret manifest and encrypt the secret's data field with kubeseal:

```sh
kubectl create secret generic secret-name --dry-run=client --from-literal=foo=bar -o yaml | \
 kubeseal \
 --controller-name=my-sealed-secrets-controller \
 --controller-namespace=my-sealed-secrets-namespace || staging> \
 --format yaml > mysealedsecret.yaml
```

To start adding secrets to an environment create a `secrets.yml` inside the
`flux/app/base/[production || staging]/` directory, populate it with the contents
of `mysealedsecret.yml` and then add it to the `kustomization.yml`: 

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base
  - secrets.yaml # <-
patchesStrategicMerge:
  - app-values.yaml
```

Push the changes:

```sh
git add -A && git commit -m "Add encrypted secret" && git push
```

Trigger a sync to avoid waiting:

```sh
flux sync
```

Verify that the secret has been created:

```sh
kubectl -n app-<production || staging> get secrets
```

You can then use Kubernetes secrets to override values of the Helm app for specific environments:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: app
  namespace: app
spec:
  targetNamespace: app-<production || staging>
  install:
    createNamespace: true
  chart:
    spec:
      version: ">=1.0.0"
  values:
    replicaCount: 2
    usePassword: true
  valuesFrom:
  - kind: Secret
    name: secret-name
    valuesKey: foo
    targetPath: example.secretFoo
```

More info on the specifics of values overrides [here](https://fluxcd.io/docs/components/helm/helmreleases/#values-overrides)

Find out more about Helm releases values overrides in the
[docs](https://toolkit.fluxcd.io/components/helm/helmreleases/#values-overrides).


## Add clusters

If you want to add a cluster to your fleet, first clone your repo locally:

```sh
git clone https://github.com/${GITHUB_USER}/${GITHUB_REPO}.git
cd ${GITHUB_REPO}
```

Create a dir inside `clusters` with your cluster name:

```sh
mkdir -p flux/clusters/dev
```

Copy the sync manifests from staging:

```sh
cp flux/clusters/staging/infrastructure.yaml flux/clusters/dev
cp flux/clusters/staging/apps.yaml flux/clusters/dev
```

You could create a dev overlay inside `apps`, make sure
to change the `spec.path` inside `clusters/dev/apps.yaml` to `path: ./apps/dev`. 

Push the changes to the master branch:

```sh
git add -A && git commit -m "add dev cluster" && git push
```

Set the kubectl context and path to your dev cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=dev \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=master \
    --personal \
    --path=flux/clusters/dev
```

## Identical environments

If you want to spin up an identical environment, you can bootstrap a cluster
e.g. `production-clone` and reuse the `production` definitions.

Bootstrap the `production-clone` cluster:

```sh
flux bootstrap github \
    --context=production-clone \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=master \
    --personal \
    --path=flux/clusters/production-clone
```

Pull the changes locally:

```sh
git pull origin master
```

Create a `kustomization.yaml` inside the `flux/clusters/production-clone` dir:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - flux-system
  - ../production/infrastructure.yaml
  - ../production/apps.yaml
```

Note that besides the `flux-system` kustomize overlay, we also include
the `infrastructure` and `apps` manifests from the production dir.

Push the changes to the master branch:

```sh
git add -A && git commit -m "add production clone" && git push
```

Tell Flux to deploy the production workloads on the `production-clone` cluster:

```sh
flux reconcile kustomization flux-system \
    --context=production-clone \
    --with-source 
```
