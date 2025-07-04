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

## 🛠️ Quick Start

### 1. Infrastructure Provisioning

```bash
# Clone the repository
git clone https://github.com/slimboi/bankapp.git
cd bankapp/infra

# Configure your settings
# Edit variables.tf - update ssh_key_name to your AWS key
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
# Create namespace
kubectl create namespace webapps

# Dry run to validate manifests
kubectl apply --dry-run=client -f k8s-manifests/

# Deploy application
kubectl apply -f k8s-manifests/ -n webapps
```

### Method 2: Helm Deployment

```bash
# Add Bitnami repository for MySQL dependency
helm repo add bitnami https://charts.bitnami.com/bitnami

# Create namespace
kubectl create namespace bankapp-project

# Navigate to Helm chart directory
cd helm-bankapp

# Update dependencies
helm dependency update

# Install application
helm install my-bankapp . --namespace bankapp-project
```

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
- Application accessible via LoadBalancer external IP
- Banking application login page should load
- Database connection successful

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
| Load balancer not accessible | External IP not reachable | Check security groups, VPC configuration, and AWS LB controller |
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

### Remove kubectl Deployment
```bash
kubectl delete namespace webapps
```

### Remove Helm Deployment
```bash
helm uninstall my-bankapp -n bankapp-project
kubectl delete namespace bankapp-project
```

### Remove Infrastructure
```bash
cd ../infra
terraform destroy --auto-approve
```

## 📖 Learning Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [AWS EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

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