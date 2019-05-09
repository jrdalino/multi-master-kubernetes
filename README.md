# multi-master-kubernetes

## Prerequisites
- Ubuntu 16

## Intall Kubernetes Master and nodes

- Get the Docker gpg key:
```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

- Add the Docker repository:
```
$ sudo add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

- Get the Kubernetes gpg key:
```
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

- Add the Kubernetes repository
```
$ cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

- Update your packages
```
$ sudo apt-get update
```

- Install docker, kubelet, kubeadm, and kubectl
```
$ sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu kubelet=1.13.5-00 kubeadm=1.13.5-00 kubectl=1.13.5-00
```

- Hold them at the current version
```
$ sudo apt-mark hold docker-ce kubelet kubeadm kubectl
```

- Add the iptables rule to sysctl.conf
```
$ echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
```

- Enable iptables immediately:
```
$ sudo sysctl -p
```

- Initialize the cluster (run only on the master)
```
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

- Set up local kubeconfig
```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Apply Flannel CNI network overlay
```
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
```

- Join the worked nodes to the cluster
```
$ kubeadm join [your unique string from the kubeadm init command]
```

- Verify the worker nodes have joined the cluster succesfully
```
$ kubectl get nodes
```

- Compare this result of the kubectl get nodes command
```
NAME                            STATUS   ROLES    AGE   VERSION
chadcrowell1c.mylabserver.com   Ready    master   4m18s v1.13.5
chadcrowell2c.mylabserver.com   Ready    none     82s   v1.13.5
chadcrowell3c.mylabserver.com   Ready    none     69s   v1.13.5
```

## Build a Highly Available Kubernetes Cluster

- View the pods in the default namespace with a custom view:
```
$ kubectl get pods -o custom-columns=POD:metadata.name,NODE:spec.nodeName --sort-by spec.nodeName -n kube-system
```

- View the kube-scheduler YAML:
```
$ kubectl get endpoints kube-scheduler -n kube-system -o yaml
```

- Create a stacked etcd topology using kubeadm:
```
$ kubeadm init --config=kubeadm-config.yaml
```

- Watch as pods are created in the default namespace:
```
$ kubectl get pods -n kube-system -w
```

