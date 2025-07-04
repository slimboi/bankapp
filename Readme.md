# Banking Application Deployment on AWS EKS

A complete cloud-native banking application deployed on Amazon EKS using Terraform for infrastructure provisioning and Kubernetes for container orchestration.

## 🏗️ Architecture Overview

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Load Balancer │────│  Banking App     │────│     MySQL       │
│   (AWS ELB)     │    │  (Spring Boot)   │    │   (Persistent)  │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                       │                      │
         │              ┌────────┴────────┐             │
         │              │                 │             │
         └──────────────│   AWS EKS       │─────────────┘
                        │   Cluster       │
                        └─────────────────┘
```

## 🚀 Features

- **Infrastructure as Code**: Terraform-managed AWS EKS cluster
- **Persistent Database**: MySQL with EBS-backed persistent volumes
- **Auto-scaling**: Kubernetes deployments with resource management
- **Health Monitoring**: Comprehensive liveness and readiness probes
- **Secure Configuration**: Kubernetes secrets and configmaps
- **Custom Domain**: DNS configuration with load balancer integration
- **Helm Support**: Both kubectl and Helm deployment methods

## 📋 Prerequisites

- AWS CLI configured with appropriate permissions
- Terraform >= 1.0
- kubectl
- eksctl
- Docker
- Helm >= 3.0
- An existing SSH key pair in your AWS account
- A domain name (optional, for custom domain setup)

## 📂 Project Structure

```
bankapp/
├── helm-bankapp/                    # Helm chart for application deployment
│   ├── Chart.lock                   # Helm dependency lock file
│   ├── Chart.yaml                   # Helm chart metadata
│   ├── charts/                      # Chart dependencies
│   │   └── mysql-13.0.2.tgz        # MySQL Helm chart dependency
│   ├── templates/                   # Kubernetes manifest templates
│   │   ├── storageclass.yaml        # EBS storage class definition
│   │   ├── bankapp-deployment.yaml  # Banking app deployment
│   │   └── bankapp-service.yaml     # Banking app service
│   └── values.yaml                  # Helm chart configuration values
├── infra/                           # Terraform infrastructure code
│   ├── main.tf                      # Main infrastructure configuration
│   ├── output.tf                    # Terraform outputs
│   ├── variable.tf                  # Terraform variables
│   ├── terraform.tfstate            # Terraform state file
│   └── terraform.tfstate.backup     # Terraform state backup
├── k8s-manifests/                   # Raw Kubernetes manifests
│   ├── banking-app-deployment.yaml  # Banking app deployment
│   ├── banking-app-manifest.yaml    # Combined banking app manifest
│   ├── banking-app-service.yaml     # Banking app service
│   ├── configmap.yaml               # MySQL configuration
│   ├── mysql-deployment.yaml        # MySQL deployment
│   ├── mysql-service.yaml           # MySQL service
│   ├── pvc.yaml                     # Persistent volume claim
│   ├── secret.yaml                  # MySQL credentials
│   └── storageclass.yaml            # EBS storage class
└── README.md                        # This documentation
```

## 🛠️ Quick Start

### 1. Infrastructure Provisioning

```bash
# Clone the repository
git clone https://github.com/slimboi/bankapp.git
cd bankapp/infra

# Configure your settings
# Edit variable.tf - update ssh_key_name to your AWS key
# Edit main.tf - update region (e.g., ap-southeast-2)

# Format, initialize and validate Terraform
terraform fmt
terraform init
terraform validate

# Deploy infrastructure
terraform plan
terraform apply --auto-approve
```

⏱️ **Expected deployment time**: ~10 minutes

### 2. Kubernetes Configuration

```bash
# Update kubeconfig for EKS cluster
aws eks --region ap-southeast-2 update-kubeconfig --name bankapp-cluster

# Associate IAM OIDC provider
eksctl utils associate-iam-oidc-provider \
  --region ap-southeast-2 \
  --cluster bankapp-cluster \
  --approve

# Create service account for EBS CSI driver
eksctl create iamserviceaccount \
  --region ap-southeast-2 \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster bankapp-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts

# Install EBS CSI driver
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.11"
```

**Verify EBS CSI driver installation:**
```bash
kubectl get all -n kube-system | grep ebs-csi
# Expected output: CSI controller and node pods in ContainerCreating/Running state
```

## 🎯 Deployment Methods

### Method 1: kubectl Deployment

```bash
# Navigate to project root
cd bankapp

# Create namespace
kubectl create namespace webapps

# Dry run to validate manifests
kubectl apply --dry-run=client -f k8s-manifests/

# Deploy application
kubectl apply -f k8s-manifests/ -n webapps
```

**Available manifest files:**
- `banking-app-deployment.yaml` - Banking application deployment
- `banking-app-service.yaml` - Banking application service
- `banking-app-manifest.yaml` - Combined banking app manifest
- `mysql-deployment.yaml` - MySQL database deployment
- `mysql-service.yaml` - MySQL database service
- `configmap.yaml` - MySQL configuration
- `secret.yaml` - MySQL credentials
- `pvc.yaml` - Persistent volume claim
- `storageclass.yaml` - EBS storage class

### Method 2: Helm Deployment

```bash
# Add Bitnami repository for MySQL dependency
helm repo add bitnami https://charts.bitnami.com/bitnami

# Create namespace
kubectl create namespace bankapp-project

# Navigate to Helm chart directory
cd helm-bankapp

# Update dependencies (this will download mysql-13.0.2.tgz)
helm dependency update

# Install application
helm install my-bankapp . --namespace bankapp-project
```

**Helm Chart Structure:**
- `Chart.yaml` - Chart metadata and dependencies
- `values.yaml` - Default configuration values
- `templates/` - Kubernetes manifest templates
  - `storageclass.yaml` - EBS storage class template
  - `bankapp-deployment.yaml` - Banking app deployment template
  - `bankapp-service.yaml` - Banking app service template
- `charts/mysql-13.0.2.tgz` - MySQL dependency chart

## 🔍 Verification & Testing

### Check Deployment Status

```bash
# For kubectl deployment
kubectl get all -n webapps

# For Helm deployment
kubectl get all -n bankapp-project
helm list -n bankapp-project
```

### Verify Persistent Volumes

```bash
# Check persistent volumes
kubectl get pv

# Check persistent volume claims
kubectl get pvc -n <namespace>
```

### Access Application

```bash
# Get load balancer endpoint
kubectl get service bankapp-service -n <namespace>

# Check application logs
kubectl logs -l app=bankapp -n <namespace>
```

**Expected Output:**
- Application accessible via LoadBalancer external URL (AWS ELB)
- Banking application login page should load in browser
- Database connection successful
- LoadBalancer will have a format like: `a5db4ceba255546f78b8135eb5f355f7-243499697.ap-southeast-2.elb.amazonaws.com`

## 📁 Application Components

### Database Layer
- **MySQL 8**: Production database with persistent storage
- **Storage**: 5Gi EBS volume with `gp3` type
- **Security**: Root password stored in Kubernetes secret
- **Health Checks**: TCP socket and mysqladmin ping probes

### Application Layer
- **Spring Boot**: Banking application (v6)
- **Resources**: 256Mi-512Mi memory, 250m-500m CPU
- **Database Connection**: JDBC connection to MySQL service
- **Health Endpoints**: `/login` endpoint for health monitoring

### Networking
- **Internal**: ClusterIP service for MySQL (port 3306)
- **External**: LoadBalancer service for web app (port 80 → 8080)
- **Domain**: Optional custom domain configuration

## 🌐 Custom Domain Setup (Optional)

1. **Get Load Balancer URL**:
   ```bash
   kubectl get service bankapp-service -n <namespace> -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
   ```

2. **Configure DNS** (example with GoDaddy):
   - **Type**: CNAME
   - **Name**: bankapp
   - **Value**: `<your-elb-url>.elb.amazonaws.com`

3. **Verify DNS Propagation**:
   - Visit: https://www.whatsmydns.net
   - Check: `bankapp.yourdomain.com` (CNAME record)
   - Command: `nslookup bankapp.yourdomain.com`

4. **Access Application**:
   - URL: `http://bankapp.yourdomain.com` (note: HTTP, not HTTPS)

## 🔧 Configuration Files

### Key Configuration Parameters

| Component | Configuration | Value |
|-----------|---------------|-------|
| MySQL | Root Password | `Test@123` (base64: `VGVzdEAxMjM=`) |
| MySQL | Database Name | `bankappdb` |
| Storage | Volume Size | `5Gi` |
| Storage | Storage Class | `ebs-sc` (gp3) |
| App | Image | `adijaiswal/bankapp:v6` |
| App | Replicas | `2` (kubectl) / `2` (Helm) |

### Environment Variables

```yaml
# Database connection
SPRING_DATASOURCE_URL: jdbc:mysql://mysql-service:3306/bankappdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
SPRING_DATASOURCE_USERNAME: root
SPRING_DATASOURCE_PASSWORD: <from-secret>
```

## 📊 Resource Requirements

### MySQL Database
- **CPU**: 500m (request) / 1 core (limit)
- **Memory**: 512Mi (request) / 1Gi (limit)
- **Storage**: 5Gi persistent volume

### Banking Application
- **CPU**: 250m (request) / 500m (limit)
- **Memory**: 256Mi (request) / 512Mi (limit)
- **Replicas**: 2 (can be scaled)

## 🔍 Monitoring & Troubleshooting

### Common Commands

```bash
# View all resources
kubectl get all -n <namespace>

# Check pod status
kubectl describe pod <pod-name> -n <namespace>

# View application logs
kubectl logs <pod-name> -n <namespace>

# Check service endpoints
kubectl get endpoints -n <namespace>

# Verify persistent volume binding
kubectl describe pvc mysql-pvc -n <namespace>
```

### Common Issues & Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| Pod stuck in Pending | Pod doesn't start | Check node capacity, storage class, and PVC binding |
| MySQL connection failed | App can't connect to DB | Verify service names, secrets, and network policies |
| Load balancer not accessible | External URL not reachable | Check security groups, VPC configuration, and AWS LB controller |
| Domain not resolving | Custom domain doesn't work | Verify DNS propagation and CNAME record configuration |
| Storage issues | Database data loss | Check PVC status and EBS CSI driver installation |

### Debugging Commands

```bash
# Debug networking
kubectl exec -it <pod-name> -n <namespace> -- nslookup mysql-service

# Check resource usage
kubectl top nodes
kubectl top pods -n <namespace>

# View events
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

## 🚀 Scaling & Production Considerations

### Horizontal Scaling
```bash
# Scale banking application (kubectl)
kubectl scale deployment bankapp --replicas=3 -n webapps

# Scale banking application (Helm)
helm upgrade my-bankapp . --set replicaCount=3 -n bankapp-project
```

### Production Improvements
- [ ] Implement MySQL high availability (master-slave)
- [ ] Add SSL/TLS certificates
- [ ] Configure ingress controller (NGINX/ALB)
- [ ] Implement monitoring (Prometheus/Grafana)
- [ ] Add CI/CD pipeline
- [ ] Configure backup strategies
- [ ] Implement network policies
- [ ] Add resource quotas and limits
- [ ] Configure pod disruption budgets

## 🧹 Cleanup

### Complete Cleanup Process

**⚠️ Important**: Follow this cleanup order to avoid orphaned AWS resources and additional charges.

#### Step 1: Remove Kubernetes Applications
```bash
# Remove kubectl deployment
kubectl delete namespace webapps

# OR remove Helm deployment
helm uninstall my-bankapp -n bankapp-project
kubectl delete namespace bankapp-project
```

#### Step 2: Remove EKS Add-ons and Service Accounts
```bash
# Delete EBS CSI service account (created with eksctl)
eksctl delete iamserviceaccount \
  --region ap-southeast-2 \
  --cluster bankapp-cluster \
  --namespace kube-system \
  --name ebs-csi-controller-sa

# Delete OIDC provider association
eksctl utils disassociate-iam-oidc-provider \
  --region ap-southeast-2 \
  --cluster bankapp-cluster \
  --approve
```

#### Step 3: Manual AWS Console Cleanup (if needed)
If the above commands don't remove everything, manually delete from AWS Console:

1. **EBS Volumes**:
   - Go to EC2 → Volumes
   - Delete any volumes with tags related to your cluster
   - Look for volumes in "available" state after cluster deletion

2. **IAM Service Account**:
   - Go to IAM → Roles
   - Delete: `eksctl-bankapp-cluster-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa`

3. **OIDC Provider**:
   - Go to IAM → Identity providers
   - Delete OIDC provider for your EKS cluster

4. **Load Balancers**:
   - Go to EC2 → Load Balancers
   - Delete any ELB created by Kubernetes services

#### Step 4: Remove Infrastructure
```bash
cd infra
terraform destroy --auto-approve
```

### Cleanup Verification Commands
```bash
# Verify no running pods
kubectl get pods --all-namespaces

# Check for remaining PVs
kubectl get pv

# Verify EKS cluster removal
aws eks describe-cluster --name bankapp-cluster --region ap-southeast-2
# Should return: ResourceNotFoundException

# Check for remaining EBS volumes
aws ec2 describe-volumes --region ap-southeast-2 \
  --filters "Name=tag:kubernetes.io/cluster/bankapp-cluster,Values=owned"
```

### Cost Optimization Tips
- **EBS Volumes**: Can incur charges even when detached (~$0.10/GB/month)
- **Load Balancers**: Classic/Network Load Balancers charge hourly (~$18/month)
- **EKS Cluster**: Charges $0.10/hour (~$73/month) just for the control plane
- **EC2 Instances**: Node groups will incur compute charges

### Troubleshooting Cleanup Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| EBS volumes remain after deletion | PV reclaim policy or finalizers | Manual deletion from AWS Console |
| IAM service account persists | eksctl didn't clean up properly | Delete manually from IAM console |
| Load balancer remains | Service deletion didn't trigger cleanup | Delete manually from EC2 console |
| OIDC provider remains | Disassociate command failed | Delete manually from IAM console |

## 🔧 File-Specific Configurations

### Terraform Files (`infra/`)

#### `main.tf`
Contains the main EKS cluster configuration:
- EKS cluster setup
- Node group configuration
- VPC and networking setup
- Security groups

#### `variable.tf`
Key variables to customize:
```hcl
variable "ssh_key_name" {
  description = "Name of the SSH key pair in AWS"
  type        = string
  default     = "your-key-name"  # Update this
}

variable "region" {
  description = "AWS region"
  type        = string
  default     = "ap-southeast-2"  # Update as needed
}
```

#### `output.tf`
Outputs important information:
- EKS cluster endpoint
- EKS cluster name
- Node group details

### Kubernetes Manifests (`k8s-manifests/`)

#### Key Configuration Files:
- **`secret.yaml`**: Contains MySQL root password (base64 encoded)
- **`configmap.yaml`**: MySQL configuration settings
- **`storageclass.yaml`**: EBS storage class with gp3 type
- **`pvc.yaml`**: 5Gi persistent volume claim for MySQL data

### Helm Chart (`helm-bankapp/`)

#### `Chart.yaml`
```yaml
dependencies:
  - name: mysql
    version: 13.0.2
    repository: https://charts.bitnami.com/bitnami
```

#### `values.yaml`
Configure application settings:
- Replica count
- Resource limits
- MySQL configuration
- Service type and ports

## 📖 Learning Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [AWS EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Bitnami MySQL Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/mysql)

## 🎯 DevOps Learning Outcomes

This project demonstrates proficiency in:

### Infrastructure as Code
- **Terraform**: AWS EKS cluster provisioning
- **Resource Management**: VPC, security groups, node groups
- **State Management**: Terraform state handling

### Container Orchestration
- **Kubernetes**: Pod, deployment, service, and persistent volume management
- **Storage**: EBS CSI driver integration and persistent volumes
- **Networking**: Service discovery and load balancing

### Package Management
- **Helm**: Chart creation, dependency management, and templating
- **Chart Development**: Custom templates and values configuration

### AWS Services
- **EKS**: Managed Kubernetes service
- **EBS**: Persistent storage for databases
- **ELB**: Load balancing for external access
- **IAM**: Service accounts and OIDC integration

### AWS Resource Management
- **EKS Cleanup**: Understanding eksctl-created resources
- **Cost Optimization**: Identifying and removing billable resources
- **Manual Cleanup**: AWS Console resource management
- **Resource Tagging**: Kubernetes resource identification in AWS

### Best Practices
- **Security**: Kubernetes secrets and configmaps
- **Monitoring**: Health checks and probes
- **Scalability**: Resource limits and horizontal scaling
- **Documentation**: Comprehensive project documentation

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## 📝 License

This project is licensed under the MIT License - see the LICENSE file for details.

---

**Note**: This documentation assumes basic knowledge of Kubernetes, Docker, and AWS services. For production deployments, additional security and performance considerations should be implemented.