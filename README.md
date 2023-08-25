# Kubernetes GitOps Repo

This is an example on how I would structure a 1:1 (repo-to-single cluster)
git repo for a Kubernetes cluster. This is based on the [Christian Hernandez Kubernetes](https://github.com/christianh814/example-kubernetes-go-repo)
repo and is using Argo CD.

# Structure

Below is an explanation on how this repo is laid out. 

```shell
cluster-one
├── applications
│   └── nginx
│       ├── base
│       │   ├── deployment.yaml
│       │   ├── kustomization.yaml
│       │   ├── namespace.yaml
│       │   └── service.yaml
│       └── overlays
│           ├── dev
│           │   └── kustomization.yaml
│           ├── prod
│           │   └── kustomization.yaml
│           └── uat
│               └── kustomization.yaml
├── bootstrap
│   ├── base
│   │   ├── argocd-ns.yaml
│   │   └── kustomization.yaml
│   └── overlays
│       └── default
│           └── kustomization.yaml
├── components
│   ├── applicationsets
│   │   ├── applications-appset.yaml
│   │   ├── core-components-appset.yaml
│   │   └── kustomization.yaml
└── core
    └── gitops-controller
        └── kustomization.yaml
```

|#|Directory Name|Description|
|---|----------------|-----------------|
| 1. |`cluster-one`| This is the cluster name. This name should be unique to the specific cluster you're targeting. If you're using CAPI, this should be the name of your cluster, the output of `kubectl get cluster`|
| 2. | `bootstrap` | This is where bootstrapping specifc configurations are stored. These are items that get the cluster/automation started. They are usually install manifests.<br /><br />`base` is where are the "common" YAML would live and `overlays` are configurations specific to the cluster.<br /><br />The `kustomization.yaml` file in `default` has `cluster-one/components/applicationsets/` as a part of it's `bases` config.|
| 3. | `components` | This is where specific components for the GitOps Controller lives (in this case Argo CD).<br /><br />`applicationsets` is where all the ApplicationSets YAMLs live.<br /><br />Other things that can live here are RBAC, Git repo, and other Argo CD specific configurations (each in their repsective directories).|
| 4. | `core` | This is where YAML for the core functionality of the cluster live. Here is where the Kubernetes administrator will put things that is necissary for the functionality of the cluster (like cluster configs or cluster workloads).<br /><br />Under `gitops-controller` is where you are using Argo CD to manage itself. The `kustomization.yaml` file uses `cluster-one/bootstrap/overlays/default` in it's `bases` configuration. This `core` directory gets deployed as an applicationset which can be found under `cluster-one/components/applicationsets/core-components-appset.yaml`.<br /><br />To add a new "core functionality" workoad, one needs to add a directory with some yaml in the `core` directory.|
| 5. | `applications` | This is where the workloads for this cluster live.<br /><br />Similar to `core`, the `applications` directory gets loaded as part of an ApplicationSet that is under `cluster-one/components/applicationsets/applications-appset.yaml`.<br /><br />This is where Devlopers/Release Engineers do the work. They just need to commit a directory with some YAML and the applicationset takes care of creating the workload.<br /><br />|

# Testing

To see this in action, first get yourself a cluster (using [k3d](k3d.io) as an example)

```shell
k3d cluster create cluster-one
```

Then, just apply this repo.

```shell
until kubectl apply -k https://github.com/paoloyx/example-kubernetes-go-repo/cluster-one/bootstrap/overlays/default; do sleep 3; done
```

This should give you 4 applications

```shell
$ kubectl get applications -n argocd
NAME                          SYNC STATUS   HEALTH STATUS
nginx                         Synced        Healthy
gitops-controller             Synced        Healthy
```

Backed by 2 applicationsets

```shell
$ kubectl get appsets -n argocd
NAME      AGE
cluster   110s
applications   110s
```

To see the Argo CD UI, you'll first need the password

```shell
kubectl get secret/argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d ; echo
```

Then port-forward to see it in your browser (using `admin` as the username).

```shell
kubectl -n argocd port-forward service/argocd-server 8080:443