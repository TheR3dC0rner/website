---
title: "Setting up FluxCD and K8s Infrastructure Part II"
summary:  "bootstrapping, k0s, k8s and fluxcd"
date: 2025-09-13
tags: ["deployment", "flux", "k8s", "fluxcd","k0s" ]
draft: false
---

## Setting up FluxCD and K8s Infrastructure Part II
In our last blog  (https://www.th3redc0rner.com/infrastructure/fluxpart1/), we set up FluxCD and put it under git control.  The only service we installed  was Cilium and the Cilium-based load balancer.  In this blog we will install the rest of our needed core infrastructure.

Pods will usually start on the node with the most resources available in a Kubernetes cluster.  There are ways to specifically assign but the ability to move to a new node is a feature we want.   Since we are going to use command and controls, we want them to have storage no matter which node run on.  So let's say we are running an Adaptix instance and for some reason we restart it or it fails.  The new instance might start on a new node.  If we don't define storage for it, when the pod restarts it won't have any of our logs or data.  This is why we need some sort of storage mechanism in the cluster.  


## Setting up the cluster storage 

Longhorn is a fairly easy to manage storage provider for Kubernetes.   Its main advantages are it keeps replicas of its data (similar to RAID 1), can be mounted by any pod from any node, can perform backups to either an NFS share or S3 style bucket, and has a fairly simple UI.  

One of the reasons we need 3 worker nodes is by default Longhorn makes 3 replicas of data, though they don't necessarily have to spread to 3 different nodes in the cluster.  To set up Longhorn we are going to follow similar steps to how we set up Cilium.  We are going to add our helm reference to our resources directory in:

```
infrastructure/base/resources
```

longhorn-chart.yaml


```console
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: longhorn-chart
  namespace: flux-system
spec:
  interval: 5m
  url: https://charts.longhorn.io

```
We have to add this to kustomization.yaml file in the resource directory:


```console
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: longhorn-system
resources:
  - cilium-chart.yaml
  - longhorn-chart.yaml 
```
We will now move back to our base directory and create a directory for the longhorn deployment:

```shell
mkdir infrastructure/base/longhorn/
cd infrastructure/base/longhorn
```

We will now create the base deployment file just like we did for Cilium:

helmrelease.yaml:

```console
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: longhorn
  namespace: flux-system
spec:
  chart:
    spec:
      chart: longhorn
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: longhorn-chart
  interval: 5m50s
  releaseName: longhorn
  targetNamespace: longhorn-system
```

The documentation on the official Longhorn site tells us that this should be installed in the namespace longhorn-system so we should make sure we have a namespace defined for this.   For reference a namespace in Kubernetes is a  mechanism for isolating groups of resources within a single cluster.  We installed Cilium into a built-in namespace for system resources called kube-system but for Longhorn we are defining a namespace.

namespace.yaml

```console
apiVersion: v1
kind: Namespace
metadata:
  name: longhorn-system
```

Every definition needs to have the kustomization.yaml file to match.

kustomization.yaml
```console
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: flux-system
resources:
  - namespace.yaml
  - helmrelease.yaml
```

And to finish off the base of Longhorn we need to modify the kustomization file in infrastructure/base/kustomization.yaml

```console
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - resources
  - cilium
  - longhorn
```

To load Longhorn onto our cluster we are going to do the same thing we did for Cilium and lock it to the latest release so we can control the upgrades.

In the /infrastructure/staging directory we are going to create a longhorn directory:

```shell
mkdir infrastructure/staging/longhorn
cd infrastructure/staging/longhorn
```

At the time of this writing Longhorn was on version 1.9.1 so we are going to lock it into that version:

```console
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: longhorn
  namespace: longhorn-system
spec:
  chart:
    spec:
      chart: longhorn
      version: "1.9.1"
```

Now we are going to add it to the staging kustomization.yaml file.

infrastructure/staging/kustomization.yaml

```console
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base
patches:
  - path: cilium/cilium-patches.yaml
    target:
      kind: HelmRelease
      name: cilium

  - path: longhorn/longhorn-patches.yaml
    target:
      kind: HelmRelease
      name: longhorn
```

For safe measure run use kustomize to make sure there are no errors:

```shell
kustomize build
```

We should see our new chart added

Now its time to commit and let Longhorn install itself

From the infrastructure folder run

```shell
git add .
git commit -m "adding longhorn to the cluster"
git push
```

After 5 minutes you can use flux to make sure everything is installing ok

At this point the tree of the entire repository for the k8scluster looks like this:

```
.
├── README.md
├── apps
│   └── staging
│       └── README.md
├── clusters
│   └── staging
│       ├── applications.yaml
│       ├── flux-system
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       └── infrastructure.yaml
└── infrastructure
    ├── base
    │   ├── cilium
    │   │   ├── cilium-values.yaml
    │   │   ├── helmrelease.yaml
    │   │   ├── kustomization.yaml
    │   │   └── kustomizeconfig.yaml
    │   ├── kustomization.yaml
    │   ├── longhorn
    │   │   ├── helmrelease.yaml
    │   │   ├── kustomization.yaml
    │   │   └── namespace.yaml
    │   └── resources
    │       ├── cilium-chart.yaml
    │       ├── kustomization.yaml
    │       └── longhorn-chart.yaml
    └── staging
        ├── README.md
        ├── cilium
        │   └── cilium-patches.yaml
        ├── kustomization.yaml
        └── longhorn
            └── longhorn-patches.yaml

14 directories, 22 files
```

## Setting up the ingress-controller

We are now going to set up the ingress-controller.  This will map our external websites to applications in the cluster.  In this case it will map our domains to a backend website but specified URIs in the AdaptixC2 profile to the AdaptixC2 instances.

The preferred ingress controller for me is ingress-nginx. With its annotations it's very flexible and fits our needs, such as rewrites.  The ingress controller adds a lot of features people usually handle with .htaccess rewrites in most deployments such as hosting a website for URLs not matching our C2 URLs and putting in whitelists for IPs. To set this up we will follow the similar steps to setting up Longhorn.


Starting back at infrastructure/base/resources we are going to create a new resource file.

infrastructure/base/resources/ingress-nginx-chart.yaml

```console
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: ingress-nginx-chart
  namespace: flux-system
spec:
  interval: 5m
  url: https://kubernetes.github.io/ingress-nginx
```

Now in infrastructure/base/ we are going to create our ingress-nginx directory.  This will include a helmrelease.yaml file, a namespace.yaml file and our kustomization file.

infrastructure/base/ helmrelease.yaml:

```console
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: flux-system
spec:
  chart:
    spec:
      chart: ingress-nginx
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx-chart
  interval: 5m50s
  releaseName: ingress-nginx
  targetNamespace: ingress-nginx
  values:
    controller:
      config:
        use-forwarded-headers: "true"
```

Notice in this release we added the value use-forwarded-headers.  This allows us to pass the clients original ip to our command and control instead of the internal Kubernetes ip ranges.

infrastructure/base/ namespace.yaml:

```console
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - helmrelease.yaml
  - namespace.yaml
admin1@devadmin1:~/dev/k8sclusters/infrastructure/base/ingress-nginx$ cat namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    name: ingress-nginx
```

infrastructure/base/ kustomization.yaml:

```console
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - helmrelease.yaml
  - namespace.yaml
```

This has to be loaded in our main kustomization.yaml for the base directory:


/dev/k8sclusters/infrastructure/base/ kustomization.yaml:

```console
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - resources
  - cilium
  - longhorn
  - ingress-nginx
```


Now we are going to go into our staging directory again and lock down our version numbers.  One item to note is when we role out production we are going to define a default certificate but for staging we will just let ingress-nginx generate its own.  

First we make the directory in infrastructure/staging.

```shell
mkdir infrastructure/staging/ingress-nginx
```

we then will create our patch for version control:
infrastructure/staging/ingress-nginx/ingress-nginx-patches.yaml

```console
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: flux-system
spec:
  chart:
    spec:
      chart: ingress-nginx
      version: "4.13.1"
```

Finally we define this patch in our staging kustomization.yaml file:
/infrastructure/staging/kustomization.yaml:

```console
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base
patches:
  - path: cilium/cilium-patches.yaml
    target:
      kind: HelmRelease
      name: cilium

  - path: longhorn/longhorn-patches.yaml
    target:
      kind: HelmRelease
      name: longhorn

  - path: ingress-nginx/ingress-nginx-patches.yaml
    target:
      kind: HelmRelease
      name: ingress-nginx
```

Now from root of infrastructure we would want to perform our commits:

```shell
git add .
git commit -m "adding ingress-nginx resources"
git push

```

After FluxCD reconciles you should be able to check if your controller is ready

```shell
kubectl get pods -n ingress-nginx
```

You should have your controller pod running

```
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-67bbdf7d8d-npq5b   1/1     Running   0          21m
```

From our jumphost we should be able to use curl and get a 404 response if we try retrieve the root uri from any node on our cluster

```shell
 curl -k https://192.168.200.186

```

Response:

```
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

## Outro 

At this point we should have our core staging infrastructure set up.  When we se tup the production version later, it should go very quickly since almost everything is defined.  

The goal of our next blog is to set up a pipeline to build a container image for AdaptixC2 so we can set it up to be deployable in our cluster. 

