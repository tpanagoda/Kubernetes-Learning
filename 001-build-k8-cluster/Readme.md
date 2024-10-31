# Building a Kubernetes Cluster

# Building a Kubernetes Cluster with Kubeadm on Ubuntu Servers

This article will include a step-by-step lab guide to setting up a Kubernetes cluster using `kubeadm` on three Ubuntu servers. Here we will set 1 control node and 2 worker nodes (You can set 1 worker node as well). Following this guide, you can set up your Kubernetes cluster from scratch. I have created this article by referring to the official documentation and some other references including ChatGPT, I have tried this in a test environment and indicated the steps I have tried. If you are doing this in a production environment there will be more factors to consider than indicated below.

***Reference***

[Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

[Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

## Prerequisites:

- **3 Ubuntu Servers** (1 for Control Plane, 2 Worker Nodes)
- **Basic Linux Knowledge.**
- **Internet Connectivity** on the Servers to install dependencies.
- **SSH Access** to all 3 nodes.
- **3 Private IP Addresse**s (1 for each node)

## Table of Contents

1. [Overview] (#overview)
2. [Setting up the Hosts] (#setting-up-the-hosts)
3. [Install Docker and Kubernetes Dependencies] (#install-docker-and-kubernetes-dependencies)
4. [Initialize Control Plane] (#initialize-control-plane)
5. [Join Worker Nodes to the Cluster] (#join-worker-nodes-to-the-cluster)
6. [Verify the Cluster] (#verify-the-cluster)

## Overview

The below guide will guide you through setting up a Kubernetes cluster using `kubeadm` We will have 1 Control Plane and 2 Worker Nodes which Control Plane will manage the Kubernetes Cluster and 2 worker nodes will run the applications. We must install relevant dependencies (`Docker, kubeadm, kubelet, kubectl`), then configure the nodes and initialize the cluster.

## Setting up the Hosts

### Step 1: Set hostnames for all the servers

On each node, we will set a unique hostname so we can easily identify each server.

**On the Control Plane**

```bash
sudo hostnamectl set-hostname k8s-control
```

![image.png](assets/image.png)

**On the First Worker Node**

```bash
sudo hostnamectl set-hostname k8s-worker1
```

![image.png](assets/image%201.png)

**On the Second Worker Node**

```bash
sudo hostnamectl set-hostname k8s-worker2
```

![image.png](assets/image%202.png)

### Step 02: Update the `/etc/hosts`

This is to make sure all nodes can communicate using their hostnames. We will add the private IP and hostname of each node to the file `/etc/hostnames` of all nodes.

```bash
sudo vi /etc/hosts
```

```bash
<control-plane-IP> k8s-control
<worker-node1-IP> k8s-worker1
<worker-node2-IP> k8s-worker2
```

![image.png](assets/image%203.png)

Do the same for all the nodes.

## Install Docker and Kubernetes Dependencies

### Step 03: Install Docker on All Nodes

We have to install Docker on all nodes to run Containers in Kubernetes

```bash
sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg lsb-release apt-transport-https
```

When installing docker, the above dependencies are crucial. Especially packages like `ca-certificates, curl, GnuP, lsb-release, apt-transport-https` ensure secure connections, enable package verification, support compatibility checks, and most importantly provide necessary tools for installation and troubleshooting. 

![image.png](assets/image%204.png)

Setup the Docker engineer repository, add the Docker GPG Key, and set up the Repository.

```bash
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker Engine, Containerd, and Docker Compose.

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

You need to add your current user to the Docker group.

```bash
sudo usermod -aG docker $USER
```

Log out and log in again to effect the changes to the user modification.

![image.png](assets/image%205.png)

You can verify whether Docker is installed correctly by running the below command.

```bash
docker --version
```

![image.png](assets/image%206.png)

You might need to load kernel modules and modify some system settings while setting up this. This has to be done in all the nodes. If you haven’t done these steps there will be some issues will occur when you initialize the Kubernetes control nodes and join the worker nodes.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Setup the below `sysctl` params

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

You can apply the `sysctl` params without a reboot.

```bash
sudo sysctl --system
```

Comment the `disabled_plugins` in your `config.toml` file and restart the `containerd`

```bash
sudo sed -i 's/disabled_plugins/#disabled_plugins/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

Configure the `containerd` and set `SystemdCgroup = true` and restart the `containerd` 

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

### Step 04: Install Kubernetes (`kubeadm, kubelet, kubectl`)

You can install the Kubernetes components on all Nodes.

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

![image.png](assets/image%207.png)

At the end of this command, we are holding all these Kubernetes packages being updated automatically. This will ensure version consistency in your cluster among all the nodes until you decide to update the packages manually when required.

### Step 05: Disable Swap in All Nodes

Kubernetes required the swap to be disabled.

```bash
sudo swapoff -a
```

Kubernetes is required to disable the swap mainly related to resource management and performance predictability. Swap can interfere with managing the resources especially memory for containers and pods. Disabling swap ensures containers will use only their allocated memory allowing better performance and improved isolation between pods.

## Initialize Control Plane

### Step 06: Initialize the Kubernetes Control Plane

Once all the above steps are done we can initialize the Kubernetes Cluster in the Control Plane using `kubeadm`

```bash
sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.29.1
```

![image.png](assets/image%208.png)

<aside>
💡

**At this point, if you get any errors, Please refer to the Troubleshooting section end of this article.**

</aside>

Once the initialization is completed, set the `kubeconfig` file for the `kubectl` command. 

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Then verify whether the `kubectl` command works.

```bash
kubectl get nodes
```

![image.png](assets/image%209.png)

### Step 07: Install the Calico network plugin

You need to install a network plugin in the Control plane to manage the pod communication.

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

![image.png](assets/image%2010.png)

After you install the Calico Network plugin you will see the Control Plane in `Ready` State.

```bash
Kubectl get nodes
```

![image.png](assets/image%2011.png)

## Join Worker Nodes to the Cluster

### Step 08: Get the join command

Run the below command in the Control plane to get the join command for worker nodes. 

```bash
kubeadm token create --print-join-command
```

Then you will get the join command as below.

![image.png](assets/image%2012.png)

### Step 09: Join Worker Nodes

On each worker node run the command from the above step to join the worker nodes to the cluster.

```bash
sudo kubeadm join <control-plane-IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

![image.png](assets/image%2013.png)

## Verify the Cluster

### Step 10: Verify the node status

```bash
kubectl get nodes
```

You should see the control plane and both worker nodes in a `READY` state.

![image.png](assets/image%2014.png)

Thank you for reading this article, if you think there should be anything to be added to this article or if I am missing information feel free to let me know.

## Troubleshooting

### ERROR: Container Runtime is not running

![image.png](assets/image%2015.png)

Kubernetes expects some default settings in the `containerd` Edit the configuration file if the CRI plugin isn’t enabled.

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
```

### ERROR: FileContent--proc-sys-net-bridge-bridge-nf-call-iptables

![image.png](assets/image%2015.png)

Run the below command.

```bash
sudo modprobe br_netfilter
```

### ERROR: The connection to the server <Control Plane IP Address>:6443 was refused

If you get the error `The connection to the server <Control Plane IP Address>:6443 was refused - did you specify the right host or port?` when running `kubectl get pods` follow the below steps to fix the issue.

Add `systemd.unified_cgroup_hierarchy=0` to your `GRUB_CMDLINE_LINUX_DEFAULT` in the grub file `/etc/default/grub for Debian`. 

```bash
sudo vi /etc/default/grub
```

![image.png](assets/image%2016.png)

Then run the below command.

```bash
sudo update-grub.
```

![image.png](assets/image%2017.png)

You can find detailed information about the problem in the article [Who's killing my pods](https://gjhenrique.com/cgroups-k8s/)?