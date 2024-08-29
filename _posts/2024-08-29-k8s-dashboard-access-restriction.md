---
layout: post
title:  "Restricting Kubernetes Dashboard Access for Developers"
date:   2024-08-29
excerpt: "Securely empower your developers on Kubernetes with restricted access to the dashboard, balancing control and productivity while safeguarding your cluster"
tag:
- Kubernetes Dashboard
- Kubernetes Access Control
- RBAC Kubernetes
- Kubernetes Security
- Kubernetes Permissions
- Kubernetes Service Account
- Kubernetes ClusterRole
- Kubernetes Best Practices
comments: true
---

# Restricting Kubernetes Dashboard Access for Developers

To implement restricted access to the Kubernetes Dashboard for developers, we will create a custom `ClusterRole` that allows developers to perform specific actions such as port forwarding, pod execution, and connecting to the dashboard, while preventing destructive actions like deleting pods or scaling deployments. We will then bind this `ClusterRole` to a service account that developers will use to log in.

## Step-by-Step Guide

### 1. Create a Service Account in the `kubernetes-dashboard` Namespace

First, create a new service account named `developer` in the `kubernetes-dashboard` namespace. This service account will be used by developers to access the dashboard.

```bash
kubectl create serviceaccount developer -n kubernetes-dashboard
```

> **Note:** This service account can be binded with an IAM role for cluster access, allowing for further integration with AWS IAM policies/SSO login.

### 2. Define the `ClusterRole` with Restricted Permissions

Create a `ClusterRole` named `developer` with the following permissions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
rules:
  - verbs:
      - get
      - list
      - watch
    apiGroups:
      - '*'
    resources:
      - '*'
  - verbs:
      - '*'
    apiGroups:
      - '*'
    resources:
      - services/proxy
  - verbs:
      - '*'
    apiGroups:
      - '*'
    resources:
      - pods/portforward
  - verbs:
      - create
    apiGroups:
      - '*'
    resources:
      - pods/exec
  - verbs:
      - create
    apiGroups:
      - '*'
    resources:
      - serviceaccounts/token
    resourceNames:
      - developer
  - verbs:
      - patch
    apiGroups:
      - apps
    resources:
      - deployments/scale
  - verbs:
      - delete
    apiGroups:
      - '*'
    resources:
      - pods
```

#### Explanation of Permissions:

- **Read Permissions (get, list, watch)**: These are the basic read-only permissions that allow developers to view resources across all API groups.
  
- **`services/proxy`**: Allows developers to proxy services, which is essential for accessing the Kubernetes Dashboard through a secure proxy.

- **`pods/portforward`**: Enables port forwarding to pods, allowing developers to interact with services running inside the cluster without exposing them externally.

- **`pods/exec`**: Allows the execution of commands inside a pod, which is necessary for troubleshooting and running diagnostics.

- **`serviceaccounts/token`**: Grants permission to create tokens, specifically for the `developer` service account, to facilitate dashboard login.

- **`deployments/scale` (patch)**: Permits scaling of deployments, which can be necessary for testing and development purposes.

- **Restricting Destructive Actions**:
  - **`pods (delete)`**: Only allows deletion of pods, which might be needed to restart a malfunctioning pod, while preventing more critical destructive actions.

{% include donate.html %}
{% include advertisement.html %}

### 3. Bind the `ClusterRole` to the Service Account

Next, create a `ClusterRoleBinding` to associate the `developer` service account with the `developer` `ClusterRole`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developer-dashboard-binding
subjects:
  - kind: ServiceAccount
    name: developer
    namespace: kubernetes-dashboard
roleRef:
  kind: ClusterRole
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Apply the `ClusterRoleBinding`:

```bash
kubectl apply -f developer-dashboard-binding.yaml
```

### 4. Generate a Token for Dashboard Login

Finally, generate a token that the developer can use to log in to the Kubernetes Dashboard:

```bash
kubectl create token developer -n kubernetes-dashboard
```

Use this token to authenticate to the Kubernetes Dashboard.

### Summary

By following these steps, you can ensure that developers have the necessary permissions to interact with the Kubernetes Dashboard and the cluster resources without granting them the ability to perform destructive actions. This setup strikes a balance between enabling productive work and maintaining the security and stability of your Kubernetes environment.

{% include donate.html %}
{% include advertisement.html %}