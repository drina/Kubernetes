# Lab 1 - Deploy a Kubernetes Cluster using Kubeadm

#### Navigate to the folder where the repository is cloned, then run the following command to check if we are on the right location where the Vagrantfile is:

```
vagrant status
```

After confirming that we are in the right path, run the following command to spin up the VMs defined in Vagrantfile:
```
vagrant up
```

After 'vagrant up' finishes and all nodes are up, open up new terminals for each of the nodes and connect to them using 'vagrant ssh', for example:
```
vagrant ssh manager
vagrant ssh node01
vagrant ssh node02
```

### Next step is to do the necessary pre-configurations and to install a container runtime. We will follow the official Kubernetes documentation:

- [Install Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Install a container runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

**NOTE:** THE FOLLOWING STEPS NEED TO BE DONE ON **ALL NODES**.

First, execute the below mentioned instructions:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

### Then we need to install a container runtime, for our lab (cluster) we will install containerd
- [Container runtimes - containerd](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)
- [Install containerd](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg 
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
"deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
"$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install containerd.io 
```
After installation of containerd we need to [configure the systemd cgrop driver](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd)

**FIRST REMOVE ALL FROM**
```bash
sudo vi /etc/containerd/config.toml
```
**THEN ADD AND SAVE**
```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```
After adding and saving we need to restart the service:
```
sudo systemctl restart containerd
```

### Next step is [Installing kubeadm, kubelet and kubectl](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)
  
**NEEDS TO BE DONE ON ALL NODES!!!**
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

**DO THE FOLLOWING STEPS ONLY ON MANAGER NODE!!!**

check the ip address (use following command):
```bash
ip add
```
then initialise the cluster where the --apiserver-advertise-address is the ip adress of the manager
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.56.2
```
after the initialization we need to [Installing a Pod network add-on](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)
we will use [weave](https://github.com/weaveworks/weave)
```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```
check for the weave daemonset
```bash
kubectl get ds -A
```
then edit the daemonset 
```bash
kubectl edit ds weave-net -n kube-system
```
add in weave container another env variable:
```bash
- name: IPALLOC_RANGE
  value: 10.244.0.0/16
```

On worker nodes do the join command that was outputted in 'kubeadm init', it will look something like this:
```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

