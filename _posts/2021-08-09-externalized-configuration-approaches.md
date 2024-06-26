---
layout: post
title: "Externalized Configuration Approaches"
date: 2021-08-09
excerpt: "Compare different approaches for externalizing app config"
tag:
- spring boot external properties file
- spring cloud config server
- kubernetes configmaps
- aws app config
- external configuration files
- externalize configuration
- twelve factor
- kubernetes configmap watcher
- configmaps
- aws app config
comments: true
---
## Spring Cloud Config Server vs Kubernetes Config Maps vs AWS App Config

We compare three approaches to externalize application configuration.

[1] Spring Cloud Config Server - provides server-side and client-side support for externalized configuration in a distributed system

[2] Kubernetes ConfigMaps - API object that lets you store configuration for other objects to use

[3] AWS App Config - supports controlled deployments to applications of any size and includes built-in validation checks and monitoring


| |Spring Cloud Config Server |Kubernetes ConfigMaps |AWS App Config|
|---|---|---|---|
|Config Format |	Plain text, yml, properties, json |	Configuration files, property keys |* YAML, JSON, or text documents in the AWS AppConfig hosted configuration store<br/><br/>* Objects in an Amazon Simple Storage Service (Amazon S3) bucket<br/><br/>* Documents in the Systems Manager document store<br/><br/>* Parameters in Parameter Store |
|Config Size Limit | NA| 1MB| * 4 to 8KB - Parameter Store<br/><br/>* 64KB - Document/AppConfig store<br/><br/>* 1MB - S3|
|Config Source| Git, Local, S3, JDBC database, Redis, Vault with Proxy Support| Cluster ConfigMaps| AppConfig Store, SSM Document Store, Parameter Store, S3|
|Immutable Config|For spring apps, using immutable configuration properties would prevent bean recreation |* v1.19 beta support<br/><br/>* ConfigMap and Pods need to be re-created |For spring apps, using immutable configuration properties would prevent bean recreation|
| Config Refresh|* Push/Poll Model<br/><br/>* Push Model - Github Webhooks, Spring Cloud Bus| * Approach 1: Use events api to watch for changes and reload beans/context (supported by spring-cloud-kubernetes) <br/><br/>* Approach 2: Restart the pods as part of deployment on config changes (Helm using sha256 checksum)<br/><br/>* Approach 3: Mounted ConfigMaps are automatically refreshed by Kubernetes. So, we need reload capability in our app upon change.| Poll model|
| Deployment Strategy|NA | * Possible to use rolling update strategy of deployments e.g. https://github.com/fabric8io/configmapcontroller<br/><br/>* Also, we can achieve config deployment rollout using Helm by checking for sha256 checksum| Linear, Exponential, AllAtOnce, Linear50PercentEvery30Seconds, Canary10Percent20Minutes|
| Sharing Config Across Apps| | | |
| Encryption & Decryption for Passwords||Need to use Secrets/Vault(supported by spring-cloud-kubernetes) |Secrets need to be stored in parameter store. No encryption/decryption feature available as part of config files |
| High Availability| * Multiple load balanced cloud config servers<br/><br/>* (Optional) Discovery Client - e.g. Eureka| EKS resiliency - Three etcd nodes that run across three Availability Zones within a Region| AWS managed|
|Advantages | REST API can be used by non-Spring apps| * Kubernetes native solution<br/><br/>* For Spring apps, spring-cloud-kubernetes helps to run spring apps in Kubernetes using native services| Validation checks, deployment strategy and rollbacks|
|Disadvantages |* Single point of failure without HA<br/><br/>* Additionally, we might need Discovery Client & Cloud Bus depending on the design | ConfigMaps consumed as environment variables are not updated automatically and require a pod restart| AWS managed service|
{:.table-striped}

{% include donate.html %}
{% include advertisement.html %}

## Spring Cloud Config Server

1. Supports pattern matching and multiple repositories for config 
2. Search paths feature to look for configs in sub directories
3. Supports shared configuration with all applications
4. Config refresh can be done using poll/push model
5. Push model, for example, requires Github webhook and Spring Cloud Bus to broadcast changes to matching applications
6. By default, the server clones remote repositories when configuration is first requested. This can be changed by using clone first setting.
7. Proxy support is also available
8. Configs can be accessed from multiple backends such as local, Git, S3 etc.

### Pros

1. Non spring apps can use the REST API to get the config

### Cons

1. Single point of failure in the absence of HA design - If cloud server is down then the pods won't get started
2. Making cloud server highly available requires service discovery feature - either built-in Kubernetes service discovery or Eureka
3. Config refresh using push model from Github requires Spring Cloud Bus so that we can refresh all the shared apps

{% include donate.html %}
{% include advertisement.html %}

## Kubernetes Config Maps

1. Spring apps can use `spring-cloud-kubernetes` starter to simplify access to configmaps and secrets
2. Supports hot reloading of beans/context when configmap changes
3. Define configmap name and namespace
4. We can have a Single `application.yaml` config map with multiple profile properties or config map per profile
5. Config changes are detected by watching events API or polling configured via settings (deprecated) - new approach is to deploy Kubernetes Configuration Watcher
6. Requires RBAC permissions to watch config changes

### Pros

1. Kubernetes native solution to which devops are accustomed
2. Can leverage Kubernetes tools
3. EKS resiliency means etcd is HA with 3 nodes

### Cons

1. Config size can't exceed 1 MB - otherwise consider mounting a volume or use a separate database or file service

{% include donate.html %}
{% include advertisement.html %}


## AWS App Config

1. Validator has syntactic and semantic checks using schema or lambda function

2. Configuration can be stored in parameter store, ssm document, app config store or s3

3. Supports deployment strategies such as growth factor, final bake time in mins, all-in-one,  canary

### Configuration Store Options

Here we will compare the different configuration stores available with AWS AppConfig.

| |AWS AppConfig hosted configuration store| S3|Parameter Store| Document store| 
|---|---|---|---|---|
|Configuration size limit | 64 KB|* 1 MB<br/><br/>* Enforced by AWS AppConfig, not S3| 4 KB (free tier) / 8 KB (advanced parameters)| 64 KB|
|Resource storage limit | 	1 GB| Unlimited| 10,000 parameters (free tier) / 100,000 parameters (advanced parameters)| 500 documents|
|Server-side encryption | Yes| No| No| No|
|AWS CloudFormation support |Yes |Not for creating or updating data |Yes |No |
|Validate create or update API actions| Not supported| Not supported|Regex supported | JSON Schema required for all put and update API actions|
|Pricing | Free|See Amazon S3 pricing| See AWS Systems Manager pricing|Free|
{:.table-striped}

### Pros
 
1. Support for validation checks and monitoring 
2. Configuration storage support in variety of AWS services like S3, SSM, App config store etc.

### Cons

1. Not cloud agnostic

{% include donate.html %}
{% include advertisement.html %}

