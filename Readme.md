# Get me to the Cluster Lab - NGINX+ Ingress Controllers + NGINX+ Edge Servers
> This is a lab environment to demonstrate the configuration and setup of a Kubernetes Cluster, Calico CNI, NGINX+ L4 Edge Servers, and NGINX+ Ingress Controllers. This lab environment is loosely based on the whitepaper https://www.nginx.com/resources/library/get-me-to-the-cluster/.  

Lab Details
- 10.1.1.4 - nginxedge01.f5.local 
- 10.1.1.5 - nginxedge02.f5.local 
- 10.1.1.6 - nginxedge03.f5.local
- 10.1.1.7 - k8scontrol01.f5.local 
- 10.1.1.8 - k8sworker01.f5.local 
- 10.1.1.9 - k8sworker02.f5.local 
- 10.1.1.10 - k8sworker03.f5.local 

- user: user01
- pass: f5agility!

Additional changes that were made from the white paper:
- Ubuntu 22.04 LTS 
- FRR routing package was used instead of Quagga
- 10.1.1.0/24 is the underlay network
- 172.16.1.0/24 will by the POD network to be advertised via iBGP
- Additional DNS resiliency was added by advertising a single /32 route for the KUBE-DNS service instead of the individual kube-dns pod IPs

## Table of Contents

- [K8s control node initialization](#K8s Control Node initialization)
- [Calico installation](#Calico installation)
- [Add K8s worker nodes to cluster](#Worker Nodes K8s initilization)
- [Calico iBGP configuration](#Calico iBGP configuration)
- [NGINX+ Ingress Controller deployment](#NGINX+ Ingress Controller deployment)
- [NGINX+ installation](#NGINX+ installation)
- [FRR installation](#FRR installation)
- [FRR iBGP configuration](#FRR iBGP configuration)
- [iBGP testing](#iBGP testing)
- [NGINX+ Edge DNS resolution](#NGINX+ Edge DNS resolution)
- [NGINX+ Edge L4 configuration](#NGINX+ Edge L4 configuration)
- [NGINX+ HA configurations](#NGINX+ HA configurations)
- [Deploy an App](#Deploy an App)
- [Map an App to NGINX+ Ingress Controller](#Map an App to NGINX+ Ingress Controller)

---
## K8s installation
> During this section, you will initialize the kubernetes cluster on k8scontrol01.f5.local.  Utilize the UDF portal to log in via the web shell.
1. Log into k8scontrol01.f5.local 
2. Switch to user01 (Password: f5agility!)
```shell
su - user01
```
3. Initialize kubernentes cluster
```shell
sudo kubeadm config images pull
sudo kubeadm init --control-plane-endpoint=k8scontrol01.f5.local --pod-network-cidr=172.16.1.0/24
```
4. Note the output of k8s intialization.  Copy to a seperate file to notate how to add additional control/worker nodes to the K8s cluster.  
5. Create kubernetes directories in user01 folder
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
6. Confirm cluster intallation
```shell
kubectl cluster-info
kubectl get nodes
```

---
## Calico installation
During this section, you will be installing Calico as the CNI.  You will also install Calicoctl. Additional Documentation:
- https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart
- https://projectcalico.docs.tigera.io/maintenance/clis/calicoctl/install

1. Download the Calico manifest to the K8s control01 node.  
```shell
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/tigera-operator.yaml -O
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/custom-resources.yaml -O
```
2. Edit the custom-resources manifest to match the 172.16.1.0/24 CIDR pod network.
```shell
nano custom-resources.yaml
```
3. Below is an example of custom-resources.yaml
```shell
# This section includes base Calico installation configuration.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 172.16.1.0/24
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer 
metadata: 
  name: default 
spec: {}
```
4. Deploy Calico CNI
```shell
kubectl create -f tigera-operator.yaml
kubectl create -f custom-resources.yaml
```
5. Confirm Calico deployment.  Confirm that all of the calico pods are in a RUNNING state
```shell
watch kubectl get pods -n calico-system
```
6. Install Calicoctl into the /usr/local/bin/
```shell
cd /usr/local/bin/
sudo curl -L https://github.com/projectcalico/calico/releases/download/v3.24.1/calicoctl-linux-amd64 -o calicoctl
sudo chmod +x ./calicoctl
```
7. Confirm Calicoctl installation
```shell
sudo calicoctl node status
```
8. Switch back to home directory
```shell
cd /home/user01
```
