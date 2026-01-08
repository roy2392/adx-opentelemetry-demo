# OpenTelemetry Demo - Azure Infrastructure

This Terraform configuration deploys the complete Azure infrastructure needed to run the OpenTelemetry Demo application with Azure Data Explorer (ADX) for observability.

## 🏗️ What Gets Deployed

This configuration creates:

- **Azure Data Explorer (ADX) Cluster** - For storing and querying telemetry data
  - Database with pre-configured tables for traces, metrics, and logs
  - Ingestion mappings for OpenTelemetry data
  - Automated schema creation
  
- **Azure Kubernetes Service (AKS)** - To run the demo application
  - Workload Identity enabled for secure Azure authentication
  - Auto-scaling node pool
  - Network isolation with VNet

- **Managed Identity** - For secure service-to-service authentication
  - Federated credentials for AKS workload identity
  - Data Ingestor role for ADX
  - Data Reader role for Grafana

- **Virtual Network** - Network infrastructure
  - Dedicated subnet for AKS
  - Network security configurations

## 📋 Prerequisites

Before you begin, ensure you have:

1. **Azure Account** with an active subscription
   - [Create a free account](https://azure.microsoft.com/free/) if you don't have one

2. **Azure CLI** installed and authenticated
   ```bash
   # Install Azure CLI (macOS)
   brew install azure-cli
   
   # Or download from: https://docs.microsoft.com/cli/azure/install-azure-cli
   
   # Login to Azure
   az login
   
   # Set your subscription (if you have multiple)
   az account set --subscription "YOUR_SUBSCRIPTION_ID"
   
   # Verify you're logged in
   az account show

   # Register Communication service
   az provider register --namespace Microsoft.Communication 
   ```

3. **Terraform** installed (version 1.0+)
   ```bash
   # Install Terraform (macOS)
   brew tap hashicorp/tap
   brew install hashicorp/tap/terraform
   
   # Verify installation
   terraform version
   ```

4. **Required Permissions** - Your Azure account needs:
   - Contributor role on the subscription (to create resources)
   - User Access Administrator role (to assign roles to managed identities)

## 🚀 Quick Start

### Step 1: Clone and Navigate

```bash
cd terraform
```

### Step 2: Configure Your Variables

Copy the example configuration file:

```bash
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars` with your preferred editor:

```bash
# Minimum required configuration
project_name = "otel-demo"        # Your project name (lowercase, alphanumeric)
environment  = "dev"              # dev, staging, or prod
location     = "eastus"           # Azure region (eastus, westus2, etc.)

# Optional: Add your email to tags for cost tracking
tags = {
  Project   = "OpenTelemetry Demo"
  Owner     = "your-email@example.com"
  ManagedBy = "Terraform"
}
```

**💡 Cost Optimization Tip:** For development/testing, use the default `Dev(No SLA)_Standard_D11_v2` SKU which is significantly cheaper.

### Step 3: Initialize Terraform

Download required providers and modules:

```bash
terraform init
```

### Step 4: Preview Changes

Review what will be created:

```bash
terraform plan
```

This shows you all resources that will be created without actually creating them.

### Step 5: Deploy Infrastructure

Create the infrastructure:

```bash
terraform apply
```

Type `yes` when prompted to confirm.

⏱️ **Expected deployment time:** 15-20 minutes

### Step 6: Get Your Connection Information

After successful deployment, Terraform outputs important values:

```bash
# View all outputs
terraform output

# Get specific values for Helm chart
terraform output -raw adx_cluster_uri
terraform output -raw aks_cluster_name
terraform output -raw managed_identity_client_id
```

### Step 7: Configure kubectl

Connect to your new AKS cluster:

```bash
az aks get-credentials \
  --resource-group $(terraform output -raw resource_group_name) \
  --name $(terraform output -raw aks_cluster_name)

# Verify connection
kubectl get nodes
```

## 📝 Configuration Reference

### Essential Variables

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `project_name` | Prefix for all resources | `otel-demo` | `mycompany-otel` |
| `environment` | Environment name | `dev` | `dev`, `staging`, `prod` |
| `location` | Azure region | `eastus` | `westus2`, `westeurope` |

### ADX Configuration

| Variable | Description | Default | Notes |
|----------|-------------|---------|-------|
| `adx_sku_name` | ADX cluster SKU | `Dev(No SLA)_Standard_D11_v2` | Use `Standard_D11_v2` for production |
| `adx_sku_capacity` | Number of instances | `1` | Min: 1, Max: 1000 |
| `adx_database_name` | Database name | `otel_demo` | Lowercase, alphanumeric |
| `adx_hot_cache_days` | Days in hot cache | `30` | SSD storage period |
| `adx_retention_days` | Total retention days | `365` | Includes cold storage |

### AKS Configuration

| Variable | Description | Default | Notes |
|----------|-------------|---------|-------|
| `aks_kubernetes_version` | K8s version | `1.32` | Check available versions with `az aks get-versions -l eastus` |
| `aks_default_node_pool_vm_size` | VM size | `Standard_DS2_v2` | 2 vCPU, 7GB RAM |
| `aks_default_node_pool_count` | Initial nodes | `2` | Min: 1, Max: 100 |
| `aks_min_node_count` | Auto-scale minimum | `1` | For auto-scaling |
| `aks_max_node_count` | Auto-scale maximum | `5` | For auto-scaling |

## 🔧 Common Operations

### Update Infrastructure

After modifying `terraform.tfvars`:

```bash
terraform plan    # Preview changes
terraform apply   # Apply changes
```

### View Current State

```bash
terraform show
```

### Destroy Infrastructure

⚠️ **WARNING:** This permanently deletes all resources!

```bash
terraform destroy
```

Type `yes` when prompted. This will:
- Delete AKS cluster
- Delete ADX cluster (including all data!)
- Delete managed identities
- Delete resource group

## 🎯 Next Steps

After infrastructure is deployed:

1. **Deploy the OpenTelemetry Demo application** to AKS:
   ```bash
   cd ../kubernetes
   # Follow the instructions in kubernetes/README.md
   ```

2. **Configure your Helm values** with the Terraform outputs:
   - ADX cluster URI
   - Managed identity client ID
   - Azure tenant ID

3. **Access Grafana** to visualize your telemetry data

## 💰 Cost Estimation

Approximate monthly costs (as of 2026):

| Component | Configuration | Est. Monthly Cost |
|-----------|--------------|-------------------|
| ADX Dev Cluster | 1x D11_v2 (Dev/No SLA) | ~$60-100 |
| AKS Cluster | 2x Standard_DS2_v2 nodes | ~$140 |
| Networking | VNet, Load Balancer | ~$20 |
| **Total** | | **~$220-260** |

💡 **Cost Savings:**
- Use `adx_auto_stop_enabled = true` to auto-stop ADX when idle
- Enable AKS auto-scaling to scale down during off-hours
- Delete when not in use: `terraform destroy`

## 🐛 Troubleshooting

### "Error: Insufficient permissions"

**Solution:** Ensure your Azure account has Contributor role:
```bash
az role assignment create \
  --assignee YOUR_EMAIL@example.com \
  --role Contributor \
  --scope /subscriptions/YOUR_SUBSCRIPTION_ID
```

### "Error: Cluster name already exists"

**Solution:** Change `project_name` in `terraform.tfvars` to something unique.

### "Error: Kubernetes version not available"

**Solution:** Check available versions:
```bash
az aks get-versions --location eastus --output table
```
Update `aks_kubernetes_version` in `terraform.tfvars`.

### "terraform.tfvars: no such file"

**Solution:** Copy the example file:
```bash
cp terraform.tfvars.example terraform.tfvars
```

### Check Terraform Logs

Enable detailed logging:
```bash
export TF_LOG=DEBUG
terraform apply
```

## 📚 Additional Resources

- [Azure Data Explorer Documentation](https://docs.microsoft.com/azure/data-explorer/)
- [Azure Kubernetes Service Documentation](https://docs.microsoft.com/azure/aks/)
- [Terraform AzureRM Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)

## 🔐 Security Best Practices

1. **Never commit `terraform.tfvars`** to version control
   - It may contain sensitive information
   - Already in `.gitignore`

2. **Use Workload Identity** (enabled by default)
   - More secure than service principal keys
   - Automatic credential rotation

3. **Review outputs before sharing**
   - Some outputs contain cluster endpoints
   - Don't share in public channels

4. **Enable private cluster** for production
   - Set `aks_enable_private_cluster = true`
   - Requires VPN or bastion host access

## 📄 License

Apache License 2.0 - See main repository LICENSE file
