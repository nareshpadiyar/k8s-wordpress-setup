# WordPress Deployment on Kubernetes

A comprehensive guide and reference implementation for deploying a production-ready, two-tier WordPress application architecture on a Kubernetes cluster (specifically tested on IBM Cloud Kubernetes Service - IKS).

---

## 📋 Table of Contents

- [Architecture Overview](#-architecture-overview)
- [Tech Stack & Components](#-tech-stack--components)
- [Prerequisites](#-prerequisites)
- [Repository Structure](#-repository-structure)
- [Important Architectural Notes](#-important-architectural-notes)
- [Step-by-Step Deployment](#-step-by-step-deployment)
  - [1. Connect to Cluster](#1-connect-to-cluster)
  - [2. Create Namespace](#2-create-namespace)
  - [3. Deploy Database Tier (MySQL)](#3-deploy-database-tier-mysql)
  - [4. Deploy Application Tier (WordPress)](#4-deploy-application-tier-wordpress)
- [Verification & Access](#-verification--access)
- [Useful Commands & Troubleshooting](#-useful-commands--troubleshooting)
- [Cleanup](#-cleanup)

---

## 🏗 Architecture Overview

The setup consists of a decoupled **Two-Tier Architecture**:

1. **Database Tier:**
   - **Workload:** MySQL 5.7 running as a `StatefulSet` (1 Replica).
   - **Storage:** Static `PersistentVolume` (PV) and `PersistentVolumeClaim` (PVC) providing persistent database storage.
   - **Networking:** Internal communication via a `ClusterIP` Service on port `3306`.
   - **Security:** Passwords managed securely via Kubernetes `Secret`.

2. **Application Tier:**
   - **Workload:** WordPress (`wordpress:5.8.3-php7.4-apache`) deployed as a `Deployment` (2 Replicas).
   - **Storage:** Persistent storage attached to `/var/www/html` via PV/PVC for website media and assets.
   - **Networking:** Publicly exposed using a `LoadBalancer` Service on port `80`.

```
                  +----------------------------------------+
                  |           Internet / Client            |
                  +----------------------------------------+
                                      |
                                      v
                  +----------------------------------------+
                  |   Service: wordpress-service           |
                  |         (Type: LoadBalancer)           |
                  +----------------------------------------+
                                      |
                        +-------------+-------------+
                        |                           |
                        v                           v
             +--------------------+       +--------------------+
             | Pod: wordpress-1   |       | Pod: wordpress-2   |
             +--------------------+       +--------------------+
                        \                           /
                         +------------+------------+
                                      |
                                      v
                  +----------------------------------------+
                  |         Service: mysql-service         |
                  |           (Type: ClusterIP)            |
                  +----------------------------------------+
                                      |
                                      v
                  +----------------------------------------+
                  |            Pod: mysql-0                |
                  |         (Type: StatefulSet)            |
                  +----------------------------------------+
                                      |
                                      v
                  +----------------------------------------+
                  |         PVC: mysql-pvc (5Gi)           |
                  +----------------------------------------+
```

---

## 🛠 Tech Stack & Components

- **Container Platform:** Kubernetes / IBM Cloud Kubernetes Service (IKS)
- **Database:** MySQL 5.7 (`mysql:5.7`)
- **Web Application:** WordPress (`wordpress:5.8.3-php7.4-apache`)
- **CLI Tools:** `kubectl`, `ibmcloud` CLI

---

## ⚡ Prerequisites

Before starting, ensure you have:

- Access to an active **Kubernetes Cluster** (e.g., IBM Cloud Kubernetes Service - IKS).
- `kubectl` CLI tool installed locally.
- `ibmcloud` CLI installed (if using IKS).
- Basic familiarity with Kubernetes objects (`Secrets`, `PV/PVC`, `StatefulSet`, `Deployment`, `Service`).

---

## 📁 Repository Structure

```
.
├── mysql-secret.yaml          # Kubernetes Secret for MySQL credentials
├── mysql-pv-pvc.yaml          # PersistentVolume and Claim for MySQL storage
├── mysql-statefulset.yaml     # MySQL StatefulSet manifest
├── mysql-service.yaml        # Internal ClusterIP Service for MySQL
├── wordpress-pv-pvc.yaml      # PersistentVolume and Claim for WordPress assets
├── wordpress-deployment.yaml  # WordPress Deployment manifest (2 Replicas)
└── wordpress-service.yaml     # External LoadBalancer Service for WordPress
```

---

## ⚠️ Important Architectural Notes

> **Database Scaling Notice:**  
> A single-instance MySQL setup cannot be scaled horizontally simply by increasing replica counts. For a `StatefulSet` to scale horizontally, each pod requires its own independent PVC (typically created using `volumeClaimTemplates`). 
> 
> For **production environments**, consider using database replication solutions such as:
> - **Bitnami MySQL Operator**
> - **MySQL InnoDB Cluster**
> - **Percona XtraDB Cluster**

---

## 🚀 Step-by-Step Deployment

### 1. Connect to Cluster

Authenticate and configure `kubectl` to target your cluster (example shown for IKS):

```bash
ibmcloud login -a cloud.ibm.com --sso
ibmcloud ks cluster config -c {CLUSTER_ID}

# Verify connectivity
kubectl get nodes -o wide
```

### 2. Create Namespace

Isolate the application resources inside a dedicated namespace:

```bash
kubectl create namespace wordpress-app
```

---

### 3. Deploy Database Tier (MySQL)

#### Step 3.1: Create Secret
Define the `mysql-secret.yaml` file:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: wordpress-app
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: mypassword
  MYSQL_DATABASE: wordpress
```

Apply the secret:
```bash
kubectl apply -f mysql-secret.yaml
```

#### Step 3.2: Create Persistent Volume & Persistent Volume Claim
Define `mysql-pv-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    app: mysql
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data/mysql
    type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: wordpress-app
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      app: mysql
```

Apply the PV and PVC:
```bash
kubectl apply -f mysql-pv-pvc.yaml
```

#### Step 3.3: Deploy MySQL StatefulSet
Define `mysql-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: wordpress-app
  labels:
    app: mysql
spec:
  replicas: 1
  serviceName: mysql
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: database
        image: mysql:5.7
        args:
          - "--ignore-db-dir=lost+found"
        envFrom:
          - secretRef:
              name: mysql-secret
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
      volumes:
          - name: mysql-data
            persistentVolumeClaim:
              claimName: mysql-pvc
```

Apply the StatefulSet:
```bash
kubectl apply -f mysql-statefulset.yaml
```

#### Step 3.4: Expose MySQL internally
Define `mysql-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: wordpress-app
spec:
  ports:
  - port: 3306
    protocol: TCP
  selector:
    app: mysql
```

Apply the service:
```bash
kubectl apply -f mysql-service.yaml
```

---

### 4. Deploy Application Tier (WordPress)

#### Step 4.1: Create Storage for WordPress
Define `wordpress-pv-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
  namespace: wordpress-app
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /home/ubuntu/project/wp-data 
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: wordpress-app
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: manual
```

Apply the storage resources:
```bash
kubectl apply -f wordpress-pv-pvc.yaml
```

#### Step 4.2: Create WordPress Deployment
Define `wordpress-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: wordpress-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - name: wordpress
          image: wordpress:5.8.3-php7.4-apache
          ports:
            - containerPort: 80
              name: wordpress
          volumeMounts:
            - name: wordpress-data
              mountPath: /var/www/html
          env:
            - name: WORDPRESS_DB_HOST
              value: mysql-service:3306
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: WORDPRESS_DB_USER
              value: root
            - name: WORDPRESS_DB_NAME
              value: wordpress
      volumes:
        - name: wordpress-data
          persistentVolumeClaim:
            claimName: wordpress-pvc
```

Apply the deployment:
```bash
kubectl apply -f wordpress-deployment.yaml
```

#### Step 4.3: Expose WordPress via LoadBalancer
Define `wordpress-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-service
  namespace: wordpress-app
  labels:
    app: wordpress
spec:
  type: LoadBalancer
  ports:
    - port: 80
  selector:
    app: wordpress
```

Apply the LoadBalancer service:
```bash
kubectl apply -f wordpress-service.yaml
```

---

## 🔍 Verification & Access

### Monitor Deployment Status

Track the rollout of resources in real time:

```bash
watch -d "kubectl get all -n wordpress-app"
```

### Retrieve External IP

Get the publicly accessible External IP generated by the LoadBalancer:

```bash
kubectl get svc -n wordpress-app | grep "wordpress-service" | awk '{ print $4 }'
```

### Complete Installation

Open your browser and navigate to:
```text
http://<EXTERNAL_IP>
```
Follow the on-screen WordPress wizard to complete your site configuration.

---

## 🛠 Useful Commands & Troubleshooting

### Retrieve MySQL Root Password from Secret
```bash
kubectl get secret mysql-secret -n wordpress-app -o jsonpath="{.data.MYSQL_ROOT_PASSWORD}" | base64 --decode
echo ""
```

### Access Database CLI Directly
Connect to the running MySQL pod directly via `kubectl`:
```bash
kubectl exec -it pod/mysql-0 -n wordpress-app -- mysql -u root -p
```

---

## 🧹 Cleanup

To delete all components and clean up resources created in the cluster, execute:

```bash
kubectl delete secret mysql-secret -n wordpress-app
kubectl delete deployment wordpress -n wordpress-app
kubectl delete statefulset mysql -n wordpress-app
kubectl delete pvc mysql-pvc wordpress-pvc -n wordpress-app
kubectl delete pv mysql-pv wordpress-pv -n wordpress-app
kubectl delete service mysql-service wordpress-service -n wordpress-app
kubectl delete namespace wordpress-app
```
