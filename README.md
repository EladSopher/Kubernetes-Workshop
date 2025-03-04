# WordPress Helm Chart Project

## Overview
This repository contains a Helm chart (`wordpress-helm-chart`) for deploying a WordPress application with a MariaDB database on Kubernetes. The chart packages all necessary Kubernetes resources into a reusable deployment, supporting both local clusters (e.g., Minikube) and cloud-managed clusters (e.g., AWS EKS). The project ensures all Kubernetes resources are managed via Helm and version-controlled in Git.

### Key Features
- Deploys a fully functional WordPress instance with persistent storage.
- Utilizes MariaDB as a StatefulSet for database reliability.
- Configurable deployments using Helm values.
- Integrates with an NGINX Ingress Controller for external access.

---

## Repository Structure
```
Kubernetes-Workshop/
├── ingress-values.yaml       # Configuration for the NGINX Ingress Controller
├── files/     # Helm chart for WordPress and MariaDB
│   │   ├── db-service.yaml       # Headless service for MariaDB
│   │   ├── db-statefulset.yaml   # MariaDB StatefulSet
│   │   ├── ebs-storageclass.yaml # EBS StorageClass (AWS)
│   │   ├── ingress-values.yaml   # Configuration for the NGINX Ingress Controller
│   │   ├── wordpress-deployment.yaml # WordPress Deployment
│   │   ├── wordpress-ingress.yaml    # Ingress resource
│   │   ├── wordpress-service.yaml    # WordPress Service
│   │   ├── wp-pvc.yaml           # Persistent storage for WordPress
├── wordpress-helm-chart/     # Helm chart for WordPress and MariaDB
│   ├── Chart.yaml            # Helm chart metadata
│   ├── values.yaml           # Default configuration values
│   ├── templates/            # Kubernetes resource templates
│   │   ├── db-service.yaml       # Headless service for MariaDB
│   │   ├── db-statefulset.yaml   # MariaDB StatefulSet
│   │   ├── ebs-storageclass.yaml # EBS StorageClass (AWS)
│   │   ├── wordpress-deployment.yaml # WordPress Deployment
│   │   ├── wordpress-ingress.yaml    # Ingress resource
│   │   ├── wordpress-service.yaml    # WordPress Service
│   │   ├── wp-pvc.yaml           # Persistent storage for WordPress
│   └── .helmignore           # Ignore list for Helm packaging
└── README.md                 # Project documentation (this file)
```

---

## Prerequisites
Ensure you have the following installed before proceeding:
- **Helm**: Version 3.x (`helm version` to check)
- **Kubernetes Cluster**:
  - **Local**: Minikube (`minikube start`)
  - **Cloud**: AWS EKS (configured with `kubectl` and AWS CLI)
- **Git**: To clone this repository (`git --version`)
- **AWS CLI**: Required for EKS access and ECR image pulling (optional)
- **ECR Access**: Images hosted at `992382545251.dkr.ecr.us-east-1.amazonaws.com/eladsopher/final-workshop`

---

## Installation Instructions

### 1. Clone the Repository
```bash
git clone https://github.com/EladSopher/Kubernetes-Workshop.git
cd Kubernetes-Workshop
```

### 2. Deploy NGINX Ingress Controller
The WordPress application requires an NGINX Ingress Controller for external access. Deploy it using Helm:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-elad-sopher ingress-nginx/ingress-nginx \
  --namespace eladsopher --create-namespace -f ingress-values.yaml
```
#### Verify the deployment:
```bash
kubectl get pods -n eladsopher -l app.kubernetes.io/name=ingress-nginx
```

### 3. Deploy the WordPress Helm Chart
#### **For AWS EKS:**
Configure your EKS cluster access:
```bash
aws eks update-kubeconfig --name eks-final-workshop --region us-east-1
kubectl config use-context arn:aws:eks:us-east-1:992382545251:cluster/eks-final-workshop
```
Deploy the Helm chart:
```bash
helm install my-wordpress ./wordpress-helm-chart \
  --namespace eladsopher --create-namespace --set imagePullSecrets.enabled=false
```

#### **For Minikube:**
Start Minikube:
```bash
minikube start
kubectl config use-context minikube
```
Deploy the Helm chart:
```bash
helm install my-wordpress ./wordpress-helm-chart \
  --namespace eladsopher --create-namespace
```

### 4. Access the Application
Retrieve the Ingress ELB hostname:
```bash
kubectl get svc -n eladsopher ingress-elad-sopher-ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```
Example output:
```
afd291ea56b0e40d9aba85bc4ac72707-1219273054.us-east-1.elb.amazonaws.com
```
Add the hostname to your system's hosts file (Windows example):
```
afd291ea56b0e40d9aba85bc4ac72707-1219273054.us-east-1.elb.amazonaws.com wordpress.local
```
Now, open `http://wordpress.local` in your browser.

---

## Chart Details
### **Resources Overview**
- **StorageClass**: `ebs-sc` (AWS EBS, `gp3`, 5Gi for WordPress, 1Gi for MariaDB)
- **Persistent Volume Claim (PVC)**: `wp-pvc` (for WordPress storage)
- **StatefulSet**: `db` (MariaDB with persistent volume)
- **Services**:
  - `db-service` (Headless MariaDB service)
  - `wordpress-service` (ClusterIP for WordPress)
- **Deployment**:
  - `wordpress` (2 replicas of WordPress pods)
- **Ingress**: `wordpress-ingress` (routes traffic to `wordpress.local`)

### **Customizing Values**
Modify `wordpress-helm-chart/values.yaml` as needed:
- `namespace`: Target namespace (default: `eladsopher`)
- `wordpress.replicas`: Number of WordPress instances (default: `2`)
- `db.image.repository`, `wordpress.image.repository`: Custom image sources
- `wordpress.host`: Ingress hostname (default: `wordpress.local`)
- `imagePullSecrets.enabled`: Toggle ECR authentication (default: `false` for EKS)

---

## Notes for Instructor
### **Project Highlights**
- **Helm Usage**: Fully parameterized chart using `values.yaml` for flexible deployments on EKS and Minikube.
- **Resource Management**: StatefulSet for MariaDB, Deployment for WordPress, dynamic PVCs with an EBS StorageClass.
- **Ingress Integration**: Custom NGINX Ingress Controller (`ingressClassName: eladsopher`) for external access.

### **Challenges Addressed**
- Optimized resource requests to prevent "Too many pods" issues.
- Resolved EBS volume multi-attach errors by manually detaching volumes.
- Worked around cluster webhook failures using `--validate=false`.

### **Examination Steps**
```bash
git clone https://github.com/eladsopher/wordpress-k8s-helm.git
```
Inspect the Helm chart:
```bash
helm template ./wordpress-helm-chart
```
Deploy using the steps above on EKS or Minikube and verify:
```bash
kubectl get all -n eladsopher
```

---

## Troubleshooting
### **Common Issues & Fixes**
- **Pods stuck in `Pending`**: Adjust resource requests or replicas in `values.yaml`.
- **EBS volume errors**: Ensure the EBS CSI driver is installed:
  ```bash
  eksctl create addon --name aws-ebs-csi-driver
  ```
- **Ingress connectivity issues**: Check logs:
  ```bash
  kubectl logs -n eladsopher -l app.kubernetes.io/name=ingress-nginx
  ```
- **Webhook validation errors**: Try applying resources with `--validate=false`.

---

This documentation provides a structured guide to deploying the WordPress Helm chart efficiently. 🚀

