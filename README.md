# creative-force-test


## <a name='TableofContents'></a>Table of Contents
- [creative-force-test](#creative-force-test)
  - [Table of Contents](#table-of-contents)
  - [Requirements](#requirements)
  - [Architecture](#architecture)
  - [Solutions](#solutions)
    - [Networking](#networking)
    - [Database](#database)
    - [Amazon EKS](#amazon-eks)
    - [Git and CICD flow](#git-and-cicd-flow)
      - [CICD flow](#cicd-flow)
      - [Gitops System](#gitops-system)
    - [Security](#security)
    - [Backup and Disaster Recovery](#backup-and-disaster-recovery)
    - [Loging and Monitoring](#loging-and-monitoring)
    - [Other resource](#other-resource)
    - [Potential shortcomings](#potential-shortcomings)



## <a name='Requirements'></a>Requirements

- [Terraform](https://www.terraform.io/downloads.html)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [AWS EKS]()
- [Gitops]()


## <a name='Architecture'></a>Architecture
**Architecture Overview**

![architecture](./image/cfdevops.png)

## <a name='Solutions'></a>Solutions

***I chosen AWS cloud***


### <a name='Networking'></a>Networking

  
- I created a dedicated VPC for this project. In my VPC, I created a public subnet and a private subnet for each Availability Zones in the region. A private subnet does not have a direct route to the internet, which means that instances launched in that subnet cannot access resources on the internet or be accessed from the internet. To allow instances in a private subnet to access the internet, I used a NAT Gateway (NGW). 
- A NAT Gateway is deployed in a public subnet and acts as a bridge between instances in the private subnet and the internet. When an instance in a private subnet sends a request to the internet, the request is forwarded to the NAT Gateway, which replaces the instance’s private IP address with the NAT Gateway’s public IP address and sends the request to the internet. When the response is received, the NAT Gateway translates the response back to the instance’s private IP address and sends it back to the instance.
- To increase high availability for NGW, I created one NGW in each public subnet.

### <a name='Database'></a>Database

- I chose to use Aurora instead of running postgres directly to avoid dealing with persistent state in the k8s cluster or minimize future operational efforts. Amazon Aurora is a fully managed database service, which means AWS takes care of routine database tasks like patching, backups, and scaling, allowing I to focus on building my application.
- Amazon Aurora is designed for high performance and low latency. It provides up to five times the throughput of standard MySQL and PostgreSQL databases. This makes it suitable for applications with demanding performance requirements. Aurora can easily scale database up or down as my application's needs change. It supports auto-scaling, which can automatically adjust the database's capacity to handle varying workloads.
- Amazon Aurora provides high availability and fault tolerance. It replicates data across multiple Availability Zones (AZs), which ensures that my database remains available even in the event of an AZ failure.
- Amazon Aurora offers automated backups and point-in-time recovery. I can easily restore my database to a specific point in time, which is important for data protection and disaster recovery.
- Amazon Aurora provides a Writer for write operations, ensuring data consistency and durability, and one or more Readers for read operations, enhancing read performance, scaling, and high availability. This architecture allows to balance the workload, achieve fault tolerance, and improve overall database performance by distributing read traffic across multiple read replicas while maintaining data integrity through replication from the Writer.
  - Aurora Writer (Primary Instance):

    - Write Operations: The Aurora Writer is the primary database instance responsible for handling write operations (INSERT, UPDATE, DELETE) on the database. It is the authoritative source for data modifications.

    - High Availability: Aurora Writer is designed for high availability and durability. Data written to the Writer is replicated across multiple Availability Zones (AZs) for redundancy and fault tolerance. If the Writer instance fails, Aurora automatically promotes a read replica to become the new Writer, ensuring minimal downtime.

    - Failover: In the event of a Writer instance failure, Aurora triggers an automatic failover, promoting a read replica to become the new Writer. Failovers are usually transparent to applications and result in minimal disruption.

    - Scaling: Aurora Writers can be scaled vertically to handle more write traffic, and I can also increase the size of the underlying storage volume as my data grows.
  - Aurora Readers (Read Replicas):

    - Read Operations: Aurora Readers are read-only database instances that are asynchronously replicated from the Writer. They can handle SELECT queries and other read-intensive operations. These Readers provide horizontal scalability for read traffic.

    - High Availability: Aurora Readers can be placed in different Availability Zones, allowing I to distribute read traffic and increase availability. If one reader fails, others remain accessible.

    - Scaling: I can create multiple Aurora Readers to distribute read traffic and improve the overall performance of my application. Aurora automatically handles replication and keeps the Readers up to date.

    - Load Balancing: I can configure my application to distribute read traffic across multiple Aurora Readers using load balancing or connection pooling. This helps optimize the performance of read-heavy workloads.

    - Read Consistency: Aurora Readers provide read consistency, ensuring that the data retrieved from a Reader is up to date with the latest committed transactions.

  

### <a name='Amazon EKS'></a>Amazon EKS

- We have packaged the applications by creating docker images for them, so I will use EKS to deploy these applications there.
- To achieve zero downtime updates for components in Kubernetes, we can combine strategies as follows:
  - Rolling Updates: Kubernetes supports rolling updates for applications. When I update an application's Deployment or StatefulSet, Kubernetes ensures that a new version of the application is gradually deployed while the old version is gradually retired. This rolling update strategy prevents service disruption. I also use `Readiness and Liveness Probes` to define readiness and liveness probes for ,my containers. Readiness probes indicate when a container is ready to receive traffic, while liveness probes detect when a container is healthy and should continue running. These probes ensure that only healthy containers receive traffic.
  - Blue-green: I'll show in next section

- To horizontally scale the application automatically, I will use Horizontal Pod Autoscaling (HPA) in Kubernetes, and to automatically horizontally scale the nodes in EKS, I will use the [Auto Scaler](https://github.com/kubernetes/autoscaler).
  - HPA: automatically scales the number of pods in a Deployment, ReplicaSet, or StatefulSet based on observed CPU utilization or other select metrics. This allows my application to handle varying loads efficiently without manual intervention. Here's how HPA works:
    - Metrics Collection: HPA collects metrics from my application, such as CPU usage or custom metrics, depending on my configuration.

    - Comparison to Desired Metrics: HPA compares the collected metrics to the desired metric values I've defined in my HorizontalPodAutoscaler resource.

    - Decision Making: Based on the metric values, HPA makes a decision about whether to scale the number of pods up or down to maintain the desired metric values.

    - Scaling: If scaling is necessary, HPA updates the number of desired replicas, and Kubernetes automatically adds or removes pods to meet the new replica count.

    - Continuous Monitoring: HPA continues to monitor metrics and make scaling decisions as needed.
  - [Auto Scaler](https://github.com/kubernetes/autoscaler): Cluster Autoscaler is a tool that automatically adjusts the size of the Kubernetes cluster when one of the following conditions is true:
    - there are pods that failed to run in the cluster due to insufficient resources.
    - there are nodes in the cluster that have been underutilized for an extended period of time and their pods can be placed on other existing nodes.




### <a name='Git and CICD flow'></a>Git and CICD flow

- We have several different Source Code Management (SCM) systems, but for this project, I will use CodeCommit, a service available on the AWS cloud, it offers a secure and scalable platform for hosting and managing Git repositories.
- I used AWS Codebuild for my CI processes & build docker images. It is a fully managed continuous integration service that compiles source code, runs tests, and produces software packages. It supports multiple programming languages and build environments. And I used CodePipeline to manage and automate the release process, ensuring that code changes are consistently built, tested, and deployed in a controlled and predictable manner.
- So, How They Work Together:
    - CodeCommit is used to store my source code repositories.
    - CodePipeline is used to define and automate software release process, and it can trigger pipeline execution on code changes in CodeCommit.
    - CodeBuild is used to compile, build, and test my code, and it can be used as a build action within a CodePipeline stage.
- I also use AWS ECR - a fully managed container image registry service provided by Amazon Web Services (AWS). It's designed to store, manage, and deploy Docker container images. ECR is tightly integrated with other AWS services, particularly Amazon Elastic Container Service (ECS) and Amazon Kubernetes Service (EKS), making it a key component of AWS's containerization ecosystem.

#### <a name='CICD flow'></a>CICD flow

- I'll design CI/CD in a branching model. We will have branches corresponding to deployment environments such as dev, staging, and production.
  
  1. Codebase Setup
Developers work on a shared codebase, usually stored in a version control system (e.g., Git).
A typical branching model includes a `develop` branch and feature branches.

  1. Feature Development
Developers create feature branches for new features, bug fixes, or improvements.
These branches are isolated from the `develop` branch, allowing independent development.
Developers commit their changes to these feature branches.

  1. Pull Requests (PRs)
Developers create PRs (or merge requests) to propose merging their feature branch into the `develop` branch.
The PR undergoes code review, where team members review and discuss the changes.
CI pipelines are triggered for PRs to ensure they don't introduce issues into the `develop` branch.

  1. Merging and develop
After merge code from feature branch to `develop` branch, CI will be running. 
If the CI pipeline fails, developers are alerted to fix the issues.
If the CI pipeline passes, it indicates that the feature branch's code is stable.
Developers create PRs (or merge requests) to propose merging `develop` branch to `staging` branch.

  1. Merging and Staging
Once a PR is approved, it's merged into the `staging` branch.
The `staging`` branch represents the staging environment
CI/CD pipelines are triggered on the `staging` branch to prepare for deployment.

  1. Continuous Deployment (CD)
CD pipelines automatically deploy the application to staging or develop environments for further testing.
Automated tests, such as integration, regression, and performance tests, are run in these environments.
If the tests pass, the code is considered ready for production.

  1. Production Deployment
Once the code is thoroughly tested and validated in staging, it's deployed to the production environment via `prod` branch.
We create PRs to propose merging `staging` branch to `prod` branch. 
The CD pipeline handles the deployment, which may include blue-green deployments, or rolling updates to ensure zero downtime.

  1. Monitoring
Post-deployment, monitoring tools track the application's performance and health.
If any issues arise, the CI/CD process can be used to quickly fix and redeploy the application.


#### <a name='Gitops System'></a>Gitops System

- The output of my CI process is a docker image pushed to storage in a private ECR.
- In CD step, I will integrate a GitOps system into the Continuous Deployment (CD) process by using the ArgoCD ecosystem, which includes ArgoCD, Argo Rollout, Argo CD Image Updater. I have deployed them into EKS cluster. I also created other repository to store my code for deployment such as manifest file of k8s, ArgoCD configuration files or helm chart template for deploying my appication. When changes are pushed to the repository, ArgoCD detects them and starts a reconciliation process and ArgoCD automaticly trigger this repository to deployment applition on EKS.
- In CD process, when docker image is pushed to AWS ECR, Argo CD Image updater can check for new versions of the container images that are deployed with your Kubernetes workloads and automatically update them to their latest allowed version using Argo CD. I use Argo Rollout to apply blue-green deployment for my applications. To use Argo Rollouts, create a custom resource called a `Rollout` in k8s. This resource specifies how Blue-Green deployment should work. The Rollout manages two Replica Sets: the "Blue" Replica Set and the "Green" Replica Set. The "Blue" represents the existing version of my application, while the "Green" represents the new version. Initially, all traffic is directed to the "Blue" Replica Set, which serves the current production version of my application. Deploy new version to the "Green" Replica Set, but without routing any traffic to it. If everything looks good, I can promote the "Green" Replica Set to become the new production version, and the old "Blue" environment is retired.
If issues are detected during analysis, I can easily roll back to the previous version by shifting traffic back to the "Blue" Replica Set.



### <a name='Security'></a>Security

- To Network security, I have disabled unnecessary services from being exposed to the public internet and only allow connections within the internal VPC and also used Security Group to control all inbound and outbound traffic to a particular entity. A security group acts as a firewall that controls the traffic allowed to and from the resources in my virtual private cloud (VPC). I can choose the ports and protocols to allow for inbound traffic and for outbound traffic. For example, I disabled Public accessibility to my Aurora instances and with inbound traffic into Aurora, only allow instances's security groups are allowed and blocked all another connection. Or with EKS cluster endpoint, disabled public enpoint and only enable private endpoint. [EKS cluster endpoint](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html) 
  
- To security in transit, I have used cert-manager to automates the process of obtaining, renewing, and managing TLS certificates. Cert-Manager has built-in support for Let's Encrypt, a popular Certificate Authority (CA) that provides free TLS certificates. This integration simplifies the process of obtaining TLS certificates from Let's Encrypt. It simplifies the certificate lifecycle management, reducing the need for manual certificate handling. Cert-Manager is designed to work seamlessly with Kubernetes, making it an ideal choice for securing applications and services running in Kubernetes clusters. It leverages CustomResourceDefinitions (CRDs) to configure and manage certificates. Cert-Manager automatically handles certificate renewal and rotation. This is critical for maintaining the security and availability of applications and services.

- To security at rest, especially data, Amazon Aurora provides encryption by default, which means all data at rest is encrypted when you create a new Aurora DB cluster. For Data in Transit Encryption, I use SSL/TLS encryption for data in transit (data traveling between your application and the database).

- To secrets and encryption sensitive data, I used [Hashicorp Vault](https://developer.hashicorp.com/vault/docs/what-is-vault). It is deployed on EKS cluster. I will use Vault to store sensitive environment variables for the applications and automatically inject them during the application runtime. Vault uses various storage backends and encryption engines to store and protect sensitive data. The data encryption is performed using encryption keys generated and managed by Vault itself. Vault supports a variety of encryption methods and algorithms, such as AES, RSA, and Shamir's Secret Sharing. We can also use AWS Secret Manager as an alternative to Vault, but it may incur additional costs.

- For permissions in pod EKS interact with resource on AWS, I'll use [IAM roles for service accounts(IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html). IAM roles for service accounts provide the ability to manage credentials for my applications, similar to the way that Amazon EC2 instance profiles provide credentials to Amazon EC2 instances. Instead of creating and distributing my AWS credentials to the containers or using the Amazon EC2 instance's role, I associate an IAM role with a Kubernetes service account and configure my Pods to use the service account. For example, if one of my pods needs to upload objects to S3, instead of using AWS credentials, I can use IRSA to enhance security.

- We can also use Sonarqube to scan sourcecode to mitigate vulnerbility application.

### <a name='Backup and Disaster Recovery'></a>Backup and Disaster Recovery

- For data in Aurora, Amazon Aurora continuously backs up data to Amazon S3, providing high availability and durability for database. This continuous backup approach allows you to perform point-in-time recovery, including to the exact second, if needed. By default, Amazon Aurora retains backups for one day. We can adjust the retention period based on your requirements. Retention periods can be set to up to 35 days. In addition to automatic backups, we can create manual snapshots of Aurora database at any time. These manual snapshots are retained until we choose to delete them. Aurora supports cross-region replication, allowing we to create read replicas in different AWS regions. This can serve as a disaster recovery strategy.

- For applications on EKS, I'll create cronjob backup the ArgoCD Configuration for all applications and push it to AWS S3. We can import this backup at any time in case of a disaster, and the application will be preserved.

- All applications on EKS are deployed with multiple pods to ensure high availability and automatic auto-scaling. For example: Service Apps, Vault is deployed minimum 3 pods, .....

### <a name='Loging and Monitoring'></a>Loging and Monitoring

- Monitoring: 
  - I'll use cloudwatch for monitoring Amazon Aurora. Enhanced Monitoring is a feature that provides detailed system-level data for Amazon RDS and Amazon Aurora. You can enable Enhanced Monitoring during the creation of your Aurora instance or by modifying an existing instance. You can choose the desired granularity (e.g., 1 minute) and specify the data you want to collect (e.g., CPU, memory, file system, etc.). By default, Amazon Aurora logs to the AWS RDS Log Streams in CloudWatch Logs. You can configure log types like error logs, slow query logs, and general logs. These logs help you identify and troubleshoot issues. CloudWatch Alarms allow you to monitor metrics and automatically respond to specific conditions. You can create alarms for important metrics to trigger actions when certain thresholds are breached. For example, you can set up an alarm to notify you if the CPU utilization exceeds a certain percentage or if the number of database connections goes above a specified limit.
  - For application monitoring, I use [`Prometheus Operator provides Kubernetes`](https://github.com/prometheus-operator/prometheus-operator),  it extends the capabilities of Prometheus and makes it easier to monitor containerized applications and infrastructure within a Kubernetes environment. Here are some of the key features and explanations of how the Prometheus Operator integrates with Kubernetes. It automatically generates Prometheus scrape configurations based on ServiceMonitors. A ServiceMonitor is a custom resource that defines the endpoints and labels for scraping metrics from services. The Prometheus Operator integrates with AlertManager, which allows you to define alerting rules and notification configurations using custom resources. You can set up alerts for various conditions and define alert receivers includes Telegram, Mail, Slack, ... While not part of the Prometheus Operator itself, it's common to use Grafana alongside Prometheus for visualization. The operator can help with the setup and management of Grafana.

- Logging: EFK is best tool on EKS. I'll deploy fluentbit as daemonset in EKS cluster to collect all logs data from sources and forwards the parsed and structured log data to Elasticsearch, where it's indexed and stored. Elasticsearch provides powerful full-text search and analysis capabilities.. Kibana is used to interact with the indexed log data. Users can create custom visualizations, dashboards, and perform ad-hoc searches to gain insights into the data. This is particularly useful for monitoring and troubleshooting.


### <a name='Other resource'></a>Other resource
- ALB: 
  -  I use Application Load Balancer (ALB) ingress to expose services inside the EKS cluster to the outside. ALB (Application Load Balancer) Ingress is a Kubernetes resource that allows configure AWS Application Load Balancers to route incoming traffic to various services in a Kubernetes cluster. It's a way to expose your Kubernetes services externally and manage traffic routing efficiently. 
  -  An Ingress resource is created with routing rules, specifying how incoming requests should be handled.
  - The AWS ALB Ingress Controller watches for changes in Ingress resources.
  - When a new Ingress resource is created or an existing one is updated, the Ingress Controller configures the corresponding AWS ALB accordingly.
  - The ALB listens for incoming traffic and routes it based on the Ingress rules to the backend Services in the Kubernetes cluster.
  - I just use an ALB for all services inside EKS. Each service corresponds to an ALB target group.

- Cloudfront:
  - Integrating Amazon CloudFront with an Application Load Balancer can significantly enhance the security, performance, and scalability of your web application. CloudFront's edge locations and caching capabilities reduce latency, offload traffic, and protect your application from various online threats. It's a powerful combination for delivering content to users reliably and efficiently.

- Route53:
  - When a user accesses your application, Route 53 routes the request to the nearest CloudFront edge location.
  - CloudFront serves cached content when available, reducing latency and offloading traffic.
  - If the content isn't cached or if there's a cache miss, CloudFront forwards the request to the ALB.
  - The ALB, in turn, directs the traffic to the appropriate service or pods within your EKS cluster based on the defined rules and target groups.


### <a name='Potential shortcomings'></a>Potential shortcomings

- I think we should have tracing system such as jaeger or APM. I'll cooperate with dev team to intergrate with APM system to tracing error on my project. Application Performance Monitoring (APM), is a technique used to monitor and analyze the performance and behavior of applications, especially in distributed and microservices-based systems. It involves tracking requests as they flow through various components of an application, recording timing and contextual information at each step, and then aggregating and analyzing this data to gain insights into the application's performance. 
- Advantages of Tracing in APM:
  - End-to-End Visibility: Tracing provides a complete view of how requests move through your application, helping you identify bottlenecks and latency issues. This visibility is critical in microservices architectures where a single request can traverse multiple services.

  - Performance Optimization: Tracing helps in the identification of performance issues and bottlenecks, allowing you to optimize your application by focusing on the most critical areas. It can lead to better resource utilization, reduced latency, and improved user experience
  
  - Root Cause Analysis: When an issue occurs, tracing can pinpoint the root cause by showing where a request deviates from the expected path or where errors occur. This helps in quicker troubleshooting and resolution.

  - Resource Allocation: Tracing can highlight underutilized or overused resources, which is valuable for resource allocation decisions and cost optimization.

  - Service Dependencies: In complex systems, tracing can track dependencies between different services and components, helping you understand how changes in one service affect others.

  - Security and Compliance: Tracing can help in security audits and compliance checks by providing a detailed record of data interactions and the paths requests take through the system. 