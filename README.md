
# Deploying Infrastructure & Application to Amazon EKS using Jenkins (IAM Role Assumption – No Access Keys)

##  Project Goal
This project demonstrates end-to-end deployment of both infrastructure and application code to Amazon Elastic Kubernetes Service (EKS) using a Jenkins Multibranch Pipeline, without storing AWS access keys or secret keys in the CI/CD environment.

Instead, we attach an IAM Role to the Jenkins build node EC2 instance, enabling it to assume permissions dynamically and interact with AWS resources securely.

 **Why this matters**  
By avoiding static credentials (ACCESS_KEY & SECRET_ACCESS_KEY) and relying solely on role assumption, we greatly reduce the attack surface and enforce security-by-design.


##  Architecture Overview

| Component | Role |
|-----------|------|
| **Jenkins Master (EC2)** | Manages the overall pipeline orchestration and administrative tasks. It doesn't run any heavy workloads (light workloads only). It also hosts the User Interface |
| **Jenkins Build Node (EC2)** | This is a separate instance where all the heavy lifting happens. It executes Terraform, Docker builds, AWS CLI, `kubectl`, Helm. Its the workhorse of the pipeline. This node has the IAM Role attached, giving it temporary permissions.|
| **AWS EKS** | Kubernetes cluster for application hosting |
| **AWS ECR** | Container registry for storing application images |
| **IAM Role** | Attached to Jenkins node for AWS access (no static credentials). This role's permissions allow the node to provision infrastructure and deploy applications without hardcoded credentials. |
| **NGINX Ingress + AWS NLB** | Manages public access to apps |

---

##  Technology Stack
- **CI/CD:** Jenkins (Multibranch Pipeline)
- **Infrastructure as Code:** Terraform
- **Container Orchestration:** AWS EKS
- **Container Registry:** AWS ECR, Docker 
- **Deployment:**  Helm
- **Authentication:** AWS IAM Roles
- **Networking:** NGINX Ingress Controller,  AWS Network Load Balancer



##  Implementation Workflow (Two pipelines, one goal)
Yes, two pipelines, one goal. We have one pipeline for the infrastructure and another pipeline for the application code. Each pipeline takes its deployment file from its branche.

### 1. Infrastructure as Code (IaC) Pipeline

**Branch:** `jenkins-iac` 
**Goal:** Provision all the necessary cloud resources via Terraform.

This pipeline, defined in the `Jenkinsfile` on the `jenkins-iac` branch, is a parameterized job that allows you to either create or destroy your infrastructure.
```groovy
parameters {
    choice(
        name: 'ACTION',
        choices: ['apply', 'destroy'],
        description: 'Choose the action to perform: apply (default) or destroy.'
    )
}
```

-   **`apply` stage:** Runs `terraform apply` to provision the EKS cluster, ECR repository, and other dependencies. It also deploys the NGINX Ingress Controller, which automatically provisions an AWS NLB for traffic management.
    
-   **`destroy` stage:** Systematically tears down all the resources. It first deletes the Ingress Controller and ECR images, then runs `terraform destroy` for a clean, automated cleanup.
    So, the **Apply flow** will run: Terraform init → validate → plan → apply → configure kubeconfig → deploy NGINX ingress
And the **Destroy flow** will Remove ingress → delete ECR images → run Terraform destroy.
All in all, the highlight of deploying this infrastructure is the manner in which it was done. There were no manual AWS credentials added to the AWS console, the Network Load Balancer was automatically provisioned, and we have secure IAM role access.

### 2. Application Deployment Pipeline

**Branch:** `main` 
**Goal:** Build, tag, push to ECR and deploy the application to the EKS cluster.

This pipeline automates the application delivery process.

-   **Build & Push:** It builds a Docker image from the application's `Dockerfile`. The image is automatically tagged with the unique Jenkins `BUILD_NUMBER` and `latest` for traceability and then pushed to ECR using IAM role authentication.
    
-   **Deploy to EKS:** Using Helm, the pipeline deploys the application to the EKS cluster. It manages credentials for pulling images from ECR and ensures the application is deployed with the correct image tag. Although one must ensure the kubeconfig file is updated with the right registry credentials first, and also make sure the right AWS account is the one making the calls to the AWS console where the cluster resides.

---

### **3. Domain Mapping & Public Access**
To expose the deployed application to the public internet, a crucial final step involved configuring DNS records on GoDaddy.

-   **Ingress Controller and NLB:** The NGINX Ingress controller deployed to the EKS cluster automatically provisions an AWS Network Load Balancer (NLB). The DNS name for this NLB is obtained by running `kubectl get ingress` from the command line.
-   **Kubeconfig Update:** To run `kubectl` commands outside of the Jenkins pipeline, the `kubeconfig` file must first be updated. The `aws eks update-kubeconfig --name <cluster_name> --region <region>` command, which is also logged in the pipeline output, is used for this purpose. The `aws sts get-caller-identity` command can be run beforehand in order to verify that the correct IAM principal is making the call, or else, no access to EKS cluster.
-   **CNAME Record Configuration:** Two CNAME records were created on GoDaddy, both pointing to the NLB's DNS name. One was for the database and the other for the application.
-   **Traffic Routing:** The NGINX Ingress controller handles the traffic routing. Based on the incoming hostname, the Ingress rules direct the traffic to the correct service within the EKS cluster.

---

## Key DevOps Principles Applied
-   **Infrastructure as Code (IaC):** All infrastructure both for EKS and the Ingress is defined in Terraform, ensuring consistency and repeatability.
    
-   **Continuous Integration/Continuous Delivery (CI/CD):** The entire workflow is automated, from a code commit to a full deployment ensuring that we have a fully automated deploy pipeline.
    
-   **Version Control:** All code, including IaC and pipeline scripts, is managed in Git for a complete history and easy rollbacks.
 
-   **Security :** IAM role-based authentication is a fundamental security control integrated directly into the pipeline. In this project, only the IAM Role principal can manage EKS resources.

## A Deeper Look into the Security Advantage

This project's core security principle is the use of IAM roles for authentication. This design decision has a critical implication:

> **You cannot manage Kubernetes objects in the EKS cluster from the AWS Console.**

The EKS cluster was created by the Jenkins build node's IAM Role, not by an individual user. This means only a principal explicitly granted permission can interact with the cluster. This design prevents manual, ad-hoc changes to the cluster and enforces a strict, auditable, pipeline-driven management process.

---

## Conclusion
This project provides a highly secure blueprint for a CI/CD pipeline on AWS. By embracing IAM role-based authentication and a distributed Jenkins architecture, it achieves a workflow that is not only automated but also auditable and secure by design.
By combining Terraform, Jenkins Multibranch Pipelines, IAM role assumption, and Helm, we achieve **Security** (in terms of no static secrets), **Consistency** (via IaC), **Speed** (via CI/CD automation), and **Scalability** (via Kubernetes on EKS).

The successful end-to-end deployment, from infrastructure provisioning to application deployment and custom domain mapping, validates the effectiveness of this approach. It proves that a secure, automated, and repeatable deployment process is achievable without ever needing to expose sensitive credentials.


