# Homelab: Building high availability 3 node kubernetes cluster using old mini pc

Every homelab enthusiast reaches a point where the simple stuff just isn't enough. Docker containers are great, but you start to crave something more—something that mirrors the real-world, resilient infrastructure that powers the cloud. That's where I was. Standing in my lab, looking at three perfectly capable mini PCs, I had a vision: a fully resilient, High-Availability (HA) Kubernetes cluster, built from the ground up on bare metal for maximum performance.

I thought it would be a straightforward weekend project. I was wrong.

What followed was a week-long odyssey into the deepest, most frustrating corners of network configuration, kernel settings, and obscure incompatibilities. It was a battle, but I came out the other side with a working cluster and a battle-tested blueprint.

This is the story of that journey. It’s a guide not just for how to build a production-ready Kubernetes cluster at home, but also for how to debug it when everything inevitably goes wrong. If you're ready to build something truly powerful, this is the map of the minefield I navigated so you don't have to.

### **The Foundation: Hardware and Network**

A stable cluster needs a stable foundation.

- **The Hardware:** My setup consists of three mini PCs, giving me a good balance of performance and power efficiency. The specific configurations are:
    - **Node 1:**
        - **CPU:** Intel Core i5-10500T @ 2.30GHz
        - **RAM:** 32GB
        - **Storage:** 512GB SSD
    - **Node 2:**
        - **CPU:** 13th Gen Intel Core i5-13500T
        - **RAM:** 32GB
        - **Storage:** 256GB SSD
    - **Node 3:**
        - **CPU:** 13th Gen Intel Core i5-13500T
        - **RAM:** 32GB
        - **Storage:** 256GB SSD
- **The Network:** The most critical decision was to **not** use my ISP-provided router for server-to-server traffic. I connected all three mini PCs to a dedicated **8-Port Gigabit Switch**, which is essential for the performance and reliability of the cluster.

### **Building the Kubernetes Cluster on Ubuntu**

With the hardware sorted, I installed **Ubuntu Server 24.04 LTS** as the base operating system on each machine. Then, I used the standard `kubeadm` toolset to build the cluster.

### **Part 1: Prepare Every Node**

First, I SSH'd into all three nodes and ran the same preparation script to get them ready for Kubernetes. Using a script ensures consistency.

*I saved this as `prepare-node.sh` and ran it everywhere with `sudo`.*

```
#!/bin/bash
# Exit immediately if a command exits with a non-zero status.
set -e

echo "--- Starting Node Preparation ---"

# 1. DISABLE SWAP
echo "--- Disabling swap ---"
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 2. ENABLE KERNEL MODULES
echo "--- Enabling kernel modules ---"
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# 3. CONFIGURE SYSCTL KERNEL PARAMETERS
echo "--- Configuring sysctl kernel parameters ---"
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# 4. INSTALL AND CONFIGURE CONTAINERD
echo "--- Installing and configuring containerd ---"
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd

# 5. INSTALL KUBERNETES TOOLS (kubeadm, kubelet, kubectl)
echo "--- Installing Kubernetes tools ---"
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

echo "--- Node preparation complete! ---"

```

### **Part 2: Initialize the Cluster**

This part is run **only on the first node**. This command creates the control plane.

```
# Using the wired IP of my first node
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --control-plane-endpoint=<your-first-node-ip>

```

After it finished, I ran the commands it printed to give my user `kubectl` access:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

### **Part 3: Install the Network (CNI)**

From my first node, I installed the Calico CNI.

```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml

```

### **Part 4: Join the Other Nodes**

To join the other nodes as control-plane members, I generated a new join command from the first node:

```
# Create a new token
sudo kubeadm token create

# Get the CA hash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

# Get the certificate key
sudo kubeadm init phase upload-certs --upload-certs

```

Then, on the other nodes, I ran the full `kubeadm join` command assembled from that output.

### **Part 5: Make All Nodes Usable**

To complete the hyper-converged setup, I ran this from my first node to allow applications to run on all three nodes:

```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

```

The result was three `Ready` nodes, viewable with `kubectl get nodes`.

### **Exposing Argo CD to the World**

With a stable cluster, I could now expose my first application, Argo CD, via a public, secure domain.

### **Step 1: Install Argo CD**

First, I installed the application itself.

```
# Create a dedicated namespace for Argo CD
kubectl create namespace argocd

# Apply the official installation manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

```

### **Step 2: The Host-Level Network Fix**

For MetalLB to work on my bare-metal setup, I had to adjust a kernel setting on **all three nodes**. I edited `/etc/sysctl.conf` and added these lines:

```
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2

```

After running `sudo sysctl -p`, the MetalLB IP became reachable.

### **Step 3: Install MetalLB and NGINX Ingress**

First, I installed **MetalLB** to provide a `LoadBalancer` service type.

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.7/config/manifests/metallb-native.yaml

```

Next, I created a file named `metallb-config.yaml` to tell MetalLB which IPs from my home network it could use.

```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: primary-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: main-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - primary-pool

```

I applied this configuration with `kubectl apply -f metallb-config.yaml`.

Then, I installed the **NGINX Ingress Controller**, which is the cluster's front door.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml

```

After a minute, I ran `kubectl get svc -n ingress-nginx` and saw that the controller had been assigned the external IP `192.168.1.200` from MetalLB. This is the IP I set as the DMZ target in my router.

### **Step 4: Install Cert-Manager and Configure Ingress**

To get automatic HTTPS, I installed **Cert-Manager** and created a `ClusterIssuer` to use Let's Encrypt. The final step was to create the `Ingress` resource file (`argocd-ingress.yaml`) that ties everything together.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    # These annotations configure NGINX to handle TLS and talk to the ArgoCD backend
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "false"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.homelab.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
  tls:
  - hosts:
    - argocd.homelab.example.com
    secretName: argocd-server-tls

```

After applying this file (`kubectl apply -f argocd-ingress.yaml`), the certificate was issued, and the site `https://argocd.homelab.example.com` went live.

### **Conclusion**

Building a Kubernetes cluster from scratch is a significant undertaking, but following a standard, well-supported path (Ubuntu on bare metal with `kubeadm`) makes it an achievable and rewarding project. The foundation is everything—with a stable network and a compatible OS, the complex world of cloud-native applications becomes accessible. My home lab is now officially open for business.
