## Install kubernetes
On server1 - 3
```
$ sudo apt-get update
$ sudo apt-get install docker.io
$ sudo systemctl start docker
$ sudo systemctl enable docker

$ sudo apt-get update
$ sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ # "kubernetes-xenial" can be removed to "kubernetes-focal" in the future
$ sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
$ sudo apt-get update
$ sudo apt-get install kubeadm kubelet kubectl kubernetes-cni
$ sudo swapoff -a
$ sudo vim /etc/fstab
$ # Let me skip to set hostname on each machine
```

Change docker config to use systemd instead of cgroup driver.

```
$ cat << EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

$ /etc/docker/daemon.json
```

On server1 (master node)
```
# Modify /var/lib/kubelet/kubeadm-flags.env like below if --cgroup-driver option was existed.
KUBELET_KUBEADM_ARGS=--cgroup-driver=cgroupfs ...
  ↓  ↓  ↓
KUBELET_KUBEADM_ARGS=--cgroup-driver=systemd ...
```

## Setting up kubernetes environment

```
master$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
> Memorize kubeadm command that the command outputs.
```

On server1 (master node) with **regular user**
```
user@master$ mkdir -p $HOME/.kube
user@master$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
user@master$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Deploy a pod network
user@master$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
> podsecuritypolicy.policy/psp.flannel.unprivileged created
> Warning: rbac.authorization.k8s.io/v1beta1 ClusterRole is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRole
> clusterrole.rbac.authorization.k8s.io/flannel created
> Warning: rbac.authorization.k8s.io/v1beta1 ClusterRoleBinding is deprecated in v1.17+, unavailable in v1.22+; use > rbac.authorization.k8s.io/v1 ClusterRoleBinding
> clusterrolebinding.rbac.authorization.k8s.io/flannel created
> serviceaccount/flannel created
> configmap/kube-flannel-cfg created
> daemonset.apps/kube-flannel-ds created

user@master$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
> Warning: rbac.authorization.k8s.io/v1beta1 ClusterRole is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRole
> clusterrole.rbac.authorization.k8s.io/flannel configured
> Warning: rbac.authorization.k8s.io/v1beta1 ClusterRoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRoleBinding
> clusterrolebinding.rbac.authorization.k8s.io/flannel unchanged

# After a few seconds
user@master$ kubectl get pods --all-namespaces
> NAMESPACE     NAME                            READY   STATUS    RESTARTS   AGE
> kube-system   coredns-f9fd979d6-78ll9         1/1     Running   0          76s
> kube-system   coredns-f9fd979d6-fsqct         1/1     Running   0          76s
> kube-system   etcd-nuc01                      1/1     Running   0          86s
> kube-system   kube-apiserver-nuc01            1/1     Running   0          86s
> kube-system   kube-controller-manager-nuc01   1/1     Running   0          86s
> kube-system   kube-flannel-ds-72jdc           1/1     Running   0          43s
> kube-system   kube-proxy-bhgwn                1/1     Running   0          76s
> kube-system   kube-scheduler-nuc01            1/1     Running   0          86s
```

Run the commands to join the cluster at **worker nodes**.
This command could be see when you run `kubeadm init` command previously on the master node.

```
user@worker$ kubeadm join xxx.xxx.xxx.xxx:6443 --token yyyyyyyyyyyyyyyy \
    --discovery-token-ca-cert-hash sha256:zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz
```

Get nodes on **master node** after `kubeadm join` has succeeded.

```
user@master$ kubectl get nodes
> (You can see the nodes)
```

## Running a pod on kubernetes cluster

Run a sample deployment and a service.

```
user@master$ kubectl run --image=nginx nginx-server --port=80 --env="DOMAIN=cluster"
user@master$ kubectl expose pod nginx-server --target-port=80 --name=nginx-http --type=NodePort
// NOTE: You should create a service against the service not a pod. This is an only test use.
```

Then you can get services and its info.

```
user@master$ kubectl get svc
> NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
> kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        152m
> nginx-http   NodePort    10.108.133.68   <none>        80:30624/TCP   5m11s

user@master$ kubectl describe svc nginx-http
> ......
> NodePort:                 <unset>  30624/TCP
> ......
```

You will find a parameter `NodePort` and exposed port number.
In this example, you can communicate nginx on the pod with a port `30624` on each node that belonging the kubernetes cluster.  
  
You can get a result on any node by running the command like below for example.
```
$ curl http://localhost:30624
```

## Delete service and pod

```
user@master$ kubectl delete svc nginx-http
user@master$ kubectl delete pod nginx-server
```

Reference:  
https://linuxconfig.org/how-to-install-kubernetes-on-ubuntu-20-04-focal-fossa-linux  
https://www.linuxtechi.com/install-kubernetes-k8s-on-ubuntu-20-04/  
https://kubernetes.io/docs/setup/cri/  
https://stackoverflow.com/a/55729536/4307818
