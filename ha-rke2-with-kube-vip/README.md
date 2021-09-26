# High Available RKE2 With Kube-VIP

## Definition

RKE2, also known as RKE Government, is Rancher's next-generation Kubernetes distribution.

It is a fully conformant Kubernetes distribution that focuses on security and compliance within the U.S. Federal Government sector.

To meet these goals, RKE2 does the following:

1. Provides defaults and configuration options that allow clusters to pass the CIS Kubernetes Benchmark v1.5 or v1.6with minimal operator intervention
2. Enables FIPS 140-2 compliance
3. Regularly scans components for CVEs using trivy in our build pipeline

## What is kube-vip

The kube-vip project provides High-Availability and load-balancing for both inside and outside a Kubernetes cluster

## Instruction

### Prereqs

In order to proceed with this guide, you will need the following:

DNS server or modification of /etc/hosts with the node hostnames and rke2 master HA hostname
firewalld turned off

### Assumptions

| Host | IP | VIP | Notes|
|:----|:----:|:---:|:-----|
|master1|192.168.69.12|192.168.69.200|etcd|
|master1|192.168.69.13|192.168.69.200|etcd|
|master1|192.168.69.14|192.168.69.200|etcd|


If you do not have a DNS server available/configured, the `/etc/hosts` file on each node will need to include the following.

```
192.168.69.12 master1 master1.demo.local
192.168.69.13 master2 master2.demo.local
192.168.69.14 master3 master3.demo.local
192.168.69.200 k8s-ha.demo.local
```

## Install

### RKE2 installation

#### On master1, excute the following commands

``` shell
master1$ mkdir -p /etc/rancher/rke2/

# Create tls-san for kubernetes
master1$ cat > /etc/rancher/rke2/config.yaml << HERE
tls-san:
- master1
- master1.demo.local
- k8s-ha.demo.local
- 192.168.69.200
HERE

# Setup environment
export VIP=192.168.69.200
export TAG=v0.3.8
export INTERFACE=ens3
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock
export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock
export PATH=/var/lib/rancher/rke2/bin:$PATH
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
alias k=kubectl


# Run bootstrap to install rke2
master1$ curl -sfL https://get.rke2.io | sh -

systemctl enable rke2-server
systemctl start rke2-server # Wait about 2-3 minutes for rke2 to be ready

# Install kube-vip installation

# Create kube-vip rbac manifest
curl -s https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/rke2/server/manifests/kube-vip-rbac.yaml

# Pull image with container runtime
crictl pull docker.io/plndr/kube-vip:$TAG

alias kube-vip="ctr run --rm --net-host docker.io/plndr/kube-vip:$TAG vip /kube-vip"

# generate manifest
kube-vip manifest daemonset \
    --arp \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --leaderElection \
    --taint \
    --services \
    --inCluster | tee /var/lib/rancher/rke2/server/manifests/kube-vip.yaml

# check kube-vip pod and daemonset
master1$ k -n kube-system get ds

master1$ k -n kube-system get pod -l name=kube-vip-ds
NAME                READY   STATUS    RESTARTS   AGE
kube-vip-ds-2vjkz   1/1     Running   0          2m1s

# check vip
master1$ ip a s | grep -B 5 ens3
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq state UP group default qlen 1000
    link/ether 02:00:c0:a8:45:0c brd ff:ff:ff:ff:ff:ff
    inet 192.168.69.12/24 brd 192.168.69.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet 192.168.69.200/32 scope global ens3

# get the token from demo-a
master1$  cat /var/lib/rancher/rke2/server/token
K103043ee68dfecaf979b32a5892257b11c8014fa314a5886160e82f0677e6404e1::server:f213955a6d2e3ccc0596de138cb5c2ec
```

#### On master2, excute the following commands

``` shell
master2$ mkdir -p /etc/rancher/rke2

master2$ cat > /etc/rancher/rke2/config.yaml << HERE
token: K103043ee68dfecaf979b32a5892257b11c8014fa314a5886160e82f0677e6404e1::server:f213955a6d2e3ccc0596de138cb5c2ec
server: https://k8s-ha.demo.local:9345
tls-san:
- master2
- master2.demo.local
- k8s-ha.demo.local
- 192.168.69.200
HERE

master2$ curl -sfL https://get.rke2.io | sh -
master2$ systemctl enable rke2-server
master2$ systemctl start rke2-server

```

#### On master2, excute the following commands

``` shell
master3$ mkdir -p /etc/rancher/rke2

master3$ cat > /etc/rancher/rke2/config.yaml << HERE
token: K103043ee68dfecaf979b32a5892257b11c8014fa314a5886160e82f0677e6404e1::server:f213955a6d2e3ccc0596de138cb5c2ec
server: https://k8s-ha.demo.local:9345
tls-san:
- master3
- master3.demo.local
- k8s-ha.demo.local
- 192.168.69.200
HERE

master3$ curl -sfL https://get.rke2.io | sh -
master3$ systemctl enable rke2-server
master3$ systemctl start rke2-server

```
