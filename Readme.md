# Consul Multi-Cluster Service Mesh with Failover

A demo project for setting up Consul service mesh across multiple Kubernetes clusters (AWS EKS + Linode) with automatic failover.

## Quick Start

### Prerequisites
- AWS account with credentials
- Linode account with a Kubernetes cluster
- `terraform`, `kubectl`, `helm`, and AWS CLI installed

### Step 1: Create AWS EKS Cluster

```bash
cd terraform

# Create terraform.tfvars with your credentials
cat > terraform.tfvars <<EOF
aws_access_key_id = "your-key"
aws_secret_access_key = "your-secret"
aws_region = "us-east-1"
EOF

# Deploy infrastructure
terraform init -upgrade
terraform apply -var-file terraform.tfvars
```

### Step 2: Connect to EKS Cluster

```bash
aws configure
aws eks update-kubeconfig --region us-east-1 --name myapp-eks-cluster
kubectl get svc  # verify connection
```

### Step 3: Deploy Microservices on EKS

```bash
cd ../kubernetes

# Deploy applications
kubectl apply -f config-consul.yaml

# Verify pods are running
kubectl get pod
```

### Step 4: Install Consul on EKS

```bash
# Add Helm repo
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Install Consul
helm install eks hashicorp/consul --version 1.0.0 \
  --values consul-values.yaml \
  --set global.datacenter=eks

# Verify everything is running
kubectl get pod

# Get Consul UI URL
kubectl get svc -l app=consul,component=ui
# Open the EXTERNAL-IP in browser (accept self-signed cert)
```

### Step 5: Inject Consul Sidecars

```bash
# Restart deployments to inject Consul sidecars
kubectl rollout restart deployment -l app!=consul

# Verify all pods are ready
kubectl get pod
```

### Step 6: Configure Linode Cluster

```bash
# Download kubeconfig from Linode dashboard
export KUBECONFIG=~/.kube/config:/path/to/linode/kubeconfig.yaml

# Switch to Linode context
kubectl config get-contexts
kubectl config use-context <linode-context>

# Install Consul on Linode
helm install lks hashicorp/consul --version 1.0.0 \
  --values consul-values.yaml \
  --set global.datacenter=lks

# Deploy microservices
kubectl apply -f config-consul.yaml

# Restart deployments for sidecars
kubectl rollout restart deployment -l app!=consul

# Get Consul UI URL
kubectl get svc -l app=consul,component=ui
```

### Step 7: Setup Consul Peering (Multi-Cluster)

1. **On EKS Consul UI** (AWS):
   - Navigate to **Peers**
   - Click **Add peer connection**
   - Click **Generate token**
   - Copy the token

2. **On Linode Consul UI**:
   - Navigate to **Peers**
   - Click **Establish peering**
   - Paste the token
   - Name it **"eks"**
   - Wait for peering status to change to **Active**

### Step 8: Configure Service Failover

```bash
# On EKS cluster
kubectl config use-context <aws-context>
kubectl apply -f service-resolver.yaml

# On Linode cluster
kubectl config use-context <linode-context>
kubectl apply -f exported-service.yaml

# Verify peering has imported/exported services
# Check Consul UI under Peers tab
```

### Step 9: Test Failover

```bash
# Delete a service from EKS
kubectl delete deployment shippingservice

# The service should now route to Linode cluster automatically
# The application should still work
```

## Commands Reference

```bash
# View Consul peers
consul peering list

# Check peering details
consul peering read <peer-name>

# Switch between clusters
kubectl config use-context <context-name>
kubectl config current-context
kubectl config get-contexts

# View all resources
kubectl get pod
kubectl get svc
kubectl get mesh

# View Consul
kubectl get svc -l app=consul,component=ui
```

## Cleanup

```bash
# Destroy AWS resources
cd terraform
terraform destroy -var-file terraform.tfvars

# Delete Linode cluster from Linode dashboard
```

## Troubleshooting

**Pods in CrashLoopBackOff:**
- Check logs: `kubectl logs <pod-name>`
- paymentservice may crash due to Google Cloud Profiler - this is expected

**Peering not connecting:**
- Ensure both Consul servers are running: `kubectl get pod -l app=consul`
- Check Consul logs: `kubectl logs -l app=consul,component=server`
- Verify external IPs are reachable between clusters

**Storage issues:**
- EBS storage class must be `gp3` type (not `gp2`)
- Verify with: `kubectl get storageclass`


## Evidence of Work
 ### Set Up
![eks-services](./images/eks-services.png)
Consul Services dashboard on eks cluster showing 2 healthy services

![eks-cluster](./images/eks-cluster.png)
![lks-cluster](./images/lks-cluster.png)

### Peer Connection
![lks-peer](./images/lks-peers.png)
Linode Consul (lks) peers page showing initial state with pending connection to "eks" cluster. The "Add peer connection" button is visible for initiating peering.

![eks-peer](./images/lks-peers.png)
AWS Consul (eks) peers page showing active peering connection established with "lks" cluster. Status indicator shows both clusters are communicating successfully.

### Configure Failover
![eks-peer-details](./images/eks-peer-details.png)
Detailed view of eks peer connection showing:

Status: Active with recent heartbeat communication (a few seconds ago)
Exported Services: shippingservice (available for Linode cluster to import)
Server Addresses: Successfully connected to peer cluster servers

![lks-peer-details](./images/lks-peer-details.png)
Detailed view of lks peer connection showing:

Status: Pending (connection still initializing)
Imported Services: shippingservice from eks cluster (1 instance in service mesh with proxy)
Demonstrates failover setup - if eks shippingservice fails, requests route to lks
## Reference
* https://gitlab.com/twn-youtube/consul-crash-course  
* https://www.youtube.com/watch?v=s3I1kKKfjtQ