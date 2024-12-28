---
layout: post
title:  "Securing Your React App on Cloud Run for Internal Authenticated Users"
date:   2024-12-28
excerpt: "Learn how to secure your React app on Cloud Run by restricting access to authenticated GCP users using IAP and a Load Balancer"
tag:
- Secure Cloud Run React App
- Google Cloud IAP for Authentication
- Cloud Run Load Balancer Setup
- React App GCP Authentication
- Exposing Internal GCP Apps
comments: true
---

## Securing Your React App on Cloud Run for Internal Authenticated Users

### Introduction
Hosting a React app on Google Cloud Run is simple and efficient. However, restricting access to internal, authenticated users (such as GCP groups) requires additional configuration. In this guide, we will walk through setting up Cloud Run, a Serverless NEG, a Load Balancer, and enabling IAP to secure the React app.


### Prerequisites
1. A deployed React app on Cloud Run.
2. A GCP project with billing enabled.
3. A custom domain (optional but recommended for user-friendly URLs).
4. Basic knowledge of GCP networking and IAM.


### Step 1: Secure the Cloud Run Service
Set the authentication mode of your Cloud Run service to **"Require authentication"**:
1. Navigate to your Cloud Run service in the GCP console.
2. Under the **Security** tab, set **Authentication** to **"Require authentication (Manage authorized users with Cloud IAM)"**.
3. Save the changes.

This ensures that the service cannot be accessed publicly without authentication.

<figure>
    <a href="{{ site.url }}/assets/img/2024/12/gcp-cloud-run-auth.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2024/12/gcp-cloud-run-auth.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2024/12/gcp-cloud-run-auth.png">
            <img src="{{ site.url }}/assets/img/2024/12/gcp-cloud-run-auth.png" alt="">
        </picture>
    </a>
</figure>


### Step 2: Create a Serverless NEG
A **Serverless Network Endpoint Group (NEG)** allows integration of Cloud Run with Google Cloud Load Balancer.

Run the following command to create a NEG for your React app:

```bash
gcloud compute network-endpoint-groups create neg-react \
  --region=us-central1 \
  --network-endpoint-type=serverless \
  --cloud-run-service=react-app
```

Replace `react-app` with the name of your Cloud Run service.

### Step 3: Set Up a Proxy-Only Subnet for VPC
To configure a load balancer, you need a **proxy-only subnet** for your VPC:

1. In the GCP console, navigate to **VPC Network > Subnets**.
2. Click **Add Subnet**.
3. Provide a name and choose a region.
4. Set **Purpose** to **Regional Managed Proxy**.
5. Save the changes.

<figure>
    <a href="{{ site.url }}/assets/img/2024/12/gcp-proxy-only-subnet.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2024/12/gcp-proxy-only-subnet.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2024/12/gcp-proxy-only-subnet.png">
            <img src="{{ site.url }}/assets/img/2024/12/gcp-proxy-only-subnet.png" alt="">
        </picture>
    </a>
</figure>

### Step 4: Register a Domain (Optional)
If you don’t already have a domain, register one via **Cloud Domains**:
1. Navigate to **Cloud Domains** in the GCP console.
2. Register a new domain and complete the verification process.
3. Use **Cloud DNS** to manage your domain’s DNS configuration.

{% include donate.html %}
{% include advertisement.html %}

### Step 5: Set Up the Load Balancer
Configure a global, public-facing load balancer:

#### 5.1: Frontend Configuration
1. Choose **Application Load Balancer**.
2. Set it to **Public Facing** with **Global Deployment**.
3. Configure HTTPS as the protocol.
4. For the SSL certificate, create a **Google-managed certificate** for your domain (e.g., `react.example.com`).

#### 5.2: Backend Configuration
1. Create a backend service and choose **Serverless NEG** as the backend type.
2. Select the NEG created earlier (`neg-react`).

#### 5.3: Finalize and Review
Once the frontend and backend are configured, review and create the load balancer. This will generate a public IP for your application.

<figure>
    <a href="{{ site.url }}/assets/img/2024/12/gcp-lb-configuration.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2024/12/gcp-lb-configuration.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2024/12/gcp-lb-configuration.png">
            <img src="{{ site.url }}/assets/img/2024/12/gcp-lb-configuration.png" alt="">
        </picture>
    </a>
</figure>

### Step 6: Configure DNS
To map your domain to the load balancer:
1. Navigate to **Cloud DNS > Zones**.
2. Create a public DNS zone for your domain.
3. Add an **A Record** pointing to the public IP of the load balancer.
4. Example: `react.example.com -> [LOAD_BALANCER_IP]`.

{% include donate.html %}
{% include advertisement.html %}

### Step 7: Enable Identity-Aware Proxy (IAP)
IAP provides an additional security layer by authenticating users before allowing access.

1. Enable the **IAP API**:
   ```bash
   gcloud services enable iap.googleapis.com
   ```

2. Create an IAP service account:
   ```bash
   gcloud beta services identity create --service=iap.googleapis.com --project=${PROJECT_ID}
   ```

3. Grant the IAP service account the **Cloud Run Invoker** role:
   ```bash
   gcloud projects add-iam-policy-binding ${PROJECT_ID} \
     --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-iap.iam.gserviceaccount.com" \
     --role="roles/run.invoker"
   ```

### Step 8: Configure User Access
To restrict access to a specific GCP group:
1. Go to **Identity-Aware Proxy** in the GCP console.
2. Under **Applications**, find your backend service.
3. Toggle IAP to **Enabled**.
4. Add the GCP group as a principal:
   - Set the role to **Cloud IAP > IAP-secured Web App User**.

Users in this group will need to sign in with their GCP credentials to access the app.

### Step 9: Test Your Setup
1. Access the React app URL (e.g., `https://react.example.com`).
2. You will be prompted to sign in with your Google account.
3. Only authenticated users in the specified GCP group will be granted access.

### Summary
By integrating Cloud Run, a Serverless NEG, a Load Balancer, and IAP, you can securely expose your React app to internal users authenticated through GCP. This approach provides robust security, user-friendly URLs, and seamless access management for internal teams.

{% include donate.html %}
{% include advertisement.html %}