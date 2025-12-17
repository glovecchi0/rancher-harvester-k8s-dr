# rancher-harvester-k8s-dr

Implement Cross-Cluster Kubernetes Disaster Recovery with Rancher using Harvester as Cloud Provider.

## Architecture Overview

```console
Rancher Server (Management Kubernetes Cluster)
 ├─ Longhorn (Persistent Block Storage / CSI)
 │   └─ Persistent Volumes for MinIO
 │
 └─ MinIO (S3-compatible Object Storage)
     └─ Bucket: velero-backups

Harvester Cluster AAA
 └─ RKE2 Cluster AAA
     └─ Velero
         └─ Backups → MinIO (on Rancher Server)

Harvester Cluster BBB
 └─ RKE2 Cluster BBB (empty / DR target)
     └─ Velero
         └─ Restores ← MinIO (on Rancher Server)
```

### Disaster Recovery Workflow

1. Rancher provisions RKE2 Cluster AAA on Harvester AAA
2. Velero backs up RKE2 Cluster AAA to MinIO on Rancher Server
3. A failure occurs on Harvester AAA or RKE2 Cluster AAA
4. Rancher provisions RKE2 Cluster BBB on Harvester BBB
5. Velero restores the backup from MinIO into RKE2 Cluster BBB
6. Applications and data are recovered on Harvester BBB

### Key Principles

- Velero runs inside each RKE2 cluster
- MinIO runs outside the protected RKE2 clusters
- Longhorn provides persistent storage for MinIO
- Backup storage is centralized and S3-compatible
- No VM-level or node-level backups are involved

## Prerequisites

### Infrastructure Deployment

Before starting the Disaster Recovery workflow, the following infrastructure components must be in place:

1. A **Rancher Server Kubernetes cluster** deployed and operational
  - The cluster must have **additional data disks attached to the nodes**
  - These disks are required for deploying **Longhorn** as persistent storage
  - Longhorn will be used to provide persistent volumes for MinIO
2. **Two Harvester clusters** already deployed and managed by Rancher
  - Harvester Cluster AAA (primary site)
  - Harvester Cluster BBB (disaster recovery site)
3. Rancher must be able to:
  - Provision RKE2 clusters on both Harvester clusters
  - Manage lifecycle operations for the downstream Kubernetes clusters

If the infrastructure is not already prepared, it must be deployed before following this guide.

##### Rancher Deployment

1. Clone the Repository

```bash
cd 
git clone git@github.com:rancher/tf-rancher-up.git
cd tf-rancher-up
```

2. Configure the `variables` File

Assuming the lab will be based on DigitalOcean, before running the Terraform code to deploy Rancher, you'll need to configure the environment-specific variables. You'll find the `terraform.tfvars.example` file in the recipe's root directory.

Copy this example file and rename it to `terraform.tfvars`:

```bash
cd recipes/upstream/google-cloud/rke2
cp terraform.tfvars.example terraform.tfvars
vi terraform.tfvars
```

Example:

```console
$ cat terraform.tfvars
prefix           = "demo-rancher-gl"
do_token         = "************"
region           = "fra1"
instance_count   = 3
os_type          = "sles"
rancher_hostname = "demo-rancher-gl"
rancher_password = "************"
```

3. Terraform Deploy

```bash
terraform init -upgrade && terraform apply -auto-approve
```

4. Create an API Key

To integrate Harvester clusters, you'll need to create an API key.
The [official documentation](https://ranchermanager.docs.rancher.com/reference-guides/user-settings/api-keys#creating-an-api-key) explains this process very well.

5. Enable the Harvester UI Extension

The [official documentation](https://docs.harvesterhci.io/v1.6/rancher/harvester-ui-extension) explains this process very well.

##### Harvester Cluster AAA Deployment

1. Clone the Repository

```bash
cd
git clone git@github.com:rancher/harvester-cloud.git
cd harvester-cloud
```

2. Configure the `variables` File

Assuming the lab will be based on DigitalOcean, before running the Terraform code to deploy Harvester, you'll need to configure the environment-specific variables. You'll find the `terraform.tfvars.example` file in the recipe's root directory.

Copy this example file and rename it to `terraform.tfvars`:

```bash
cd projects/google-cloud
cp terraform.tfvars.example terraform.tfvars
vi terraform.tfvars
```

Example:

```console
$ cat terraform.tfvars
prefix               = "hrv-gl-aaa"
do_token             = "************"
region               = "fra1"
harvester_node_count = 3
harvester_cluster_size = "medium"
rancher_api_url      = "https://demo-rancher-gl.<PUBLIC_IP>.sslip.io"
rancher_access_key   = "************"
rancher_secret_key   = "************"
rancher_insecure     = true
```

##### Harvester Cluster BBB Deployment

It will follow the same path as the previous point.

At the end of this phase, the environment must provide:
- One Rancher Server cluster
- Two Harvester clusters managed by Rancher
- A working base infrastructure ready for the Disaster Recovery workflow

The following sections describe the configuration required to enable cross-cluster Kubernetes Disaster Recovery.

## DR Workflow Setup Instructions

### 
