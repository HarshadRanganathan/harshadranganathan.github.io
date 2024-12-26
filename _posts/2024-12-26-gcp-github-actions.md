---
layout: post
title:  "Deploying to Google Cloud Run with GitHub Actions and Workload Identity Federation"
date:   2024-12-26
excerpt: "Securely deploy to Google Cloud Run from GitHub Actions using Workload Identity Federation, eliminating the need for service account keys"
tag:
- GitHub Actions Cloud Run
- Deploy to GCP from GitHub
- Workload Identity Federation GCP
- Secure CI/CD GCP
- GitHub Actions GCP Authentication
comments: true
---

## Deploying to Google Cloud Run with GitHub Actions and Workload Identity Federation

### Introduction
GitHub Actions provide a powerful CI/CD pipeline for automating application deployments. In this guide, we'll explore how to securely deploy applications to Google Cloud Run by setting up Workload Identity Federation between GitHub and GCP. This allows GitHub Actions to authenticate with GCP without needing long-lived service account keys.

---

### Step 1: Create a Workload Identity Pool
A **Workload Identity Pool** allows external identities (e.g., GitHub Actions) to authenticate with Google Cloud resources.

Run the following command to create a workload identity pool named `github`:

```bash
gcloud iam workload-identity-pools create "github" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --display-name="GitHub Actions Pool"
```

---

### Step 2: Create a Provider for the Identity Pool
Next, create an OIDC (OpenID Connect) provider under the workload identity pool. This provider will be scoped to the GitHub organization level, meaning any repository in the organization can interact with GCP resources. You can also scope it to a specific repository if needed.

```bash
gcloud iam workload-identity-pools providers create-oidc "org" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="github" \
  --display-name="Org Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner" \
  --attribute-condition="assertion.repository_owner == '${ORG_NAME}'" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

#### Key Options:
1. **Attribute Mapping:** This translates claims in the GitHub Actions OIDC token to attributes usable in IAM policy bindings.
   - Example: `google.subject=assertion.sub` maps the OIDC subject claim to the `google.subject` attribute.
2. **Attribute Condition:** Ensures that only requests from a specific GitHub organization (`${ORG_NAME}`) are allowed.

---

{% include donate.html %}
{% include advertisement.html %}

### Step 3: Create and Bind a Service Account
Now, create a **Service Account** that GitHub Actions will use to authenticate with GCP.

1. **Create the Service Account:**
   ```bash
   gcloud iam service-accounts create "github-deploy" \
     --project="${PROJECT_ID}" \
     --display-name="GitHub Deployment Service Account"
   ```

2. **Bind the Provider to the Service Account:**
   ```bash
   gcloud iam service-accounts add-iam-policy-binding "${SERVICE_ACCOUNT_EMAIL}" \
     --project="${PROJECT_ID}" \
     --role="roles/iam.workloadIdentityUser" \
     --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github/attribute.repository_owner/${ORG_NAME}"
   ```

---

### Step 4: Assign Necessary Permissions
Grant the service account permissions required to deploy to Cloud Run and interact with GCP services. Use the following roles:

- **Service Account User**
- **Storage Admin**
- **Artifact Registry Administrator**
- **Cloud Build Editor**
- **Cloud Run Admin**

```bash
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
  --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" \
  --role="roles/iam.serviceAccountUser"

# Repeat the above command for other roles as needed
```

These roles ensure the service account has permissions to:
- Build container images (via Cloud Build).
- Push artifacts to Artifact Registry or storage buckets.
- Deploy services to Cloud Run.

---

{% include donate.html %}
{% include advertisement.html %}

### Step 5: Configure GitHub Actions Workflow
In your GitHub Actions workflow file, use the Google GitHub Actions library to authenticate with GCP.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - id: 'auth'
        name: 'Authenticate to Google Cloud'
        uses: 'google-github-actions/auth@v2'
        with:
          workload_identity_provider: 'projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github/providers/${ORG_NAME}' # Replace with your identity provider
          service_account: '${SERVICE_ACCOUNT_EMAIL}' # Replace with your service account email

      - id: 'deploy'
        name: 'Deploy to Cloud Run'
        run: |
          gcloud run deploy my-service \
            --image=gcr.io/${PROJECT_ID}/my-image:latest \
            --platform=managed \
            --region=us-central1
```

---

### Best Practices
1. **Scope Providers Narrowly:** Scope providers to the smallest necessary scope (e.g., a specific repository rather than an entire organization) to enhance security.
2. **Avoid Service Account Keys:** Workload Identity Federation eliminates the need for service account keys, reducing the risk of key leakage.
3. **Monitor Usage:** Use Cloud Monitoring to track service account activity and ensure compliance.

---

### Conclusion
By integrating GitHub Actions with GCP using Workload Identity Federation, you can achieve secure and efficient CI/CD pipelines for deploying applications to Cloud Run. This setup avoids the risks associated with service account keys while providing the flexibility to scale across multiple repositories or organizations.