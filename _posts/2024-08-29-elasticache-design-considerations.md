---
layout: post
title:  "AWS ElastiCache Cluster Design Considerations"
date:   2024-08-29
excerpt: "Optimize your AWS ElastiCache setup with best practices for architecture, security, and performance tuning"
tag:
- AWS ElastiCache Best Practices
- Redis Cluster Design
- AWS Redis Architecture
- Data Caching Strategies
- AWS In-Memory Caching
comments: true
---

# AWS ElastiCache Cluster Design Considerations

## Introduction

AWS ElastiCache offers in-memory caching services that help accelerate data access for applications requiring microsecond read and write performance. This service is particularly useful for scaling out database-intensive workloads and enhancing overall application performance.

## Cluster Mode Disabled vs. Cluster Mode Enabled

**Cluster Mode Disabled:**

- **Sharding:** One shard (node group) with up to six nodes (1 read/write node, others as read replicas).
- **Endpoints:**
  - **Single Node:** A single endpoint handles both reads and writes.
  - **Multi Node:** Separate endpoints for primary and reader nodes.

**Cluster Mode Enabled:**

- **Sharding:** Supports up to 500 shards (each with primary and replica nodes).
- **Endpoints:** Provides a single endpoint to discover primary and read endpoints for each shard.
- **Data Seeding:** When creating a new cluster, you can seed it with data from an old cluster, provided the shard count remains the same. This is useful when changing node types or engine versions.

## Design Options

**Option 1: Multiple DB Instances within Same Redis (Not Recommended)**

- **DB Instance for each App:** Using DB instances for each app within same Redis cluster is not scalable since Redis is single-threaded.
- **Cons:**
  - Cross Data Access limitations.
  - Eviction policies per DB are not feasible.

**Option 2: Key Prefixing**

- **Key Management:** Utilize a key prefix to segregate data per application within a single Redis instance.
- **Pros:** Cost-effective, co-located data.
- **Cons:** Limited eviction policy flexibility.

**Option 3: Separate Instances**

- **Data Isolation:** Each application has its own Redis instance.
- **Pros:** Better scalability, unique eviction policies.
- **Cons:** Higher costs.

<figure>
    <a href="{{ site.url }}/assets/img/2024/08/elasticache-design-considerations.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2024/08/elasticache-design-considerations.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2024/08/elasticache-design-considerations.png">
            <img src="{{ site.url }}/assets/img/2024/08/elasticache-design-considerations.png" alt="">
        </picture>
    </a>
</figure>


{% include donate.html %}
{% include advertisement.html %}

## Eviction Policies

Different eviction policies are available to manage memory usage:
- **noeviction:** No new values are saved when memory is full.
- **allkeys-lru:** Least Recently Used keys are evicted.
- **volatile-lru:** LRU eviction for keys with an expiration field.
- **volatile-ttl:** Evicts keys with the least time-to-live (TTL) remaining.
- (Refer to Redis eviction policy documentation for more details.)

## Data Tiering and Node Selection

- **r6gd Instance Family:** 20% of data remains in-memory for regularly accessed items.
- **Recommendation:** Key sizes should generally be smaller than value sizes when using data tiering.

## Multi-AZ and Multi-Region Considerations

- **Multi-AZ:** At least two nodes with replicas are recommended for failover and high availability.
- **Multi-Region:**
  - **Global Datastore:**
    - **Primary Cluster:** Accepts writes replicated across all clusters within the global datastore.
    - **Secondary Cluster:** Only accepts read requests and replicates data from the primary.
    - Note: ElastiCache does not support autofailover across regions; promotion of a secondary cluster is manual.

## High Throughput and Enhanced Performance

- **Enhanced I/O Multiplexing:** Optimizes performance for high-throughput scenarios.
- **Sharded Pub/Sub:** Implement sharded Pub/Sub using SSUBSCRIBE, SUNSUBSCRIBE, and SPUBLISH.

{% include donate.html %}
{% include advertisement.html %}

## Security and Monitoring

- **Security Group Configuration:** Use `AuthorizeCacheSecurityGroupIngress` to authorize EC2 security groups for cluster access.
- **Monitoring:**
  - Consider Redis's single-threaded nature when evaluating CPU usage.
  - For example, a four-core CPU with 20% reported usage implies that the single core Redis is using is at 80% utilization.

## Connectivity and Management

- **TLS Connectivity:** Build Redis distribution with TLS support using:
  ```bash
  yum install -y openssl-devel
  make BUILD_TLS=yes
  ```
- To restart the build process after initial issues, use:
  ```bash
  make distclean
  ```
- Example command to connect securely:
  ```bash
  src/redis-cli -c -h master.redis.jske8798.use1.cache.amazonaws.com --tls --askpass
  ```

## Conclusion
When designing your ElastiCache cluster, carefully consider the architecture, whether to use cluster mode, eviction policies, and the specifics of your use case. Properly configured, ElastiCache can significantly accelerate data access and improve your application's performance.

{% include donate.html %}
{% include advertisement.html %}