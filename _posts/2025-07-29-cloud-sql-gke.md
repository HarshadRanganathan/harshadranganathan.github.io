---
layout: post
title:  "Securely connect your GKE applications to private Cloud SQL"
date:   2025-07-29
excerpt: "Learn how to securely connect to your private Cloud SQL instance from GKE pods using the Cloud SQL Proxy and Workload Identity"
tag:
- Connect GKE to Cloud SQL Private IP
- GCP Cloud SQL Proxy Setup
- Secure MySQL Access from Kubernetes
- GKE Cloud SQL Private Connection
- Workload Identity Cloud SQL GKE
comments: true
---

## Connect to a Private GCP Cloud SQL Instance from GKE Using Cloud SQL Proxy

### Introduction

When Cloud SQL is deployed with a **private IP**, it cannot be accessed publicly. To enable your GKE pods to connect securely, we use **Cloud SQL Proxy** as a sidecar container. This guide walks you through setting up Workload Identity, provisioning credentials, and using the proxy for secure internal DB connectivity.


### Architecture Overview

* GKE → Private GCP VPC
* Cloud SQL (MySQL/Postgres) → Private IP only
* Cloud SQL Proxy runs as an **init container** in your application pod
* GKE workload identity + IAM permissions enable secure proxy connection


### Step 1: Grant IAM Permissions to GKE Service Account

If you're using **Workload Identity Federation**, ensure your GKE service account has the following IAM role:

```text
roles/cloudsql.client
```

This allows the proxy sidecar to initiate a secure connection to the Cloud SQL instance.


### Step 2: Create Secret for Database Credentials

You need to pass DB credentials (username, password, DB name) as environment variables:

```bash
kubectl create secret generic cloud-sql-secrets \
  --from-literal=database=test \
  --from-literal=username=<your-db-user> \
  --from-literal=password=<your-db-password>
```


### Step 3: Add Cloud SQL Proxy Sidecar to Deployment

In your app’s `deployment.yaml`, define an **init container** to start the proxy:

```yaml
initContainers:
  - name: cloud-sql-proxy
    image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.14.1 # ensure you keep this updated regularly
    args:
      - "--private-ip"
      - "--structured-logs"
      - "--port=3306"
      - "<CLOUDSQL_CONNECTION_NAME>" # e.g. project:region:instance
    securityContext:
      runAsNonRoot: true
    resources:
      requests:
        memory: "2Gi"
        cpu: "1"
```

📌 **Note**:

* `--private-ip` ensures the proxy connects via internal IP.
* Replace `<CLOUDSQL_CONNECTION_NAME>` with your actual connection string.

{% include donate.html %}
{% include advertisement.html %}

### Step 4: Inject Environment Variables into Main Container

In the main container spec, load DB credentials and point to the proxy's localhost endpoint:

```yaml
containers:
  - name: app
    env:
      - name: DB_USER
        valueFrom:
          secretKeyRef:
            name: cloud-sql-secrets
            key: username
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: cloud-sql-secrets
            key: password
      - name: DB_NAME
        valueFrom:
          secretKeyRef:
            name: cloud-sql-secrets
            key: database
      - name: DB_IP
        value: "127.0.0.1"
```

### Step 5: Use Env Vars in Application Code

Your app will connect to MySQL via the proxy on `127.0.0.1:3306` using provided credentials:

**Python Example (SQLAlchemy + PyMySQL):**

```python
import os

DB_NAME = os.getenv("DB_NAME")
DB_USER = os.getenv("DB_USER")
DB_PASSWORD = os.getenv("DB_PASSWORD")
DB_IP = os.getenv("DB_IP")
DB_PORT = "3306"

def get_google_cloud_sql_db_url():
    return f"mysql+pymysql://{DB_USER}:{DB_PASSWORD}@{DB_IP}:{DB_PORT}/{DB_NAME}"
```

### Troubleshooting: Debug/Verify Connectivity from a GKE Pod

If you're facing connection issues, deploy a test pod to debug:

```yaml
# cloudsql-mysql-tester.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudsql-mysql-tester
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudsql-mysql-tester
  template:
    metadata:
      labels:
        app: cloudsql-mysql-tester
  spec:
    containers:
      - name: mysql-client
        image: mysql/mysql-server:latest
        command: ["tail", "-f", "/dev/null"]
        resources:
          limits:
            cpu: "100m"
            memory: "128Mi"
```

Access the pod and try connecting manually:

```bash
POD_NAME=$(kubectl get pods -l app=cloudsql-mysql-tester -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME -- bash
mysql -u root --host 10.x.x.x --port 3306 -p
```

### Troubleshooting: Test from a VM 

```bash
sudo apt-get update && sudo apt-get -y install mysql-server
mysql -u root --host 10.x.x.x --port 3306 -p
```

### Summary

| Component         | Purpose                             |
| ----------------- | ----------------------------------- |
| Workload Identity | Secure authentication for the proxy |
| Secret            | Stores DB credentials               |
| Cloud SQL Proxy   | Establishes secure internal tunnel  |
| Init Container    | Runs the proxy before app starts    |
| Env Variables     | Feed DB info to your application    |

{% include donate.html %}
{% include advertisement.html %}