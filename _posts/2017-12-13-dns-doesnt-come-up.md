---
layout: post
title: Quirks/failures with Kubernetes bringup
category: posts
---

Jotting down some notes to help recall various pitfalls/errors observed when bringing up Kubernetes

## With CRIO

When doing a kubeadm reset, appears that things aren't necessarily cleaned up the same as when docker
is used. To make sure all ports are released, ```ps -ae | grep kub```, and kill any remaning
kube* processes.

This is observed with CRIO 1.0.4 and K8S v1.8.3

## kube-dns stuck in ContainerCreating state

On my system, I would laucnh a cluster using kubeadm and then immediately create a flannel
configuration:

```
sudo -E kubeadm init --pod-network-cidr 10.244.0.0/16
export KUBECONFIG=/etc/kubernetes/admin.conf
sudo -E kubectl create -f "k8s/kube-flannel-rbac.yml"
sudo -E kubectl create --namespace kube-system -f "k8s/kube-flannel.yml"
sudo -E kubectl get pods --all-namespaces -w
```

Have observed that even though the flannel containers start up, the DNS is still stuck:

```
$ sudo -E kubectl get pods --all-namespaces                                                                                         
NAMESPACE     NAME                                        READY     STATUS              RESTARTS   AGE
kube-system   etcd-eernstworkstation                      1/1       Running             0          2m
kube-system   kube-apiserver-eernstworkstation            1/1       Running             0          2m
kube-system   kube-controller-manager-eernstworkstation   1/1       Running             0          2m
kube-system   kube-dns-545bc4bfd4-4c82k                   0/3       ContainerCreating   0          3m
kube-system   kube-flannel-ds-rwk28                       2/2       Running             0          3m
kube-system   kube-proxy-qt2bw                            1/1       Running             0          3m
kube-system   kube-scheduler-eernstworkstation            1/1       Running             0          2m
```

Checking journalctl, we see that it is trying to grab an address from cbr0.  This is expected,
given that the /etc/cni/net.d conf is as follows:

```
sudo cat /etc/cni/net.d/10-flannel.conf 
{
  "name": "cbr0",
  "type": "flannel",
  "delegate": {
    "isDefaultGateway": true
  }
}
```

Snippet from journal, ```journalctl --reverse -u kubelet```:

```
Dec 13 08:40:22 eernstworkstation kubelet[159502]: E1213 08:40:22.475516  159502 remote_runtime.go:92] RunPodSandbox from runtime service failed: rpc error: code = Unknown desc = NetworkPlugin cni failed 
Dec 13 08:40:22 eernstworkstation kubelet[159502]: W1213 08:40:22.323451  159502 helpers.go:847] eviction manager: no observation found for eviction signal allocatableNodeFs.available
Dec 13 08:40:22 eernstworkstation kubelet[159502]: E1213 08:40:22.323179  159502 cadvisor_stats_provider.go:87] Partial failure issuing cadvisor.ContainerInfoV2: partial failures: ["/kubepods/burstable/po
Dec 13 08:40:22 eernstworkstation kubelet[159502]: E1213 08:40:22.221520  159502 cni.go:250] Error while adding to cni network: no IP addresses available in network: cbr0
Dec 13 08:40:22 eernstworkstation kubelet[159502]: E1213 08:40:22.221486  159502 cni.go:301] Error adding network: no IP addresses available in network: cbr0
```
