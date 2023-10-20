# ONOS 
#### SDN Controller 
---
## Required software

In order to run ONOS on a host is required to have installed:
 - **Kubernetes cluster** - a running cluster in order to provide master IP to installation yaml and spread scripts to nodes via DeamonSets,
 - **Docker** - in version **>= 1.24** on all nodes to set up Contrail containers.
 - this example installation was done using Kind (Kubernetes in Docker) on Ubuntu 18. 

Linux updates
```sh
apt-get update
apt-get upgrade
```

Docker installation

```sh
apt install docker.io
```

KinD instalation

Install GO
```sh 
wget https://dl.google.com/go/go1.14.2.linux-amd64.tar.gz
tar -xzf go1.14.2.linux-amd64.tar.gz
rm go1.14.2.linux-amd64.tar.gz
mv go /usr/local
```
Prepare profile file for GO
```sh 
cat << 'EOF' >> ~/.profile
export GOROOT=/usr/local/go
export GOPATH=~/go/kind
export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
EOF
```
```sh 
source ~/.profile

GO111MODULE="on" go get sigs.k8s.io/kind@v0.8.0
```

Install Helm
```sh 
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
apt-get update
apt-get install helm
```

Installation Kubectl

```sh
snap install kubectl --classic
```


## Installation


Seccomp activation

For ONOS installation seccomp option -  computer security facility in the Linux kernel must be enabled

```sh
mkdir ./profiles
curl -L -o profiles/audit.json https://k8s.io/examples/pods/security/seccomp/profiles/audit.json
curl -L -o profiles/violation.json https://k8s.io/examples/pods/security/seccomp/profiles/violation.json
curl -L -o profiles/fine-grained.json https://k8s.io/examples/pods/security/seccomp/profiles/fine-grained.json
ls profiles
```
```sh
curl -L -O https://k8s.io/examples/pods/security/seccomp/kind.yaml
```




```sh
vi kind.yaml
```
For single node installation

```
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
- role: control-plane
  extraMounts:
  - hostPath: "./profiles"
    containerPath: "/var/lib/kubelet/seccomp/profiles"
```

For High availability (HA) installation

```
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
  extraMounts:
  - hostPath: "./profiles"
    containerPath: "/var/lib/kubelet/seccomp/profiles"
```

Create cluster

```sh
kind create cluster --name onos-classic --config=kind.yaml  --image=kindest/node:v1.23.6@sha256:b1fa224cc6c7ff32455e0b1fd9cbfd3d3bc87ecaa8fcb06961ed1afb3db0f9ae
```


Kubectl access

```sh
kind get kubeconfig --name=onos-classic > ~/.kube/kind

export KUBECONFIG=~/.kube/kind
```

Adding helm repo

```sh
helm repo add cord https://charts.opencord.org

helm repo add atomix https://charts.atomix.io

helm repo add onosproject https://charts.onosproject.org

helm repo update

helm search repo onos
```

Create namespace

```sh
kubectl create namespace micro-onos

helm install -n kube-system atomix-controller atomix/atomix-controller 

helm install -n kube-system atomix-raft-storage atomix/atomix-raft-storage

helm install -n kube-system onos-operator onosproject/onos-operator
```

ONOS Installation


For single node
```sh
helm -n micro-onos install onos-classic onosproject/onos-classic --set atomix.replicas=0 --set replicas=1
```

For HA version
```sh
helm -n micro-onos install onos-classic onosproject/onos-classic --set atomix.persistence.enabled=false
```
## Checking status

Installation verification

```sh
kubectl -n micro-onos get pods
```

Single pod

```sh
NAME                          READY   STATUS    RESTARTS      AGE
onos-classic-onos-classic-0   1/1     Running   3 (14m ago)   6d5h
```

HA
```sh
NAME                          READY   STATUS    RESTARTS        AGE
onos-classic-atomix-0         1/1     Running   4 (2m48s ago)   5h14m
onos-classic-atomix-1         1/1     Running   2 (3m51s ago)   5h14m
onos-classic-atomix-2         1/1     Running   2 (2m55s ago)   5h14m
onos-classic-onos-classic-0   1/1     Running   1 (7m34s ago)   5h14m
onos-classic-onos-classic-1   1/1     Running   2 (7m32s ago)   5h14m
onos-classic-onos-classic-2   1/1     Running   2 (7m36s ago)   5h14m
```

CLI access

```sh
kubectl -n micro-onos port-forward $(kubectl -n micro-onos get pods -l app=onos-classic -o name | cut --delimiter $'\n' --fields 1) 8101
```

```sh
ssh -p 8101 onos@localhost
```
Credentials

The credentials are by default:

| Login | Password |
| ------| ------   |
| onos  | rocks    |


## Installation of ONOS using Assist-IoT packets components


The easiest method of ONOS installation is based on usage created within Assist-IoT project helm charts from project' repository. Using this method we can install ONOS V2.4 used e.g. by auto-configurable-network enabler 


```sh
git clone https://github.com/assist-iot/sdn_controller.git
cd sdn-controller/deployment/helm/ 
helm install onos ./sdn-controller
```


## Dashboard

GUI activation

By default GUI in this ONOS versin ins not active It must be acttivated using ONOS CLI

```sh
onos@root > app-ids |grep gui

id=177, name=org.onosproject.gui
id=198, name=org.onosproject.openstacknetworkingui
id=202, name=org.onosproject.yang-gui
id=333, name=org.onosproject.gui2
```
```sh
app activate org.onosproject.gui2
```

GUI access

GUI port forwarding
```sh
kubectl -n micro-onos port-forward $(kubectl -n micro-onos get pods -l app=onos-classic-onos-classic -o name) --address <ip address of Kind VM> 8181
```

```
http://localhost:8181/onos/ui/
```


Stan
