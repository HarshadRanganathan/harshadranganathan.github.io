---
layout: post
title:  "AWS Cost Optimization - CloudWatch"
date:   2024-08-29
excerpt: "Optimize your AWS CloudWatch costs by managing log retention settings and leveraging the CloudWatch Logs Infrequent Access log class"
tag:
- AWS Cost Optimization
- CloudWatch Logs
- AWS CloudWatch
- AWS Logging
- Cloud Cost Management
- Log Retention
- AWS Best Practices
- Log Class
- Infrequent Access
comments: true
---

# Cost Optimization Tips for AWS CloudWatch 🛠️

## Log Retention Settings

### Setting Log Retention Periods to Save Costs 💰

By default, log data in AWS CloudWatch Logs is stored indefinitely, which can lead to unnecessary costs over time. To optimize costs, it's crucial to set a retention period for all your logs. 

### Why Set a Retention Period? ⏳

- **Cost Efficiency**: Storing logs indefinitely can quickly become expensive. Setting a retention period helps manage and reduce storage costs.
- **Compliance**: Different log groups may have different compliance needs. For instance, audit logs from services like RDS might require a longer retention period, while other logs can be kept for just a month.

### How to Set Log Retention Periods 📝

Follow these steps to change the retention settings for your logs:

1. **Open the CloudWatch Console**: Go to [AWS CloudWatch Console](https://console.aws.amazon.com/cloudwatch/).

2. **Navigate to Log Groups**:
   - In the navigation pane, choose **Logs**, then **Log groups**.

3. **Select the Log Group**:
   - Find the log group you want to update.

4. **Change the Retention Setting**:
   - In the **Retention** column for that log group, click on the current retention setting (e.g., **Never Expire**).
   - In the **Retention setting** menu, choose a log retention value (e.g., **1 month**).
   - Click **Save**.

### Benefits of Optimizing Log Retention 🏆

- **Reduce Storage Costs**: By deleting expired log events automatically, you can significantly lower your CloudWatch storage expenses.

- **Maintain Compliance**: Tailor the retention period based on your organization's compliance requirements, ensuring that critical logs are retained for as long as necessary.

By configuring these settings, you can achieve a more cost-effective and efficient logging strategy in AWS CloudWatch.

{% include donate.html %}
{% include advertisement.html %}

## Log Class

### Leverage Log Class for Cost Reduction 💰

One of the biggest cost contributors in CloudWatch is not the storage but the ingestion of logs. CloudWatch Logs are indexed for near real-time querying, which comes at a premium price. However, not all logs require such capabilities. 

AWS offers two classes of log groups to help manage these costs:

1. **Standard Log Class:**
   - **Use Case:** Real-time monitoring and frequently accessed logs.
   - **Features:** Full set of capabilities including live tailing, metric extraction, and alarming.

2. **Infrequent Access Log Class:**
   - **Use Case:** Logs that are infrequently accessed, ideal for ad-hoc querying and forensic analysis.
   - **Cost Savings:** 50% lower ingestion cost per GB compared to the Standard log class.
   - **Features:** Limited to essential capabilities like managed ingestion, storage, and cross-account log analytics.


**Implementation Tip:** Set the log class to "Infrequent Access" for logs that do not require real-time processing to significantly reduce your ingestion costs.

**Note:** The Infrequent Access Log Class is not supported for all managed service integrations (e.g., RDS logs), so make sure to apply it where available.

<figure>
    <a href="{{ site.url }}/assets/img/2024/08/create-log-group.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2024/08/create-log-group.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2024/08/create-log-group.png">
            <img src="{{ site.url }}/assets/img/2024/08/create-log-group.png" alt="">
        </picture>
    </a>
</figure>


{% include donate.html %}
{% include advertisement.html %}

## Identifying Cost Contributors in CloudWatch Log Groups

Understanding the biggest cost contributors within AWS CloudWatch Log Groups can be challenging, as AWS Cost Explorer does not provide a detailed breakdown of costs per log group. This section explains how to get those details using the Cost and Usage Report (CUR) and the CUDOS dashboard.

### Setting Up CUR and CUDOS for Cost Analysis 📊

To identify the major cost contributors in CloudWatch Log Groups, follow these steps:

Detailed setup guide - [https://rharshad.com/aws-cost-optimizations-cost-intelligence-dashboards/](https://rharshad.com/aws-cost-optimizations-cost-intelligence-dashboards/)

1. **Enable Cost and Usage Report (CUR):**
   - Navigate to the AWS Billing Console.
   - In the navigation pane, select "Cost & Usage Reports."
   - Click on "Create report," and set up your CUR to track detailed cost and usage information.

2. **Build a CUDOS Dashboard:**
   - Once CUR is enabled, you can use AWS's Cost and Usage Data Optimization Services (CUDOS) to create a dashboard.
   - The CUDOS dashboard will give you a detailed breakdown of costs, including individual log groups in CloudWatch.

3. **Analyze Log Group Costs:**
   - With the CUDOS dashboard, identify which log groups are contributing the most to your costs.
   - Use this information to take necessary cost optimization actions, such as:
     - **Deleting unnecessary log groups.**
     - **Setting appropriate retention periods.**
     - **Adjusting log classes for lower costs.**

<figure>
    <a href="{{ site.url }}/assets/img/2024/08/log-group-cudos-cost-insights.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2024/08/log-group-cudos-cost-insights.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2024/08/log-group-cudos-cost-insights.png">
            <img src="{{ site.url }}/assets/img/2024/08/log-group-cudos-cost-insights.png" alt="">
        </picture>
    </a>
</figure>

{% include donate.html %}
{% include advertisement.html %}