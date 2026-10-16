# Technical Specification & Architecture Brief

**Project Name:** Local PC FinOps & Policy-Driven Kubernetes Governance Platform

**Target Audience:** Engineering Team / DevOps / Cloud Architects

**Purpose:** Shared technical documentation detailing architecture, workflow, tools, and operational workflows.

## 1. Executive Summary

This project establishes a local simulation and governance framework for managing Kubernetes deployments and cloud infrastructure costs. By mimicking production setups locally using tools like KinD, Argo CD, Open Policy Agent (OPA), and mock AWS services, the platform enforces "Shift-Left" security, policy-as-code governance, automated cloud cost optimization (FinOps), and Disaster Recovery (DR) strategies without incurring live cloud infrastructure costs during initial phases.

## 2. Core Architectural Components

### A. Local KinD Cluster (Primary Environment)

- **Cluster:** Local PC KinD (Kubernetes in Docker) Cluster acting as the primary region.

- **CD Engine:** Argo CD for GitOps-driven deployment and continuous reconciliation.

- **Mocked AWS Services:**

  - **Local Postgres Container:** Simulates Amazon RDS.

  - **Local S3 Buckets:** Simulates S3 Cost and Usage Reports (CUR) and general storage.

  - **Simulated Route 53:** Handles local traffic routing and DNS simulation.

### B. CI/CD Pipeline & Compliance Gates (Bottom Layer)

- **Repository:** GitHub Repository & Workflows.

- **Security Scanning:**

  - **Trivy / Checkov:** Scans code, container images, and IaC templates for vulnerabilities and security misconfigurations.

- **Policy Validation Gate:**

  - **OPA / Conftest:** Validates policies at the CI/CD phase before code is built or deployed.

- **Artifact Registry:** Local Docker Registry storing validated images.

### C. Governance & Automation Engine (Middle Layer)

- **OPA Gatekeeper (Runtime Governance):** Enforces dynamic rules inside the Kubernetes cluster:

  - Block public Security Groups (No Public SGs).

  - Enforce storage encryption for all persistent volumes.

  - Require a valid `CostCenter` tag on all resource manifests.

- **FinOps Cost Optimization Engine:** Python-based Lambda automation responsible for:

  - Auto-shutting down idle nodes.

  - Right-sizing EC2 instances.

  - Deleting unattached EBS volumes.

- **Mock Billing API & Data Bucket:** Stores generated cost metrics and CUR reports.

### D. Disaster Recovery (DR) & Secondary Region

- **Secondary Region:** `EU-WEST-1` (Passive DR / Warm Standby EKS Cluster).

- **Infrastructure as Code (IaC):** Managed via Terraform.

- **DNS Failover:** Route 53 DNS Failover routes traffic to the standby cluster during primary system failure.

- **Alerting & Notifications:** Amazon SNS and Slack Webhook integrations for real-time compliance and cost alerts.

## 3. End-to-End Operational Workflow

```
[Developer] 
    │
    ▼ (Push Code/Config)
[GitHub Workflow] ──► [Trivy/Checkov Scan] ──► [OPA/Conftest Policy Gate]
                                                        │
                                                 (Passed / Pass Gate)
                                                        │
                                                        ▼
                                             [Local Docker Registry]
                                                        │
                                                        ▼ (Deploy)
                                                    [Argo CD]
                                                        │
                                                        ▼
                                           [KinD Cluster (Primary)]
                                                        │
                                    ┌───────────────────┴───────────────────┐
                                    ▼                                       ▼
                       [OPA Gatekeeper Runtime Check]           [Mock RDS / S3 / Route 53]
                                    │
                                    ▼
                     [FinOps Engine (Python Lambda)]
                                    │
                         (Scheduled via CloudWatch)
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         ▼                          ▼                          ▼
[Mock Billing API]       [SNS & Slack Alerts]      [Route 53 Failover to DR]
```

### Detailed Steps:

1. **Developer Commits Code:** The developer pushes code or infrastructure changes to GitHub.

2. **Shift-Left Security Gate:**

   - Trivy and Checkov scan the source code and configurations.

   - OPA/Conftest checks policies. If compliance checks fail, the pipeline halts.

3. **Artifact Creation & Deployment:** Validated code is pushed to the local Docker Registry and deployed to the local KinD cluster via Argo CD GitOps synchronization.

4. **Runtime Enforcement:** OPA Gatekeeper inspects pods and resources in real-time to ensure no prohibited configurations (like unencrypted volumes or missing cost tags) are applied.

5. **Scheduled FinOps Execution:** CloudWatch triggers the FinOps engine to detect idle nodes, oversized resources, and unattached volumes, updating the Mock Billing API.

6. **Alerting & DR:** Compliance status and cost reports are posted to Slack/SNS. In case of a major cluster outage, Route 53 switches traffic to the DR Standby cluster managed by Terraform.

## 4. Key Metrics & Value Proposition

| **Objective** | **Problem Solved** | **Implemented Solution** | | **Cost Management** | High cloud bills from orphaned EBS volumes and idle nodes. | Python Lambda FinOps Engine + Mock S3 CUR analytics. | | **Security & Compliance** | Insecure deployments reaching production environments. | Shift-Left pipeline gates (Trivy/Checkov/OPA) + OPA Gatekeeper. | | **High Availability** | Single point of failure in primary region. | Route 53 DNS Failover to warm standby EKS cluster in EU-WEST-1. | | **Developer Ergonomics** | High cost of cloud testing during development. | Complete local emulation stack using KinD and mock AWS containers. |

## 5. Team Responsibilities & Action Items

- **DevOps / Platform Lead:** Maintain Argo CD sync pipelines and local KinD cluster stability.

- **Security Engineer:** Define OPA policies, Gatekeeper constraint templates, and Trivy/Checkov scanning rules.

- **FinOps / Cloud Engineer:** Maintain the Python Lambda optimization scripts and Mock Billing API integrations.

- **Infrastructure Team:** Manage Terraform scripts for the secondary region setup (`EU-WEST-1`).

