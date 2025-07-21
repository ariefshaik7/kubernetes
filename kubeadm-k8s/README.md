
# Setting Up k8s Using Kubeadm

## **Chapter 1: The Blueprint - Understanding Our Setup**

#### **Our Architecture**

We will build a cluster using your three VMs with public IPs:

* **1 Control-Plane Node (Master):** `<master-IP>` (hostname: `k8s-master`)
    
* **2 Worker Nodes:**
    
    * `<worker1-IP>` (hostname: `k8s-worker1`)
        
    * `<worker2-IP>` (hostname: `k8s-worker2`)
        

#### **The Tools We'll Use**

1. `kubeadm`: The official Kubernetes tool for bootstrapping a cluster.
    
2. `kubelet`: The agent that runs on **every** node to manage containers.
    
3. `kubectl`: The command-line tool for interacting with your cluster from the master node.
    
4. `containerd`: The industry-standard container runtime that executes the containers.
    
5. **Calico**: Our chosen Container Network Interface (CNI) plugin for pod networking.
    
6. **Network Security** : we **must** configure to allow the nodes to communicate (Allowing Inbound Traffic).
    

---

### NOTE: Replace the IP tags with your respective IP addresses of VM’s.

---

## **Chapter 2: Prerequisites & Networking**

This section is the most critical for an error-free setup in a cloud environment.

#### **VM Specifications**

Ensure your three VMs meet these minimums:

* **OS:** Ubuntu 24.04 LTS
    
* **Size:** 2 vCPUs, 4GB RAM or larger is recommended.
    

#### **Networking**

Your VMs cannot communicate with each other until you create inbound security rules in their respective Security Groups.

For the k8s-master VM ( <master-IP> ):

Go to your k8s-master VM in the respective cloud provider-&gt; Networking -&gt; Add inbound port rule. Create the following rules:

| Priority | Name | Port | Protocol | Source | Destination | Action |
| --- | --- | --- | --- | --- | --- | --- |
| 100 | SSH | 22 | TCP | Your IP | Any | Allow |
| 110 | Kube-API | 6443 | TCP | `<worker1-IP>`, `<worker2-IP>`,`<master-IP>` | Any | Allow |
| 120 | Calico-BGP | 179 | TCP | `<worker1-IP>`, `<worker2-IP>` | Any | Allow |
| 130 | Calico-IPIP | All | IPIP | `<worker1-IP>`, `<worker2-IP>` | Any | Allow |

> **Why these rules?**
> 
> * **SSH (Port 22):** Allows you to connect to your VM for management. Restricting the source to "Your IP" is more secure.
>     
> * **Kube-API (Port 6443):** This is the **most important rule**. It allows the worker nodes to contact the Kubernetes API server on the master. The source is locked down to your workers' specific IPs.
>     
> * **Calico Rules (Port 179/IPIP):** Calico uses BGP and IPIP encapsulation for pod networking. This allows the nodes to route traffic for pods between each other. We must allow this communication from the worker IPs. *Note: In the networking settings, for the IPIP rule, you might need to select "Any" for protocol and manually add a description.*
>     

For BOTH Worker VMs (k8s-worker1 & k8s-worker2):

For each worker VM, add the following inbound rules to their NSGs.

| Priority | Name | Port | Protocol | Source | Destination | Action |
| --- | --- | --- | --- | --- | --- | --- |
| 100 | SSH | 22 | TCP | Your IP | Any | Allow |
| 110 | NodePort-Range | 30000-32767 | TCP | Any | Any | Allow |
| 120 | Calico-BGP | 179 | TCP | `<master-IP>`, *Other Worker IP* | Any | Allow |
| 130 | Calico-IPIP | All | IPIP | `<master-IP>`, *Other Worker IP* | Any | Allow |

> **Why these rules?**
> 
> * **NodePort-Range (Ports 30000-32767):** This allows you to access applications running in the cluster from the internet when using a `NodePort` service.
>     
> * **Calico Rules:** Similar to the master, each worker must be able to communicate with the master and the *other* worker node for pod networking. When configuring the rule for `k8s-worker1`, the source IPs should be `<master-IP>` and `<worker2-IP>`. For `k8s-worker2`, the source IPs should be `<master-IP>` and `<worker1-IP>`.
>     

---

## **Chapter 3: Node Preparation (Common Steps)**

The following steps must be performed on **ALL THREE** nodes: `k8s-master`, `k8s-worker1`, and `k8s-worker2`.

#### **Step 1: Set Hostnames & DNS Resolution**

**Action: On ALL Three Nodes**

1. **Set Hostnames:**
    
    * On `<master-IP>`: `sudo hostnamectl set-hostname k8s-master`
        
    * On `<worker1-IP>`: `sudo hostnamectl set-hostname k8s-worker1`
        
    * On `<worker2-IP>`: `sudo hostnamectl set-hostname k8s-worker2`
        
    
    > **Why?** Unique hostnames make identifying nodes in your cluster much easier.
    
2. **Update** `/etc/hosts` file:
    
    Bash
    
    ```bash
    sudo tee -a /etc/hosts > /dev/null <<EOF
    <master-IP>   k8s-master
    <worker1-IP>  k8s-worker1
    <worker2-IP>  k8s-worker2
    EOF
    ```
    
    > **Why?** This provides local DNS, allowing nodes to find each other by hostname (e.g., `k8s-master`), which is required by `kubeadm`.
    

#### **Step 2: Disable Swap**

Bash

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

* **What & Why:** This disables swap memory, both now and on future reboots. The Kubernetes `kubelet` manages memory directly and requires swap to be off to function correctly.
    

#### **Step 3: Enable Kernel Modules & Configure Sysctl**

Bash

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

* **What & Why:** This loads two kernel modules, `overlay` (for container storage) and `br_netfilter` (for network bridging). Both are required for `containerd` and Kubernetes networking.
    

Bash

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

* **What & Why:** This configures kernel parameters to allow Linux `iptables` to correctly see bridged traffic and enables IP forwarding. This is a mandatory prerequisite for the pod network to function.
    

#### **Step 4: Install** `containerd` Runtime

Bash

```bash
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

* **What & Why:** This installs `containerd`, creates a default configuration, and then modifies it to use the `SystemdCgroup` driver. This is **critical** because `kubelet` also uses this driver, and they must match. Finally, it restarts and enables the `containerd` service.
    

#### **Step 5: Install Kubernetes Packages (**`kubeadm`, `kubelet`, `kubectl`)

Bash

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

* **What & Why:** This sequence securely adds the official Kubernetes package repository to your system, installs the three key tools (`kubeadm`, `kubelet`, `kubectl`), and then "holds" their versions with `apt-mark hold`. Holding prevents accidental upgrades that could break the cluster, as Kubernetes versions must be managed carefully.
    

---

## **Chapter 4: The Control Plane**

These steps are performed **ONLY** on your master node (`k8s-master`, `<master-IP>`).

#### **Step 1: Initialize the Cluster with** `kubeadm`

**Action: On the** `k8s-master` Node ONLY

Bash

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=k8s-master
```

* **What it does:** This is the main command to bootstrap the control plane. It runs preflight checks, generates security certificates, and starts the core Kubernetes components.
    
* **Flags Explained:**
    
    * `--pod-network-cidr=10.244.0.0/16`: This sets the internal IP range for your pods. We use this specific range because it's what our CNI, Calico, expects by default.
        
    * `--control-plane-endpoint=k8s-master`: This provides a stable endpoint (our hostname) for the workers to find the API server.
        

**Important! After this command finishes, it will print two things. Copy them to a safe place.**

1. A block of commands to configure `kubectl`.
    
2. A `kubeadm join` command with a token. This is how your workers will join the cluster.
    

#### **Step 2: Configure** `kubectl` Access

**Action: On the** `k8s-master` Node ONLY

Bash

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

* **What & Why:** This copies the cluster's admin configuration file into your user's home directory. The `kubectl` command automatically looks for this file to get the credentials and address needed to communicate with the cluster.
    

---

## **Chapter 5: Weaving the Network**

The control plane is up, but pod networking isn't active yet. We need to install our CNI.

**Action: On the** `k8s-master` Node ONLY

Bash

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

* **What & Why:** This command uses `kubectl` to download and apply the Calico manifest. This YAML file defines all the necessary components (Pods, Services, etc.) for Calico. Kubernetes then creates these components, which will manage your pod network, allowing pods on different nodes to communicate.
    

---

## **Chapter 6: Joining the Workers**

Now, let's connect the worker nodes to our master.

**Action: On BOTH** `k8s-worker1` and `k8s-worker2` Nodes

Use the `kubeadm join` command that you saved earlier. It will look similar to this, but with your unique token and hash:

Bash

```bash
kubeadm join k8s-master:6443 --token 4tw5tj.2dyk9jf4ohn70mnc \
    --discovery-token-ca-cert-hash sha256:d7023a2d4467d026449c2cc702cf2670e42ac6d60fc8372a1fbdd15a5640b72c
```

* **What & Why:** This single command tells the worker node to contact the master's API server (`k8s-master:6443`), authenticate using the secure token, and validate the master's identity with the certificate hash. It then configures the local `kubelet` to officially join the cluster.
    

> **Lost the join command?** Don't worry. Just run this on the `k8s-master` node to generate a new one: `kubeadm token create --print-join-command`

---

## **Chapter 7: Verification and Testing**

Let's confirm the cluster is fully operational from the master node.

**Action: On the** `k8s-master` Node ONLY

#### **Step 1: Check Node Status**

Bash

```bash
kubectl get nodes -o wide
```

It may take a minute for the STATUS to become `Ready`. You should see an output like this:

```bash
NAME          STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP     OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-master    Ready    control-plane   15m   v1.30.x   10.0.0.4      <master-IP>     Ubuntu 22.04.4 LTS   6.5.0-1021-*    containerd://1.6.31
k8s-worker1   Ready    <none>          5m    v1.30.x   10.0.0.5      <worker1-IP>    Ubuntu 22.04.4 LTS   6.5.0-1021-*    containerd://1.6.31
k8s-worker2   Ready    <none>          5m    v1.30.x   10.0.0.6      <worker2-IP>   Ubuntu 22.04.4 LTS   6.5.0-1021-*    containerd://1.6.31
```

#### **Step 2: Deploy and Expose a Test Application**

1. Create an NGINX Deployment:
    
    kubectl create deployment nginx-test --image=nginx
    
2. Expose the Deployment with a NodePort:
    
    kubectl expose deployment nginx-test --type=NodePort --port=80
    
3. Find the NodePort:
    
    kubectl get service nginx-test
    
    The output will show a port mapping like 80:3XXXX/TCP. Note the 3XXXX port number.
    
4. Access the NGINX Server:
    
    Open a web browser or use curl from any machine to access your NGINX server. Use the public IP of either worker node and the NodePort.
    
    Bash
    
    ```bash
    # Example using worker1's public IP and an example port of 31234
    curl http://<worker1-IP>:31234
    ```
    

If you get the "Welcome to nginx!" response, you have successfully built and configured a K8s using kubeadm!