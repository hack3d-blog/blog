---
title: "Zero to Production: Part 1"
date: 2022-09-01T14:45:53-05:00
draft: false
categories:
  - DevOps
  - Kubernetes
tags: &tags
  - kubernetes
  - k8s
  - ansible
  - helm
  - terraform
  - linode
series: 
  - "Zero to Production"
keywords: *tags
---
# Goals
* Deploy Kubernetes Cluster using cost effective hosting
    * Linode
    * Using Terraform
* Cluster has DNS records automatically configured
    * external-dns
* Cluster has SSL certificates provisioned
    * cert-manager

# Step 1: Set up local environment
```
export AWS_ACCESS_KEY_ID=<Linode Object Storage Access Key>
export AWS_SECRET_ACCESS_KEY=<Linode Object Storage Secret Key>
export LINODE_TOKEN=<Linode API Personal Access Token>
```
# Step 2: (Optional) Create object storage for Terraform
```
s3cmd mb s3://hack3d-tfstate/
```
# Step 3: Provision Kubernetes Cluster
First up is provisioning a Kubernetes cluster. To do this, I recommend Terraform. There's a lot of assumptions made around Terraform for this guide.

1. Create a directory to manage your configs. I've opted to create a GitHub repository:
```
# mkdir cluster-configs
# cd cluster-configs
```

2. Create a directory for Terraform
```
# mkdir terraform
# cd terraform
```

3. Create `terraform.tf`
```
# cluster-configs/terraform/terraform.tf
terraform {
  backend "s3" {
    bucket                      = "hack3d-tfstate"
    key                         = "production/terraform.tfstate"
    region                      = "us-east-1"
    endpoint                    = "us-east-1.linodeobjects.com"
    force_path_style            = true 
    skip_credentials_validation = true
  }
  required_providers {
    linode = {
      source  = "linode/linode"
      version = "1.29.2"
    }
  }
}
```

4. Create `providers.tf`
```
provider "linode" { }
```

5. Create `main.tf`
```
resource "linode_lke_cluster" "cluster" {
    label       = "production"
    k8s_version = "1.23"
    region      = "us-central"
    tags        = ["prod"]

    pool {
        type  = "g6-standard-2"
        count = 3
    }
}

output "kubeconfig" {
    sensitive = true
    value = linode_lke_cluster.cluster.kubeconfig
}
```

6. Run Terraform apply
7. Get Kubernetes config from terraform outputs
```
# mkdir -p ~/.kube
# terraform output kubeconfig | sed -e's/\"//g' | base64 -d > ~/.kube/config
```

# Step 4: Install and Configure external-dns
```
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm upgrade --install external-dns external-dns/external-dns \
    --set serviceMonitor.enabled=true \
    --set provider=cloudflare \
    --set env[0].name=CF_API_TOKEN \
    --set env[0]value="<CloudFlare API Token>"
```
# Step 5: Install and Configure cert-manager
1. Install `cert-manager` helm chart:
```
helm repo add jetstack https://charts.jetstack.io
helm upgrade --install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --set installCRDs=true
```
2. Create a `ClusterIssuer` resource
```
# cat << EOF | kubectl apply
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your@email.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-secret-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```