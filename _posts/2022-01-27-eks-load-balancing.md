---
layout: post
title: "Load Balancing in EKS"
date: 2024-03-29
excerpt: "Steps to expose services through a load balancer in EKS"
tag:
- aws
- eks
- aws load balancer controller example
- aws load balancer controller ingress example
- eksaws load balancer controller helm chart
- aws load balancer ingress controller
- eks internal load balancer
- eks load balancer annotations
- kubernetes aws load balancer annotations
- eks ingress
- aws load balancer controller
- network load balancing on amazon eks
- application load balancing on amazon eks
- provide external access to kubernetes services
comments: true
---

## Introduction

AWS Load Balancer Controller is a controller to help manage Elastic Load Balancers for a Kubernetes cluster.

- It satisfies Kubernetes Ingress resources by provisioning Application Load Balancers.

- It satisfies Kubernetes Service resources by provisioning Network Load Balancers.

## Installation

You can install aws load balancer controller by following the instructions in below repo:

Pre-requisites -

- AWS LoadBalancer Controller >= v2.4.0
- Kubernetes >= v1.22
- Pods have native AWS VPC networking configured

{% include repo-card.html repo="helm-aws-load-balancer-controller" %}

## Application Load Balancer

<figure>
    <a href="{{ site.url }}/assets/img/2024/03/eks-alb.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2024/03/eks-alb.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2024/03/eks-alb.png">
            <img src="{{ site.url }}/assets/img/2024/03/eks-alb.png" alt="">
        </picture>
    </a>
</figure>

{% include donate.html %}
{% include advertisement.html %}

### How To Provision

To provision an Application Load Balancer, you need to create an Ingress resource with ALB listener rules & annotations.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "App"
  namespace: "application"
  labels:
    app.kubernetes.io/name: App
spec:
  ingressClassName: alb
  rules:
    - host: test.example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: app
                port:
                  number: 80
```

Above creates an ALB with target groups pointing to the EC2 instance Node Ports and listener rules as follows:

|Conditions |Actions |
|--|--|
|**HTTP Host Header** is `test.example.com` AND **Path Pattern** is `/*` | Forward to target group belonging to EKS Worker nodes|
{:.table-striped}

You also need to create a service of type `NodePort` to which the ALB will route the traffic using the Ingress rules defined above.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: "App"
  namespace: "application"
  labels:
    app.kubernetes.io/name: App
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
  selector:
    app.kubernetes.io/name: App
```

### Subnet Discovery

AWS Load Balancer Controller auto discovers network subnets by default.

To be able to successfully do that you need to tag your subnets as follows:

|Tag Key |Tag Value |Purpose|
|--|--|--|
|kubernetes.io/role/elb |1 |Indicates that the subnet is public. Will be used if ALB is internet-facing |
|kubernetes.io/role/internal-elb |1 |Indicates that the subnet is private. Will be used if ALB is internal |
{:.table-striped}

### Instance Mode

Instance target mode supports pods running on AWS EC2 instances. In this mode, AWS NLB sends traffic to the instances and the kube-proxy on the individual worker nodes forward it to the pods through one or more worker nodes in the Kubernetes cluster.

### IP Mode

IP target mode supports pods running on AWS EC2 instances and AWS Fargate. In this mode, the AWS NLB targets traffic directly to the Kubernetes pods behind the service, eliminating the need for an extra network hop through the worker nodes in the Kubernetes cluster.

{% include donate.html %}
{% include advertisement.html %}

### Annotations

Let's look at some of the annotations that you can configure and their behaviors.

|Annotation Example |Purpose |
|--|--|
|alb.ingress.kubernetes.io/scheme: internet-facing | specifies whether your LoadBalancer will be internet facing|
|alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]' | specifies the LoadBalancer to listen on port 443 |
|alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:... | specifies the ARN of one or more certificate managed by AWS Certificate Manager |
|alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06 | Security Policy that should be assigned to the ALB |
|alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:us-east-1:...	 | specifies ARN for the Amazon WAFv2 web ACL.|
|alb.ingress.kubernetes.io/target-type: instance | Provision ALB in Instance mode |
|alb.ingress.kubernetes.io/load-balancer-attributes: <br/>access_logs.s3.enabled=true,access_logs.s3.bucket=prod-logs-us-east-1,access_logs.s3.prefix=alb,idle_timeout.timeout_seconds=600 | specifies Load Balancer Attributes that should be applied to the ALB. |
|alb.ingress.kubernetes.io/security-groups: sg-xxxxx |custom securityGroups you want to attach to ALB instead of ALB controller managed SG|
|alb.ingress.kubernetes.io/inbound-cidrs: "192.0.0.0/16,193.0.0.0/32" |CIDR ranges which will be added to the ALB controller managed SG attached to the LB|
|alb.ingress.kubernetes.io/manage-backend-security-group-rules: true |set to 'true' so that ALB controller will manage adding rules to allow traffic from LB to Target Nodes |
|alb.ingress.kubernetes.io/healthcheck-port: status-port	 | specifies the named port/port number used when performing health check on targets. |
|alb.ingress.kubernetes.io/healthcheck-path: /healthz/ready	 | specifies the HTTP path when performing health check on targets |
|alb.ingress.kubernetes.io/tags: App=Test |  specifies additional tags that will be applied to AWS resources created |
{:.table-striped}

### TLS Termination

#### At Load Balancer Level

To terminate TLS connection at Load Balancer level, perform the following actions:

[1] Create a certificate in ACM for the custom domain (e.g. test.example.com) which you will be configuring in Route53 records to point to the ALB DNS name for internet-facing routing.

[2] Get the ARN of the ACM certificate and configure it in the Ingress using annotation `alb.ingress.kubernetes.io/certificate-arn` (or) alternatively just specify the DNS host in the `spec.tls[*].hosts` / `spec.rules[*].host` fields for automatic certificate discovery.

[3] Set the listener port for HTTPS access using `alb.ingress.kubernetes.io/listen-ports` annotation.

[4] Configure suitable security policy to negotiate SSL connections with the client using `alb.ingress.kubernetes.io/ssl-policy` annotation.

Sample ingress config is as below:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "App"
  namespace: "application"
  labels:
    app.kubernetes.io/name: App
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:...
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
spec:
  ingressClassName: alb
  rules:
    - host: test.example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: app
                port:
                  number: 80
```

{% include donate.html %}
{% include advertisement.html %}

### Security Groups

#### Frontend Security Groups

These are security groups attached to the LB which can be used for restricting access to the clients e.g. allow traffic only from certain CIDR ranges.

ALB controller by default creates a new SG which it links to the LB. You can add CIDR ranges to this SG using `alb.ingress.kubernetes.io/inbound-cidrs` annotation.

However, if you wish to use an existing SG at account level to be linked to the LB instead of the managed SG created by ALB controller, you can override the behavior by specifying the custom SGs using `alb.ingress.kubernetes.io/security-groups` annotation.

Sample ingress config is as below:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "App"
  namespace: "application"
  labels:
    app.kubernetes.io/name: App
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:...
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/inbound-cidrs: "192.0.0.0/16,193.0.0.0/32"
spec:
  ingressClassName: alb
  rules:
    - host: test.example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: app
                port:
                  number: 80
```

#### Backend Security Groups

By default, ALB controller creates a shared backend security group which has range of ports which it binds to the target EC2 instances to allow traffic from the LB.

If you're specifying custom frontend security groups then you need to explicitly enable this behavior using `alb.ingress.kubernetes.io/manage-backend-security-group-rules` annotation.

Note - ALB controller uses this single shared backend security group for all LBs attached to the cluster. This ensures you don't hit the inbound rule limits per EC2 by creating an inbound rule for each ALB exposed from the cluster. Previously, for each port an inbound rule was created which meant you will easily hit the limits with just an handful of ALBs pointing to the EKS cluster. 

Below is an example of how the LB shared backend SG is created as a inbound rule in the EC2 target SG.

|Source |Port Range |Protocol |
|--|--|--|
| sg-xxxxx|30262-32440 |TCP |
{:.table-striped}

### Health Checks

Your Application Load Balancer periodically sends requests to its registered targets to test their status. These tests are called health checks.

Each load balancer node routes requests only to the healthy targets in the enabled Availability Zones for the load balancer.

**Note - If a target group contains only unhealthy registered targets, the load balancer routes requests to all those targets, regardless of their health status. This means that if all targets fail health checks at the same time in all enabled Availability Zones, the load balancer fails open. The effect of the fail-open is to allow traffic to all targets in all enabled Availability Zones, regardless of their health status, based on the load balancing algorithm.**

Below is a sample ingress file with health checks configured -

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "App"
  namespace: "application"
  labels:
    app.kubernetes.io/name: App
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:...
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/inbound-cidrs: "192.0.0.0/16,193.0.0.0/32"
    alb.ingress.kubernetes.io/healthcheck-port: status-port
    alb.ingress.kubernetes.io/healthcheck-path: /healthz/ready
spec:
  ingressClassName: alb
  rules:
    - host: test.example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: app
                port:
                  number: 80
```

{% include donate.html %}
{% include advertisement.html %}

### Shared Ingress/ALB

Typically, teams expose an ALB for each app deployed in the cluster as each of them will have ingress & service definitions as part of their deployment.

This means you will have loads of ALBs running for your cluster depending on the app count.

One optimization that can be done is sharing ALB/Ingresses for multiple apps.

This can be done by adding multiple apps in same/different namespaces to an Ingress Group which will merge all the ingress rules to a single shared ALB for traffic routing.

To group ingress resources, `alb.ingress.kubernetes.io/group.name` annotation is used.

Below shows 2 services which define this annotation to share an ALB:

**BookApp**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "BookApp"
  namespace: "application"
  labels:
    app.kubernetes.io/name: BookApp
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:...
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/inbound-cidrs: "192.0.0.0/16,193.0.0.0/32"
    alb.ingress.kubernetes.io/healthcheck-port: status-port
    alb.ingress.kubernetes.io/healthcheck-path: /healthz/ready
    alb.ingress.kubernetes.io/group.name: prod-cluster-1-group
spec:
  ingressClassName: alb
  rules:
    - host: book.example.com
      http:
        paths:
          - path: /api/book/search
            pathType: ImplementationSpecific
            backend:
              service:
                name: BookApp
                port:
                  number: 80
```

**ProductApp**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "ProductApp"
  namespace: "application"
  labels:
    app.kubernetes.io/name: ProductApp
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:...
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/inbound-cidrs: "192.0.0.0/16,193.0.0.0/32"
    alb.ingress.kubernetes.io/healthcheck-port: status-port
    alb.ingress.kubernetes.io/healthcheck-path: /healthz/ready
    alb.ingress.kubernetes.io/group.name: prod-cluster-1-group
spec:
  ingressClassName: alb
  rules:
    - host: product.example.com
      http:
        paths:
          - path: /api/product/search
            pathType: ImplementationSpecific
            backend:
              service:
                name: ProductApp
                port:
                  number: 80
```

In above example, you can see both the apps have different routes `/api/book/search` & `/api/product/search` defined with same `group.name: prod-cluster-1-group`.

ALB controller, will create a single shared ALB containing both the listener path rules binding to the respective target groups.

#### Behaviors

- If the ALB for the new IngressGroup doesn't exist, a new ALB will be created.

- If an IngressGroup no longer contains any Ingresses, the ALB for that IngressGroup will be deleted and any deletion protection of that ALB will be ignored.

- When the groupName of an IngressGroup for an Ingress is changed, the Ingress will be moved to a new IngressGroup and be supported by the ALB for the new IngressGroup. 


#### Strategies

There are different strategies on how you can share ALBs -

[1] Share ALB by each namespace

[2] Share ALB across namespaces grouping by low traffic apps

etc.

Also, there are different strategies on how these configurations are managed -

[1] Group name & ingress definitions are defined in each app deployment

[2] Central repo which will have ingress definitions with group names managed/reviewed by Ops team to ensure no routing conflicts

**Limitations - Each ALB listener can have up to 100 rules by default. This means you might have to spin off additional shared ALB's depending on how many apps are deployed in a single namespace**

#### Conflicts

- Any team can add more rules or overwrite existing rules with higher priority to the ALB for your Ingress by using the `group.name` annotation.

- Ordering of ingress rules can lead to application-wide incorrect routing if any of the Ingresses have misconfigured rules e.g. a team can misconfigure to send all traffic to their app with wildcard pattern `/*` which means they can highjack all the traffic causing impact to other apps

- Avoid giving your group a generic name like MyIngressGroup, because someone else in the cluster may create an Ingress with the same name, which adds their Ingress to your group. If they create higher priority rules, they may highjack your application’s traffic.

{% include donate.html %}
{% include advertisement.html %}

### gRPC ALB

To expose gRPC service via ALB we use following annotations:

|Annotation Example |Purpose |
|--|--|
|alb.ingress.kubernetes.io/backend-protocol-version: GRPC |specifies the application protocol used to route traffic to pods |
{:.table-striped}

Above will create a Target Group which uses `gRPC` protocol forwarding to the target instances and linked to the LB listener rules.

Sample ingress config is as below:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "App"
  namespace: "application"
  labels:
    app.kubernetes.io/name: App
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:...
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/inbound-cidrs: "192.0.0.0/16,193.0.0.0/32"
    alb.ingress.kubernetes.io/healthcheck-port: status-port
    alb.ingress.kubernetes.io/healthcheck-path: /healthz/ready
    alb.ingress.kubernetes.io/backend-protocol-version: GRPC 
spec:
  ingressClassName: alb
  rules:
    - host: test.example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: app
                port:
                  number: 10901
```

In your service definitions, you map the ALB traffic to your gRPC traffic port as follows:

```yaml
apiVersion: networking.k8s.io/v1
kind: Service
metadata:
  name: "App"
  namespace: "application"
  labels:
    app.kubernetes.io/name: App
spec:
  ports:
    - name: grpc
      protocol: TCP
      port: 10901
      targetPort: 10901
  selector:
    app.kubernetes.io/name: App
```

Your pod definition will be as follows:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "App"
  namespace: "application"
  labels:
    app.kubernetes.io/name: App
spec:
  containers:
    - name: grpc-app
      image: xxxxx
      ports:
        - name: grpc
          containerPort: 10901
          protocol: TCP
```

{% include donate.html %}
{% include advertisement.html %}

## Network Load Balancer

<figure>
    <a href="{{ site.url }}/assets/img/2022/01/eks-nlb-flow-instance-mode.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2022/01/eks-nlb-flow-instance-mode.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2022/01/eks-nlb-flow-instance-mode.png">
            <img src="{{ site.url }}/assets/img/2022/01/eks-nlb-flow-instance-mode.png" alt="">
        </picture>
    </a>
</figure>

{% include donate.html %}
{% include advertisement.html %}

### How To Provision

To provision a Network Load Balancer, you need to create a service of type `LoadBalancer`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: platform
  labels:
    app: web-app
spec:
  ports:
    - name: https
      protocol: TCP
      port: 443
      targetPort: 443
  selector:
    app: web-app
type: LoadBalancer
```

Following actions are performed by the AWS Load balancer controller, when it sees a service object of type `LoadBalancer`:

1. NLB is created in AWS for the service. Depending on the annotations it's either internal or external.

2. Target groups are created in either instance or ip mode depending on the annotations.

3. Listeners are created for each port detailed in the service definition.

4. Health checks are configured.

### Subnet Discovery

AWS Load Balancer Controller auto discovers network subnets by default.

To be able to successfully do that you need to tag your subnets as follows:

|Tag Key |Tag Value |Purpose|
|--|--|--|
|kubernetes.io/role/elb |1 |Indicates that the subnet is public. Will be used if NLB is internet-facing |
|kubernetes.io/role/internal-elb |1 |Indicates that the subnet is private. Will be used if NLB is internal |
{:.table-striped}

### Instance Mode

Instance target mode supports pods running on AWS EC2 instances. In this mode, AWS NLB sends traffic to the instances and the kube-proxy on the individual worker nodes forward it to the pods through one or more worker nodes in the Kubernetes cluster.

### IP Mode

IP target mode supports pods running on AWS EC2 instances and AWS Fargate. In this mode, the AWS NLB targets traffic directly to the Kubernetes pods behind the service, eliminating the need for an extra network hop through the worker nodes in the Kubernetes cluster.

{% include donate.html %}
{% include advertisement.html %}

### Annotations

Let's look at some of the annotations that you can configure and their behaviors.

|Annotation Example |Purpose |
|--|--|
|service.beta.kubernetes.io/aws-load-balancer-type: "external" |Indicate to use the external AWS Load balancer controller instead of the in-tree controller available in kubernetes <br/><br/>Alternatively, you can set `spec.loadBalancerClass: service.k8s.aws/nlb` which is a CloudProvider agnostic way of offloading the reconciliation for Kubernetes Service resources of type LoadBalancer to an external controller |
|service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance" |Provision NLB in Instance mode |
|service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip" |Provision NLB in IP mode |
|service.beta.kubernetes.io/aws-load-balancer-scheme: "internal" |Provision internal NLB (default)|
|service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing" |Provision internet-facing NLB |
|service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:.." |ACM certificate for NLB to terminate TLS at Load Balancer level |
|service.beta.kubernetes.io/aws-load-balancer-ssl-ports: 443 |Port to be used for listening TLS traffic |
|service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: 'Stage=prod,App=web-app' |comma-separated list of key-value pairs which will be recorded as additional tags in the ELB |
|service.beta.kubernetes.io/aws-load-balancer-attributes: access_logs.s3.enabled=true,access_logs.s3.bucket=prod-bucket,access_logs.s3.prefix=loadbalancing/web-app |Enable access logs<br/><br/>Name of the Amazon S3 bucket where load balancer access logs are stored<br/><br/>Specify the logical hierarchy you created for your Amazon S3 bucket |
|service.beta.kubernetes.io/aws-load-balancer-attributes: load_balancing.cross_zone.enabled=true |Specifies whether cross-zone load balancing is enabled for the load balancer |
{:.table-striped}

### Security Groups

You can restrict access to your NLB's in two ways:

[1] `spec.loadBalancerSourceRanges` property / `service.beta.kubernetes.io/load-balancer-source-ranges` annotations - List of CIDRs that are allowed to access the NLB.

[2] `service.beta.kubernetes.io/aws-load-balancer-security-groups` - frontend securityGroups you want to attach to an NLB. 

If you use this setting then you need to explicitly enable `service.beta.kubernetes.io/aws-load-balancer-manage-backend-security-group-rules` to `true` so that the ALB controller will manage the backend rules in the worker nodes/ENI security group.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: platform
  labels:
    app: web-app
spec:
  ports:
    - name: https
      protocol: TCP
      port: 443
      targetPort: 443
  selector:
    app: web-app
  type: LoadBalancer
  loadBalancerSourceRanges:
    - 10.1.0.0/16
```

It's critical to understand the importance of `loadBalancerSourceRanges`.

|NLB Type |Worker Node|loadBalancerSourceRanges|Behavior|
|--|--|--|--|
|Internal |Private |Not set |If you didn't set the value for loadBalancerSourceRanges, the default is 0.0.0.0/0<br/>Since the worker node and NLB is private, 0.0.0.0/0 wouldn't cause impact since it won't be reachable directly from public internet |
|Internal |Private |VPC CIDR e.g. 10.0.0.0/16 (or) ENI private IP |You can either grant access to the entire VPC CIDR or the private IP of the load balancer's network interface to allow traffic from NLB |
|Internet-facing |Private |Not set |If you didn't set the value for loadBalancerSourceRanges, the default is 0.0.0.0/0<br/>This will open traffic for the entire internet |
|Internet-facing |Private |Client CIDR |This will allow traffic only from the Client CIDR ranges since NLB by default has client IP preservation enabled |
{:.table-striped}

#### Gotchas

When you create a NLB for an application it adds below rules to the security group of the worker nodes:

|Purpose |Rule |Number of Rules|
|--|--|--|
|Health Check |TCP;Node Port;Source - Subnet Range CIDR;Description - kubernetes.io/rule/nlb/health= |One per subnet CIDR|
|Client |TCP;Node Port;Source - Source Range CIDR;Description - kubernetes.io/rule/nlb/client= |One per CIDR Range|
{:.table-striped}

Consider this setup - EKS having worker nodes across 3 AZ's and you add one loadBalancerSourceRange.

This will result in 4 rules added to the worker node security group - 3 health check rules (1 per subnet) and 1 loadBalancerSourceRange rule.

So, for one NLB you end up with 4 rules.

Security groups have a default limit of 60 rules that can be added.

As you create many NLB's/add more source ranges for your apps running in the same cluster, you will soon hit this limit and you can't create any more NLB's.

You can increase the quota but that multiplied by the quota for security groups per network interface cannot exceed 1,000.


{% include donate.html %}
{% include advertisement.html %}

Complete service file with the annotations will look as follows:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: platform
  labels:
    app: web-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
    service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: 'Stage=prod,App=web-app'
    service.beta.kubernetes.io/aws-load-balancer-attributes: access_logs.s3.enabled=true,access_logs.s3.bucket=prod-bucket,access_logs.s3.prefix=loadbalancing/web-app,load_balancing.cross_zone.enabled=true
spec:
  ports:
    - name: https
      protocol: TCP
      port: 443
      targetPort: 443
  selector:
    app: web-app
  type: LoadBalancer
  loadBalancerSourceRanges:
    - 10.1.0.0/16
```

Above file results in below configuration:

1. Uses AWS Load Balancer Controller instead of the in-tree kubernetes controller

2. Provisions an internal NLB with instance mode

3. Sets the tags provided

4. Enables access logs to the provided S3 path

5. Enables Cross Zone load balancing

{% include donate.html %}
{% include advertisement.html %}

### Traffic Flow

#### Instance Mode

<figure>
    <a href="{{ site.url }}/assets/img/2022/01/eks-nlb-flow-instance-mode.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2022/01/eks-nlb-flow-instance-mode.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2022/01/eks-nlb-flow-instance-mode.png">
            <img src="{{ site.url }}/assets/img/2022/01/eks-nlb-flow-instance-mode-instance-mode.png" alt="">
        </picture>
    </a>
</figure>

When a request is sent, it reaches the NLB which will load balance the traffic to the target backends (worker nodes).

In instance mode, the traffic gets sent to the NodePort of the instance resulting in an additional hop.

After which the kube-proxy running in the node, sends the request to the desired pod.

EKS by default runs in `iptables proxy` mode, which means that the kube-proxy will make use of iptables to route traffic to the pods. iptables mode chooses a backend at random.

Another factor is `externalTrafficPolicy` which is by default set to `Cluster` mode. Here, the traffic may randomly be routed to a pod on another host to ensure equal distribution.


{% include donate.html %}
{% include advertisement.html %}

## Integration Patterns

### API Gateway

<figure>
    <a href="{{ site.url }}/assets/img/2022/01/eks-api-gateway-private-integration.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2022/01/eks-api-gateway-private-integration.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2022/01/eks-api-gateway-private-integration.png">
            <img src="{{ site.url }}/assets/img/2022/01/eks-api-gateway-private-integration.png" alt="">
        </picture>
    </a>
</figure>

## Classic Load Balancer

<figure>
    <a href="{{ site.url }}/assets/img/2022/08/eks-elb-flow.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2022/08/eks-elb-flow.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2022/08/eks-elb-flow.png">
            <img src="{{ site.url }}/assets/img/2022/08/eks-elb-flow.png" alt="">
        </picture>
    </a>
</figure>

{% include donate.html %}
{% include advertisement.html %}

### How To Provision

To provision a Classic Load Balancer, you need to create a service of type `LoadBalancer`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: platform
  labels:
    app: web-app
spec:
  ports:
    - name: health
      protocol: TCP
      port: 9080
      targetPort: 9080
    - name: https
      protocol: TCP
      port: 443
      targetPort: 443
  selector:
    app: web-app
type: LoadBalancer
```

Following actions are performed by the In-Tree Legacy Cloud Provider in Kubernetes, when it sees a service object of type `LoadBalancer`:

1. ELB is created in AWS for the service. Depending on the annotations it's either internal or external.

2. Listeners are created for each port detailed in the service definition.

3. Health checks are configured.

### Subnet Discovery

In-Tree Legacy Cloud Provider auto discovers network subnets by default.

To be able to successfully do that you need to tag your subnets as follows:

|Tag Key |Tag Value |Purpose|
|--|--|--|
|kubernetes.io/role/elb |1 |Indicates that the subnet is public. Will be used if NLB is internet-facing |
|kubernetes.io/role/internal-elb |1 |Indicates that the subnet is private. Will be used if NLB is internal |
{:.table-striped}

{% include donate.html %}
{% include advertisement.html %}

### Annotations

Let's look at some of the annotations that you can configure and their behaviors.

|Annotation Example |Purpose |
|--|--|
|service.beta.kubernetes.io/aws-load-balancer-internal: 'true' |Indicate that we want an internal ELB. Default: External ELB|
|service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval: '60' |Specify access log emit interval|
|service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: 'true' |Service to enable or disable access logs |
|service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: prod-bucket |Specify access log s3 bucket name |
|service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: loadbalancing/web-app |Specify access log s3 bucket prefix |
|service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: 'true' |Used on the service to enable or disable cross-zone load balancing |
|service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: Stage=prod,App=web-app  |Additional tags in the ELB |
{:.table-striped}

### Load Balancer Security Group

You can configure the security groups to be attached to an ELB for restricting traffic.

|Annotation Example |Purpose |
|--|--|
|service.beta.kubernetes.io/aws-load-balancer-security-groups: "sg-53fae93f" |A list of existing security groups to be configured on the ELB created.<br/><br/>This replaces all other security groups previously assigned to the ELB and also overrides the creation of a uniquely generated security group for this ELB.<br/><br/>The first security group ID on this list is used as a source to permit incoming traffic to target worker nodes (service traffic and health checks).<br/><br/>If multiple ELBs are configured with the same security group ID, only a single permit line will be added to the worker node security groups, that means if you delete any of those ELBs it will remove the single permit line and block access for all ELBs that shared the same security group ID. This can cause a cross-service outage if not used properly|
|service.beta.kubernetes.io/aws-load-balancer-extra-security-groups: "sg-53fae93f" |A list of additional security groups to be added to the created ELB, this leaves the uniquely generated security group in place, this ensures that every ELB has a unique security group ID and a matching permit line to allow traffic to the target worker nodes (service traffic and health checks). <br/><br/> Security groups defined here can be shared between services.<br/><br/>You can use this annotation to whitelist the CIDR ranges which are allowed to connect with your load balancer.|
|.spec.loadBalancerSourceRanges |Add CIDR to the uniquely generated security group for this ELB (defaults to 0.0.0.0/0) <br/><br/> You can specify your VPC CIDR range here e.g. 10.1.0.0/16 |
{:.table-striped}

{% include donate.html %}
{% include advertisement.html %}

Complete service file with the annotations will look as follows:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: platform
  labels:
    app: web-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval: '60'
    service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: 'true'
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: prod-bucket
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: loadbalancing/web-app
    service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: Stage=prod,App=web-app
    service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled: 'true'
    service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout: '300'
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: 'true'
    service.beta.kubernetes.io/aws-load-balancer-extra-security-groups: sg-53fae93f
spec:
  ports:
    - name: health
      protocol: TCP
      port: 9080
      targetPort: 9080
    - name: https
      protocol: TCP
      port: 443
      targetPort: 443
  selector:
    app: web-app
  type: LoadBalancer
  loadBalancerSourceRanges:
    - 10.1.0.0/16
```

Above file results in below configuration:

1. Provisions an external ELB in the desired subnet.

2. Sets the tags provided.

3. Enables access logs to the provided S3 path.

4. Enables Cross Zone load balancing.

5. Adds `.spec.loadBalancerSourceRanges` as an inbound rule (covering both traffic and health port e.g. 443 & 9080) to the uniquely generated ELB security group. This security group is then added as an inbound rule to the worker node security group thereby allowing traffic to the worker nodes.

6. Adds `sg-53fae93f` as an additional security group to the ELB which has your custom inbound rules say for external traffic.


{% include donate.html %}
{% include advertisement.html %}

## References

<https://kubernetes.io/docs/concepts/services-networking/service/>

<https://kubernetes-sigs.github.io/aws-load-balancer-controller/>

<https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-target-groups.html#client-ip-preservation>

<https://docs.aws.amazon.com/elasticloadbalancing/latest/network/target-group-register-targets.html#target-security-groups>

<https://kubernetes.io/docs/concepts/services-networking/_print/#pg-374e5c954990aec58a0797adc70a5039>

<https://github.com/kubernetes/legacy-cloud-providers/blob/master/aws/aws.go>