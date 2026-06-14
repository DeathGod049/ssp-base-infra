# SSP Base Infrastructure

This repository contains the foundational Terraform infrastructure for the Smart Shop Platform (SSP). 

It is designed to be deployed **before** any individual microservices. It sets up the core networking, shared databases, event bus, and container orchestration environment that the microservices rely on.

## Architecture

This base infrastructure provisions:

1.  **Networking (`vpc` module):**
    *   A VPC (`10.0.0.0/16`) spanning 2 Availability Zones.
    *   2 Public Subnets (for ALBs, NAT).
    *   2 Private Subnets (for ECS tasks, Databases, Kafka, Lambda ENIs).
    *   1 NAT Gateway (in AZ-1 for cost optimization) for outbound internet access from private subnets.
    *   Internet Gateway and Route Tables.

2.  **Container Orchestration:**
    *   An Amazon ECS Cluster to host Fargate tasks.

3.  **Application Load Balancer (`alb` module):**
    *   A shared, internet-facing ALB that routes traffic to the various microservices.
    *   Listens on HTTP (80) and redirects to HTTPS (443).

4.  **Databases:**
    *   **PostgreSQL (`rds` module):** Amazon RDS instance for relational data (used by Auth, Order, and Inventory services).
    *   **MongoDB (`documentdb` module):** Amazon DocumentDB cluster for NoSQL data (used by the Product service).
    *   **Redis (`elasticache` module):** Amazon ElastiCache cluster for in-memory caching and session state (used by the Cart service).

5.  **Event Bus (Placeholder):**
    *   An EC2 instance configured to run a self-managed Kafka cluster. *(Note: This uses a placeholder AMI and setup in the current configuration and requires manual Kafka installation or user_data scripting for a production setup).*

## The Parameter Store Pattern

A key design principle of this architecture is decoupling infrastructure provision from application configuration.

When this Terraform code runs, it automatically outputs the generated endpoints (e.g., the RDS URL, DocumentDB cluster endpoint, Redis host) to **AWS Systems Manager (SSM) Parameter Store**.

The microservices are then configured to dynamically fetch these values from SSM at runtime. This means:
*   You don't need to hardcode database URLs in your service code or environment variables.
*   If the database endpoint changes, the services just need a restart to pick up the new endpoint.

## Deployment

This infrastructure is designed to be deployed via Jenkins using the included `Jenkinsfile`.

### Prerequisites

1.  **S3 Backend:** You must manually create an S3 bucket to store the Terraform state. 
    *   Create a bucket (e.g., `ssp-terraform-state-bucket`).
    *   Enable versioning.
    *   Update the `backend "s3"` block in `main.tf` with your bucket name.
2.  **Jenkins Setup:**
    *   Ensure Jenkins has the Terraform plugin installed.
    *   Ensure Jenkins is configured with AWS credentials that have administrator privileges.

### Deployment Steps

1.  Configure a Multibranch Pipeline in Jenkins pointing to this repository.
2.  The pipeline will trigger automatically on push and run `terraform plan`.
3.  The pipeline will pause at the **"Approval"** stage.
4.  Review the plan output in Jenkins.
5.  Click **"Apply"** to authorize Jenkins to run `terraform apply` and provision the resources.

## Post-Deployment (Action Required)

While this script automates endpoints, you must manually create the **secrets** in AWS SSM Parameter Store as `SecureString` types before deploying the microservices:

*   `/ssp/auth/jwt_secret`
*   `/ssp/payment/stripe_secret_key`
*   `/ssp/payment/stripe_webhook_secret`

You should also create the sender email for notifications as a `String`:
*   `/ssp/notification/ses_sender_email`
