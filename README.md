<!--
SPDX-FileCopyrightText: Copyright (c) 2025 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Deploying Dynamo on Amazon Elastic Kubernetes Service (EKS)


## 1. Prerequisites
Before you begin, ensure that you have the following:

* An AWS account with billing enabled.
* `aws` CLI using an IAM user with EKS permissions enabled
* A Docker Hub account (or compatible registry).
* Local tooling:
  * Docker Desktop
  * Earthly
  * Helm

### Install local tooling on macOS
```bash
brew install --cask docker        # Docker Desktop
brew install earthly              # Earthly build tool
brew install helm                 # Helm package manager
brew tap weaveworks/tap           # Eksctl cli for EKS
brew install weaveworks/tap/eksctl
```

---

## 2. Provision an EKS cluster
Dynamo requires at least 3 vCPU and 8 GiB RAM per node because BuildKit is CPU / memory intensive
An EC2 instance like t3.xlarge works for this purpose
```bash
# Create a managed 2-node t3.xlarge EKS cluster with oicd for IAM Roles
# Adjust the name and region to your preference.
eksctl create cluster 
  --name dynamo-eks 
  --version 1.33 
  --region us-east-2 
  --nodegroup-name cpu-nodes 
  --node-type t3.xlarge 
  --nodes 2 
  --nodes-min 2 
  --nodes-max 4 
  --node-volume-size 50 
  --managed 
  --with-oidc
```

Fetch cluster credentials so that `kubectl` talks to the right cluster:
```bash
aws eks update-kubeconfig --region us-east-2 --name dynamo-eks
kubectl get nodes   # quick verification
```

---

## 3. Build & push Docker images
Authenticate to Docker Hub and push all images using Earthly.
```bash
docker login docker.io

export DOCKER_SERVER=docker.io/<your-username>
export IMAGE_TAG=latest

# Run from the repository root (where the Earthfile lives)
earthly --push +all-docker \
  --DOCKER_SERVER=$DOCKER_SERVER \
  --IMAGE_TAG=$IMAGE_TAG
```

---

## 4. Prepare Kubernetes namespace & values
```bash
export DOCKER_USERNAME=<your-username>
export DOCKER_SERVER=docker.io/<your-username>
export IMAGE_TAG=latest
export NAMESPACE=dynamo-cloud

# Switch to the helm chart directory
cd deploy/cloud/helm

# Create and target the namespace
kubectl create namespace $NAMESPACE
kubectl config set-context --current --namespace=$NAMESPACE
```

---

## 5. Deploy Dynamo via Helm
Run the bundled deployment script which wires up all services, deployments, and CRDs.
```bash
./deploy.sh
```

You can track progress with:
```bash
kubectl get pods -w
```

---

## 6. (Optional) Install an NGINX Ingress controller
If you need external HTTP(S) access to Dynamo services, install the community NGINX controller in its own namespace.
```bash
kubectl create namespace ingress-nginx

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.publishService.enabled=true
```

> ⚠️  Helm 3.18.0 has a known issue with the NGINX chart. Stick to **v3.17.3** (or newer once patched) if you encounter errors.

---

## 7. Troubleshooting
* **Helm install fails on `ingress-nginx`** – Downgrade to Helm 3.17.3 as noted above.

---

## 8. Next steps
After deploying the Dynamo cloud platform, you can:

1. Deploy your first inference graph using the [Dynamo CLI](operator_deployment.md)
2. Deploy Dynamo LLM pipelines to Kubernetes using the [Dynamo CLI](../../examples/llm_deployment.md)!
3. Manage your deployments using the Dynamo CLI
