# Multi Master Kubernetes

## Kubernetes Cluster Architecture

### Control Plane: Master Node
- API Server: communication hub for all cluster components, It exposes the kubernetes API
- Scheduler: Assigns your app to a worker node. Auto detects which pod to assign to which node based on resource requirements, 
- Controller Manager: Maintains the cluster, Handles nodes failures, replicating components, maintains the cirrect amount of pods
- etcd: Stores cluster configuration

### Data Plane: Worker Node
- kubelet: Runs and manages the containers ont he node and talks to the API Server
- kube-proxy: Load balancers traffic between application components
- container-runtime: The program that runs your containers

### Command Cheat Sheet: Nodes and Pods
- list all the nodes in the cluster
```
$ kubectl get nodes
```

- list pods in all namespaces
```
$ kubectl get pods --all-namespaces
```

- list all the pods in the cluster in detail
```
$ kubectl get pods --all-namespaces -o wide
```

- show all the namespace names
```
$ kubectl get namespaces
```

- deploy Nginx Pod
```
$ cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
EOF
```

- all details about the pod nginx
```
$ kubectl describe pod nginx
```

- delete the pod nginx
```
$ kubectl delete pod ngnix
```

### Command Cheat Sheet: Deployments and Labels
- create deployment from yaml
```
$ kubectl create -f nginx.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

- get the full YAML back
```
$ kubetcl get-deployment nginx-deployment -o yaml
```

- show all pod labels
```
$ kubectl get pods --show-labels
```

- apply a label to a pod
```
$ kubectl label pods <pod name> env=prod
```

- see specific labels
```
$ kubectl get pods -L env
```

- annotate a deployment
```
$ kubectl annotate deployment nginx-deployment mycompany.com/someannotation="jose"
```

- use field selectors
```
$ kubectl get pods --field-selector status.phase=Running
```

### Command Cheat Sheet: Services
- view Pods and their IP Addresses
```
$ kubectl get pods -o wide
```

- create service from yaml
```
$ kubectl create -f nginx-nodeport.yaml
```

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
  selector:
    app: nginx
```

- show the nginx-nodeport service
```
$ kubectl get services nginx-nodeport
```

- access the app over node port
```
$ curl localhost:30123
```

- View pods and their IP addreses
```
$ kubectl get pods -o wide
```

- Execute a command from a pod
```
$ kubectl exec busybox -- curl 10.244.2.9
```

```
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: busybox
    image: radial/busyboxplus:curl
    args:
    - sleep
    - "1000"
EOF
```

## Installation Prerequisites
- Ubuntu 16.04

## Install Kubernetes Master and nodes

### Install on Master and Worder Nodes

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

### Install on Cluster Only

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

## Secure Cluster Notifications

- View the kube-config
```
$ cat .kube/config | more
```

- View the service account token:
```
$ kubectl get secrets
```

- Create a new namespace named my-ns:
```
$ kubectl create ns my-ns
```

- Run the kube-proxy pod in the my-ns namespace:
```
$ kubectl run test --image=chadmcrowell/kubectl-proxy -n my-ns
```

- List the pods in the my-ns namespace:
```
$ kubectl get pods -n my-ns
```

- Run a shell in the newly created pod:
```
$ kubectl exec -it <name-of-pod> -n my-ns sh
```

- List the services in the namespace via API call:
```
$ curl localhost:8001/api/v1/namespaces/my-ns/services
```

- View the token file from within a pod:
```
$ cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

- List the service account resources in your cluster:
```
$ kubectl get serviceaccounts
```

## Test the Cluster

- Run a simple nginx deployment:
```
$ kubectl run nginx --image=nginx
```

- View the deployments in your cluster:
```
$ kubectl get deployments
```

- View the pods in the cluster:
```
$ kubectl get pods
```

- Use port forwarding to access a pod directly:
```
$ kubectl port-forward $pod_name 8081:80
```

- Get a response from the nginx pod directly:
```
$ curl --head http://127.0.0.1:8081
```

- View the logs from a pod:
```
$ kubectl logs $pod_name
```

- Run a command directly from the container:

```
$ kubectl exec -it nginx -- nginx -v
```

- Create a service by exposing port 80 of the nginx deployment:
```
$ kubectl expose deployment nginx --port 80 --type NodePort
```

- List the services in your cluster:
```
$ kubectl get services
```

- Get a response from the service:
```
$ curl -I localhost:$node_port
```

- List the nodes' status:
```
$ kubectl get nodes
```

- View detailed information about the nodes:
```
$ kubectl describe nodes
```

- View detailed information about the pods:
```
$ kubectl describe pods
```

