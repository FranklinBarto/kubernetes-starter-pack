# Kubernetes Installation and Configuration Guide

This guide provides comprehensive instructions for installing, uninstalling, and configuring Kubernetes with containerd on Ubuntu systems.

## Table of Contents
- [Uninstallation of Kubernetes and Containerd](#uninstallation-of-kubernetes-and-containerd)
- [Installation of Containerd](#installation-of-containerd)
- [Installation of Kubernetes](#installation-of-kubernetes)
- [Installation of Flannel](#installation-of-flannel)
- [Inttallations of Metallb](#installation-of-Metallb)
- [Installation of nginx-ingress](#installation-of-nginx-ingress)

## Uninstallation of Kubernetes and Containerd

### Step 1: Reset the Kubernetes Cluster
Before removing Kubernetes, reset the cluster using kubeadm:
```bash
sudo kubeadm reset
```
This removes all resources associated with the Kubernetes cluster.

### Step 2: Remove Kubernetes Packages
```bash
sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube*
sudo apt-get autoremove
```

### Step 3: Remove CNI Network Plugins
```bash
sudo rm -rf /etc/cni/net.d
sudo rm -rf /opt/cni/bin
```

### Step 4: Remove Kubernetes Configuration Files
```bash
sudo rm -rf ~/.kube
sudo rm -rf /etc/kubernetes/
sudo rm -rf /var/lib/etcd
sudo rm -rf /var/lib/kubelet
sudo rm -rf /var/lib/kubeadm.yaml
```

### Step 5: Remove Container Runtime (optional)
For Docker:
```bash
sudo apt-get purge docker-ce docker-ce-cli containerd.io
sudo apt-get autoremove
sudo rm -rf /var/lib/docker
sudo rm -rf /etc/docker
```

For containerd:
```bash
sudo apt-get purge containerd
sudo rm -rf /var/lib/containerd
```

### Step 6: Clean Up iptables Rules
```bash
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t mangle -F
sudo iptables -X
```

### Step 7: Reboot the System
```bash
sudo reboot
```

## Installation of Containerd

### Step 1: Update Your System
```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2: Install Dependencies
```bash
sudo apt install -y ca-certificates curl gnupg lsb-release
```

### Step 3: Add Docker's Official GPG Key
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### Step 4: Set Up the Docker Repository
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

### Step 5: Install containerd
```bash
sudo apt update
sudo apt-get install -y containerd.io
```

### Step 6: Configure containerd
1. Generate default configuration:
```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

2. Verify system initialization:
```bash
ps 1
# or
dpkg -S /sbin/init
```

3. Configure systemd cgroup driver:
Edit `/etc/containerd/config.toml` and set:
```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

4. Restart containerd:
```bash
sudo systemctl restart containerd
```

### Step 7: Enable and Start containerd
```bash
sudo systemctl enable containerd
sudo systemctl start containerd
```

### Step 8: Verify Installation
```bash
sudo systemctl status containerd
containerd --version
```

## Installation of Kubernetes

### Step 1: Set Up Kubernetes with kubeadm

1. Install required packages:
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Step 2: Disable Swap
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Step 3: Load containerd Modules
Create `/etc/modules-load.d/containerd.conf`:
```
overlay
br_netfilter
```

### Step 4: Load Modules
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

### Step 5: Configure Kubernetes Networking
Create `/etc/sysctl.d/kubernetes.conf`:
```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

### Step 6: Apply System Configuration
```bash
sudo sysctl --system
```

### Step 7: Configure Kubernetes with systemd
1. Generate kubeadm configuration:
```bash
kubeadm config print init-defaults > kubeadm-config.yaml
```

2. Edit `kubeadm-config.yaml`:
```yaml
nodeRegistration:
  criSocket: /run/containerd/containerd.sock
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

3. Initialize cluster:
```bash
sudo kubeadm init --config kubeadm-config.yaml
```

### Step 8: Verify Configuration
```bash
sudo cat /var/lib/kubelet/config.yaml | grep cgroupDriver
```

## Network Components Installation

### Installation of Flannel
Flannel is a simple and efficient CNI (Container Network Interface) provider for Kubernetes.
It is important to note that you need to ensure you used the correct podCIRD with the kubeadm init command as flannel works with the ip address - 10.244.0.0

1. Apply the Flannel manifest:
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
 optionally you can install Flannel with helm:
```bash
# Needs manual creation of namespace to avoid helm error
kubectl create ns kube-flannel
kubectl label --overwrite ns kube-flannel pod-security.kubernetes.io/enforce=privileged

helm repo add flannel https://flannel-io.github.io/flannel/
helm install flannel --set podCidr="10.244.0.0/16" --namespace kube-flannel flannel/flannel
```

1. Verify Flannel installation:
```bash
kubectl get pods -n kube-system | grep flannel
```

1. Check network interfaces:
```bash
ip a | grep flannel
```

### Installation of MetalLB
MetalLB provides a network load-balancer implementation for Kubernetes clusters that do not run on a cloud provider.

1. Create the MetalLB namespace:
```bash
kubectl create namespace metallb-system
```

2. Apply MetalLB manifests:
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```

3. Wait for MetalLB pods to be ready:
```bash
kubectl wait --namespace metallb-system \
                --for=condition=ready pod \
                --selector=app=metallb \
                --timeout=90s
```

4. Configure the IP address pool:
```yaml
# Create a file named metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250  # Adjust this range according to your network
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
```

5. Apply the configuration:
```bash
kubectl apply -f metallb-config.yaml
```

6. Verify MetalLB installation:
```bash
kubectl get pods -n metallb-system
```

### Installation of NGINX Ingress Controller
NGINX Ingress Controller provides an efficient way to route external HTTP and HTTPS traffic to your services.

1. Upgrade the CRDs,
```bash
    kubectl apply -f https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/v3.7.0/deploy/crds.yaml
```
2. Install Nginx Ingress  using helm:
```bash
helm install my-release oci://ghcr.io/nginxinc/charts/nginx-ingress --version 1.4.0
```

1. Verify the installation:
```bash
kubectl get pods -n ingress-nginx
kubectl get services -n ingress-nginx
```

1. Create a sample ingress resource:
```yaml
# Create a file named sample-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.local  # Replace with your domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example-service
            port:
              number: 80
```

1. Apply the ingress resource:
```bash
kubectl apply -f sample-ingress.yaml
```

### Verification and Testing

1. Test Flannel networking:
```bash
# Create test pods in different nodes
kubectl run test-pod-1 --image=nginx
kubectl run test-pod-2 --image=nginx

# Verify pod communication
kubectl exec test-pod-1 -- ping <test-pod-2-ip>
```

2. Test MetalLB:
```bash
# Create a test service
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Verify the external IP assignment
kubectl get services nginx
```

3. Test NGINX Ingress:
```bash
# Add your ingress host to /etc/hosts
echo "192.168.1.240 example.local" | sudo tee -a /etc/hosts

# Test using curl
curl -H "Host: example.local" http://192.168.1.240
```

### Troubleshooting

#### Flannel Issues
- Check Flannel pods status:
  ```bash
  kubectl logs -n kube-system <flannel-pod-name>
  ```
- Verify network configuration:
  ```bash
  kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'
  ```

#### MetalLB Issues
- Check MetalLB controller logs:
  ```bash
  kubectl logs -n metallb-system -l app=metallb -c controller
  ```
- Verify IP address pool configuration:
  ```bash
  kubectl get ipaddresspools -n metallb-system
  ```

#### NGINX Ingress Issues
- Check Ingress controller logs:
  ```bash
  kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
  ```
- Verify Ingress resource configuration:
  ```bash
  kubectl describe ingress example-ingress
  ```

### Additional Notes

- Ensure your network allows the required ports for each component
- MetalLB requires layer 2 network access between nodes
- NGINX Ingress controller requires a LoadBalancer service type (provided by MetalLB in bare metal setups)
- Adjust IP ranges according to your network configuration
- Consider security implications and implement necessary network policies

### Maintenance

Remember to regularly update these components:

```bash
# Update Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Update MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

# Update NGINX Ingress
helm upgrade ingress-nginx ingress-nginx/ingress-nginx
```

## Contributing
Feel free to contribute to this guide by submitting pull requests or creating issues for any improvements or corrections.