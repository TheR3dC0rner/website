---
title: "Setting Up Our First Kubernetes Cluster"
summary:  "Creating some basic ansible scripts as an intro to help our deloyment"
date: 2025-08-30
tags: ["deployment", "k8s", "k0s", "bootstrapping", "bootstrap"]
draft: false
---

# Setting Up Our First Kubernetes Cluster

We are now continuing our series by setting up the main backend of the entire infrastructure, Kubernetes. We are going to use the k0s distribution which has taken away some of the complexity of installing Kubernetes, but remains a fairly vanilla distribution.  Kubernetes has several features we are going to take advantage of.  Listed below are some of the major ones


- Self healing, which makes our command and control more resilient.  
- Resource efficient, allowing us to utilize our hardware and VMs to their full potential.
- When combined with GitOps our configuration in git becomes a source of truth. The built in audit trail makes it an even stronger choice.  

I'm going to use k0s to rollout infrastructure because it is a fairly vanilla Kubernetes distribution which is also easy to deploy and upgrade.   The deployment style and upgrades are very ansible like, which fits nicely into our environment.  

Since this is a staging/dev environment its not going to be made fully highly available but will be enough to test upgrades and develop/test our deployments for production infrastructure.  
I'm going to deploying to 1 master node and 3 worker nodes.  
The specs are as follows:

Master:
```
2 gb of ram 
2 CPUs
28 gb of storage
```
Workers:
```
4 gb of ram
2 CPUs
28 gb of storage
```

## Prepping our machines 

The first thing we are going to do is prep the machines for install.  To this we are going to create a new ansible playbook called cluster_prep.yaml  in our infrascripts repository.

cluster_prep.yaml:


```console
- hosts: all
  become: true

  roles:
    - osupdate

- hosts: workers
  become: true
  tasks:
    - name: install needed packages for Longhorn
      ansible.builtin.apt:
        pkg:
        - open-iscsi
        - nfs-common 

```

Now create the inventory directory and create the file cluster.yaml with your IPs. for your one master and 3 worker nodes.  Mine looks like this:

staging_cluster.yaml

```console

[masters]
dk0sm1 ansible_host=192.168.200.185
[workers]
dk0sw1 ansible_host=192.168.201.175
dk0sw2 ansible_host=192.168.200.186
dk0sw3 ansible_host=192.168.201.41

[nodes:children]
masters
workers
```


Also to access our cluster we are going to need kubectl. When we did the jumphost the setup in a previous article, I left out the steps used to install kubectl.  Since we are on Ubuntu we can use the snap version. It can be installed by doing the following:

```shell
snap install kubectl --classic
snap refresh --hold kubectl
```
In our infrascripts repository create a directory and then cd into it:
```shell
mkdir k0s-clusters
cd k0s-clusters
```

## Creating Our Cluster Configuration 

Now we are going to create the template to install our clustter. To do this perform, run  k0sctl with the flags below: 

```shell
k0sctl init --k0s -n "c2stage" > k0sstaging.yaml
```

I put a "d" in front so i know its my dev staging cluster.  k0s will name its nodes based on the machines hostname so name them as you feel fit.

The k0s option enables the full skeleton so we can modify other settings and the n option lets us name our cluster. 

We are going to edit this file to put each of our IPs and user account that can ssh into each box.

This is my sample config for basic install, the ssh key is referenced in the keypath
```console
apiVersion: k0sctl.k0sproject.io/v1beta1
kind: Cluster
metadata:
  name: k0s-cluster
  user: admin
spec:
  hosts:
  - ssh:
      address: 192.168.200.185
      user: admin1
      port: 22
      keyPath: /home/$USER/.ssh/id_admin1
    role: controller
  - ssh:
      address: 192.168.200.186
      user: admin1
      port: 22
      keyPath: /home/$USER/.ssh/id_admin1
    role: worker
  - ssh:
      address: 192.168.201.175
      user: admin1
      port: 22
      keyPath: /home/$USER/.ssh/id_admin1
    role: worker
  - ssh:
      address: 192.168.201.41
      user: admin1
      port: 22
      keyPath: /home/$USER/.ssh/id_admin1
    role: worker
      
```

We have to change the network to disable the default CNI to come in to prepare it for Cilium instead of kuberouter.

```console
# replace
 ...
   provider: kuberouter
 ...
# with
 ...
   provider: custom
 ...
```

We also want to disable the built in kube-proxy to take full advantage of Cilium, to do this we change these lines:

```console

# replace
...
   network:
      kubeProxy:
         disabled: false
...
# with
...
   network:
      kubeProxy:
         disabled: true
```

## Bootstrapping The Cluster

Finally we can deploy the cluster

```shell
k0sctl apply --config k0sstaging.yaml --no-wait

```
You need the "--no-wait" flag because there is no networking CNI yet and the cluster will otherwise get stuck waiting until the networking is running

Now retrieve the admin config file 
```shell
k0sctl apply --config dk0sctl.yaml --no-wait
k0sctl kubeconfig --config dk0sctl.yaml > ~/.kube/config
```

If you do a kubectl get nodes you will see they are not ready because we have no network CNI but the cluster is up and running
```shell
kubectl get nodes
```

```
NAME     STATUS     ROLES    AGE   VERSION
dk0sw1   Ready      <none>   67s   v1.33.3+k0s
dk0sw2   NotReady   <none>   68s   v1.33.3+k0s
dk0sw3   NotReady   <none>   67s   v1.33.3+k0s

```

## Configuring Networking

Now we are going to set up Cilium
```shell
cilium install --version 1.18.0
```

We should be able to see if Cilium is successful

```shell
cilium status --wait
```

Finally check our nodes again 
```shell
kubectl get nodes
```

We should see the nodes move to Ready status.

You can also make sure Cilium is working correctly by running its test

```shell
cilium connectivity test
```

You should get something similar to this
```
 All 71 tests (623 actions) successful, 44 tests skipped, 0 scenarios skipped
```

Now that our cluster is set up we should commit our config
```shell
git add .
git commit -m "adding k0scluster"
git push
```

## Outro 
Now that CNI is installed, we can use FluxCD, a GitOps tool, to manage Cilium versioning.

We will also use FluxCD to install the following as core infrastructure next

- Enable ipam-lb (Cilium based load balancer)
- Enable wireguard between the nodes (Cilium)
- Install our ingress controller 
- Install persistent storage 

In the next article, we’ll bootstrap FluxCD and put Cilium under GitOps control.
