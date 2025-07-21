# Accessing k8s locally

To access the k8s using locally , `~/.kube/config` file is used. By default, it's named `config` and lives in a directory called `.kube` in your user's home directory (`~/.kube/config`).

This YAML file acts like a phonebook for `kubectl`. It contains all the information needed to connect to and authenticate with one or more Kubernetes clusters, including:

* **Cluster Address:** The API server's URL (e.g., [`https://20.244.3.31:6443`](https://20.244.3.31:6443)).
    
* **User Credentials:** The security certificate and key to prove who you are.
    
* **Context:** A nickname that ties a user to a cluster, allowing you to easily switch between them.
    

The goal is to securely copy this file from your master VM to your local computer.

---

### Step-by-Step Guide to Local Access

Follow these four steps on your local computer (not in the SSH session).

## Step 1: Install `kubectl` on Your Local Computer

If you don't already have it, you need to install the `kubectl` tool.

* **Windows:** Open PowerShell and run `winget install -e --id Kubernetes.kubectl`.
    
* **macOS:** Open Terminal and run `brew install kubectl`.
    
* **Linux (Ubuntu/Debian):** Open a terminal and run `sudo apt-get update && sudo apt-get install -y kubectl`.
    

---

## Step 2: Securely Copy the `kubeconfig` File

Now, you'll copy the admin configuration file from your master VM to your local machine.

1. Open a new terminal or PowerShell window on your **local computer**.
    
2. Run the following `scp` (Secure Copy) command.
    

Make the File Readable (on the Master VM):

SSH into your master VM and run this command. It uses `sudo` to copy the file to your home directory and changes its ownership to your current user.

```bash
# Run this command on your k8s-master VM via SSH
sudo cp /etc/kubernetes/admin.conf ~/admin.conf && sudo chown $(id -u):$(id -g) ~/admin.conf
```

Copy the File (from your Local PC):

Now, from your **local Ubuntu machine**, run the `scp` command again, but point it to the new, accessible file located in the home directory (`~/admin.conf`).

```bash
# Run this command on your local PC
scp -i k8s_vm_key.pem k8s-master@20.244.3.31:~/admin.conf ~/.kube/config
```

This will successfully copy the file. For security, you can log back into the master VM and delete the temporary copy with `rm ~/admin.conf`.

This will overwrite any existing `~/.kube/config` file. If you manage other clusters (like Docker Desktop), see the "Managing Multiple Clusters" section below.

---

## Step 3: Update the Master Node's Firewall (Crucial)

Your local computer now knows *how* to talk to the cluster, but the vm firewall will block it. You need to tell the master node's Networking to allow connections from your local computer's IP address.

1. **Find Your Public IP:** On your local computer, run the following command
    

```bash
 curl ipinfo.io/ip
```

1. **Go to Cloud Networking Portal:** Navigate to the Networking tab for your master VM (`k8s-master-nsg`).
    
2. **Edit the** `Kube-API` rule: Click on the inbound rule for port `6443` (the one with Priority 110).
    
3. **Add Your IP to the Source:** In the "Source IP addresses/CIDR ranges" field, add your local public IP to the list. The field should now contain the IPs for your workers, the master itself, and your new local IP.
    
4. **Save** the rule.
    

### Why the IPs with the `ip addr` command Won't Work ?

The IP addresses like (`192.168.x.x`, `172.17.x.x`, etc.) are all **private IPs**. They are only used for communication *inside* your local network (your home or office).

When you connect to your VM over the internet, your request goes through your internet router, which swaps your private IP for your network's single **public IP address**. The firewall only sees this public IP.

Alternatively, you can open a web browser and search for "**what is my IP**".

---

## Step 4: Adding VM's **public IP** to /etc/hosts

Your local machine doesn't know the IP address for the hostname `k8s-master`.

Your `kubeconfig` file tells `kubectl` to connect to the server at the address [`https://k8s-master:6443`](https://k8s-master:6443). While you configured the hostname `k8s-master` in the `/etc/hosts` files on your VMs, you haven't done the same on your local computer. As a result, your local machine's DNS resolver fails to find where `k8s-master` is located.

### Edit Your Local `hosts` File

You need to manually tell your local machine what IP address `k8s-master` corresponds to.

1. Open the `/etc/hosts` file on your **local Ubuntu machine** with root privileges.
    
    Bash
    
    ```bash
    sudo nano /etc/hosts
    ```
    
2. Add the following line to the bottom of the file. This maps the master VM's **public IP** to its hostname.
    
    ```bash
    <master-IP>   k8s-master
    ```
    
3. Save the file and exit by pressing `Ctrl+X`, then `Y`, and then `Enter`.
    

No restart is needed. The change takes effect immediately.

---

## Step 5: Verify the Connection

You're all set! To confirm it's working, run this command on your **local computer's terminal**:

Bash

```bash
kubectl get nodes -o wide
```

If it successfully returns the list of your three cluster nodes, you are now managing your K8s cluster directly from your local machine.

---

## Managing Multiple Clusters

If you work with more than one cluster (e.g., one on AWs, one on Azure, one from Docker Desktop), simply overwriting the `~/.kube/config` file is not ideal. `kubectl` can manage multiple connection profiles, called "contexts."

You can see all your configured contexts by running:

kubectl config get-contexts

The one with an asterisk (`*`) is the one you are currently connected to.

To switch between your different clusters, use the `use-context` command:

Bash

```bash
# View All Contexts
kubectl config get-contexts

# Get current context
kubectl config current-context

# Switch to your new cluster
kubectl config use-context kubernetes-admin@kubernetes

# Switch back to a Docker Desktop cluster (example name)
kubectl config use-context docker-desktop
```

This allows you to seamlessly manage any number of clusters from a single, convenient location: your own computer.

---