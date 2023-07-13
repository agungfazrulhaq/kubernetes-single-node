## Kubernetes Single-node Installation Guide

This guide provides detailed steps to install Kubernetes on your system. Please follow the instructions below to set up Kubernetes successfully.

### Prerequisites

Before installing Kubernetes, ensure that you have the following prerequisites:

- Ubuntu operating system (Ubuntu 22.04*)
- Access to the internet
- Administrative privileges (sudo)

### Installation Steps

#### 1. Install Docker

Run the following commands to install Docker:

```shell
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg make
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Explanation:
- The first two commands update the package lists and install the necessary tools for the installation process.
- The next command creates a directory for Docker's GPG key.
- The `curl` command downloads the Docker GPG key and saves it in the keyrings directory.
- The `chmod` command changes the permissions of the GPG key file.
- The `echo` command adds the Docker repository to the system's package sources.
- The `apt-get update` command updates the package lists to include the Docker repository.
- Finally, the `apt-get install` command installs Docker and its dependencies.

#### 2. Install Container Runtime (CRI-DOCKERD)

Follow these steps to install the CRI-DOCKERD container runtime:

```shell
git clone https://github.com/Mirantis/cri-dockerd.git

# Install Go
wget https://go.dev/dl/go1.20.6.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.20.6.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
go version

cd cri-dockerd
sudo apt install make
make cri-dockerd
sudo mkdir -p /usr/local/bin
sudo install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
sudo install packaging/systemd/* /etc/systemd/system
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket

# Disable swap
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a

sudo modprobe overlay
sudo modprobe br_netfilter
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

Explanation:
- The `git clone` command clones the CRI-DOCKERD repository from GitHub.
- The following commands download and install Go, a programming language required for building CRI-DOCKERD.
- The `cd` command changes the directory to the cri-dockerd folder.
- The `make cri-dockerd` command builds the cri-dockerd binary.
- The next set of commands install cri-dockerd and configure it as a systemd service.
- The `systemctl` commands enable and start the cri-dockerd service.
- The swap is disabled to ensure Kubernetes runs optimally.
- The `modprobe` commands load required kernel modules.
- The `tee` command creates a Kubernetes configuration file to set necessary sysctl parameters.

#### 3. Install Kubernetes

Execute the following commands to install Kubernetes:

```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update -y
sudo apt -y install vim git curl wget kubelet=1.25.0-00 kubeadm=1.25.0-00 kubectl=1.25.0-00
sudo apt-mark hold kubelet kubeadm kubectl
sudo kubeadm config images pull --cri-socket unix:///var/run/cri-dockerd.sock --kubernetes-version v1.25.0
export KUBECONFIG=/etc/kubernetes/admin.conf
sudo chmod 644 /etc/kubernetes/admin.conf
kubectl cluster-info
kubectl taint nodes master node-role.kubernetes.io/control-plane-
apt-get install bash-completion
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
```

Explanation:
- The `curl` command fetches the Kubernetes repository's GPG key and adds it to the system's keyring.
- The `echo` command appends the Kubernetes repository to the system's package sources.
- The `apt update` command updates the package lists to include the Kubernetes repository.
- The subsequent `apt` command installs the necessary Kubernetes components, including kubelet, kubeadm, and kubectl.
- The `apt-mark hold` command prevents these packages from being automatically updated.
- The `kubeadm config images pull` command pulls the necessary container images required for Kubernetes.
- The `export` command sets the KUBECONFIG environment variable to the Kubernetes configuration file location.
- The `chmod` command updates the permissions of the Kubernetes configuration file.
- The `kubectl cluster-info` command verifies the Kubernetes cluster's information.
- The `kubectl taint` command removes a taint from the master node to allow scheduling regular pods.
- The final commands install bash completion for kubectl, enabling convenient command line autocompletion.

#### 4. Install CNI (Calico)

To install Calico, a CNI (Container Network Interface), perform the following steps:

```shell
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml -O
nano custom-resources.yaml
# Edit the "cidr" value in the "calicoNetwork" section to match your subnet network for pods

kubectl apply -f custom-resources.yaml
```

Explanation:
- The `kubectl create` command deploys the Tigera operator for Calico.
- The `curl` command downloads the custom resources YAML file required for configuring Calico.
- Use a text editor, such as `nano`, to edit the downloaded YAML file and set the desired CIDR value for the Calico network.
- The `kubectl apply` command applies the custom resources

 configuration to install and configure Calico.

#### 5. Install NFS Provisioning

To set up NFS Provisioning, follow the steps below:

```shell
sudo apt update
sudo apt install nfs-kernel-server nfs-common
sudo mkdir -p /nfskubernetes
ls -dl /nfskubernetes
sudo chown nobody:nogroup /nfskubernetes
sudo nano /etc/exports
# Add the following line to /etc/exports:
# /nfskubernetes 192.168.1.159(rw,sync,no_subtree_check,no_root_squash)

sudo systemctl restart nfs-kernel-server
sudo ufw allow from 192.168.1.159 to any port nfs

curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=x.x.x.x \
    --set nfs.path=/exported/path

kubectl get sc
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Explanation:
- The first commands update the package lists and install the necessary packages for NFS server and client.
- The `mkdir` command creates a directory for the NFS share.
- The subsequent commands configure the NFS server by adjusting permissions and adding the NFS share to the exports file.
- The `systemctl` command restarts the NFS server.
- The firewall (UFW) is configured to allow incoming NFS traffic.
- The next set of commands installs Helm, a package manager for Kubernetes.
- The `helm repo add` command adds the NFS Subdir External Provisioner Helm repository.
- The `helm install` command deploys the NFS Subdir External Provisioner using the provided NFS server and path.
- The `kubectl` commands verify the installed storage class and patch the NFS storage class to be the default.

Congratulations! You have successfully installed Kubernetes on your system.

For more detailed information and further configurations, you can refer to the following references:

- [Kubernetes Documentation](https://kubernetes.io/docs/setup/)
- [CRI-DOCKERD GitHub Repository](https://github.com/Mirantis/cri-dockerd)
- [Go Installation Guide](https://go.dev/doc/install#)
- [Docker Installation Guide](https://docs.docker.com/engine/install/ubuntu/)
- [Calico Getting Started Guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)
- [NFS Subdir External Provisioner GitHub Repository](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [Helm Installation Guide](https://helm.sh/docs/intro/install/)
- [Calico Self-Managed On-Premises Guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)

Feel free to explore these resources for more information and configuration options.
