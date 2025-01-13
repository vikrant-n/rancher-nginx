# DevOps_Rancher

# Rancher Kubernetes Setup

This guide provides step-by-step instructions to set up and configure Rancher on an RKE2 cluster, including installing necessary tools like `kubectl` and `Helm`.

---
![rancher-rke2](https://github.com/user-attachments/assets/9884574d-0a8a-4ebe-aca0-bf830bad9720)

---

## Prerequisites  

### Infrastructure:  
- **For Each Node**:  
  - Minimum RAM: `4GB`  
  - Minimum CPU: `2 core`  

### Inbound Network Rules:
| Port Range      | Protocol | Source              | Destination         | Description                     |
|------------------|----------|---------------------|---------------------|---------------------------------|
| 8472            | UDP      | All RKE2 nodes      | All RKE2 nodes      | Canal CNI with VXLAN           |
| 9099            | TCP      | All RKE2 nodes      | All RKE2 nodes      | Canal CNI health checks        |
| 6443            | TCP      | RKE2 agent nodes    | RKE2 server nodes   | Kubernetes API                 |
| 9345            | TCP      | RKE2 agent nodes    | RKE2 server nodes   | RKE2 Supervisor API            |
| 10250           | TCP      | All RKE2 nodes      | All RKE2 nodes      | Kubelet metrics                |
| 2379-2381       | TCP      | RKE2 server nodes   | RKE2 server nodes   | etcd client, peer, and metrics |
| 30000-32767     | TCP      | All RKE2 nodes      | All RKE2 nodes      | NodePort range                 |

### CNI Specific Inbound Network Rules (Canal):
| Port  | Protocol | Source         | Destination    | Description                      |
|-------|----------|----------------|----------------|----------------------------------|
| 8472  | UDP      | All RKE2 nodes | All RKE2 nodes | Canal CNI with VXLAN            |
| 9099  | TCP      | All RKE2 nodes | All RKE2 nodes | Canal CNI health checks         |
| 51820 | UDP      | All RKE2 nodes | All RKE2 nodes | Canal CNI with WireGuard IPv4   |
| 51821 | UDP      | All RKE2 nodes | All RKE2 nodes | Canal CNI with WireGuard IPv6/dual-stack |

### Enable SELinux:
Install `container-selinux` using Amazon Linux Extras on all nodes:  
```bash
sudo amazon-linux-extras enable selinux-ng
sudo yum install -y container-selinux
```

---

## RKE2 Installation

### On Master Nodes:  
1. **Run the Installer:**  
   ```bash
   curl -sfL https://get.rke2.io | sh -
   ```
2. **Enable and Start the RKE2 Server:**  
   ```bash
   systemctl enable rke2-server.service
   systemctl start rke2-server.service
   ```
3. **Retrieve the Cluster Token:**  
   ```bash
   cat /var/lib/rancher/rke2/server/node-token
   ```

### On Worker Nodes:  
1. **Run the Installer:**  
   ```bash
   curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
   ```
2. **Enable and Configure the RKE2 Agent:**  
   ```bash
   systemctl enable rke2-agent.service

   mkdir -p /etc/rancher/rke2/
   vim /etc/rancher/rke2/config.yaml
   ```
   Example `config.yaml`:  
   ```yaml
   server: https://<master-node-ip>:9345
   token: <cluster-token>
   ```
3. **Start the RKE2 Agent:**  
   ```bash
   systemctl start rke2-agent.service
   ```

---

## Install and Configure kubectl

1. **Download and Install kubectl:**  
   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl
   sudo mv kubectl /usr/local/bin/
   ```
2. **Verify Installation:**  
   ```bash
   kubectl version --client
   ```
3. **Configure Kubeconfig File:**  
   - Retrieve kubeconfig from master node:  
     ```bash
     mkdir -p ~/.kube
     cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
     chmod 600 ~/.kube/config
     ```
   - Replace `127.0.0.1` with the master node's IP:  
     ```yaml
     server: https://<master-node-ip>:6443
     ```
4. **Verify Connectivity:**  
   ```bash
   kubectl get nodes
   ```

---

## Install Helm

1. **Install Helm:**  
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```
2. **Verify Installation:**  
   ```bash
   helm version
   ```

---

## Deploy Rancher on RKE2

1. **Add the Rancher Helm Chart:**  
   ```bash
   helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
   helm repo update
   ```
2. **Install Cert-Manager:**  
   ```bash
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
   ```
3. **Install Rancher:**  
   ```bash
   helm install rancher rancher-latest/rancher    --namespace cattle-system --create-namespace    --set hostname=<your-domain-or-ip>    --set replicas=1
   ```
4. **Monitor Deployment:**  
   ```bash
   kubectl -n cattle-system rollout status deploy/rancher
   ```
5. **Access Rancher UI:**  
   - Navigate to `https://<your-domain>` or `https://<master-node-ip>` in a browser.  

---

## Optional: Configure SSL with Let's Encrypt
```bash
helm upgrade --install rancher rancher-latest/rancher   --namespace cattle-system --create-namespace   --set hostname=<your-domain>   --set ingress.tls.source=letsEncrypt   --set letsEncrypt.email=<your-email>   --set replicas=1 --set bootstrapPassword=Changeme123!
```

Retrieve the bootstrap password:  
```bash
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```

---

## Troubleshooting:
- Ensure each node has a unique hostname. For duplicate hostnames, set the `node-name` parameter in the `config.yaml` file.  
- Monitor logs for troubleshooting:  
  ```bash
  journalctl -u rke2-server -f
  journalctl -u rke2-agent -f
  ```
- If the bootstrap password doesn't work, you can reset it using the following command:  
  ```bash
  kubectl -n cattle-system exec $(kubectl -n cattle-system get pods -l app=rancher | grep '1/1' | head -1 | awk '{ print $1 }') -- reset-password
  ```
---

## Conclusion  
This setup provides a functional Rancher cluster for Kubernetes management. In the next steps, we can integrate tools like **NGINX Plus Ingress Controller** and **NGINX App Protect** to further enhance this deployment.  

---

## References
- [RKE2 Documentation](https://docs.rke2.io/)
- [Rancher Manager Documentation](https://ranchermanager.docs.rancher.com/)
