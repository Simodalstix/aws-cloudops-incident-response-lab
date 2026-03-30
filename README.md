# AWS CloudOps Incident Response Lab

Hands-on lab environment for building AWS operational skills using the CLI. Covers provisioning, monitoring, and troubleshooting a basic web-serving stack.

## What This Builds

- EC2 instance running a simple HTTP service
- Application Load Balancer
- CloudWatch alarms
- SNS notifications
- IAM role scoped to the EC2 instance

## Purpose

- Develop AWS CLI fluency through repetition
- Practice identifying and resolving common operational failures
- Maintain a reusable CLI reference
- Serve as a stable base environment for future projects

## Repo Structure

```
├── architecture/          # Diagrams
├── docs/
│   ├── cli-playbooks/      # Reusable CLI command reference
│   └── scenarios/         # Structured troubleshooting exercises
```

## Prerequisites

- AWS account with admin access
- AWS CLI v2 installed and configured
- A default VPC (or willingness to create one)

## Approach

All infrastructure is created via AWS CLI. No IaC, no console clicking. Each step is verified before moving to the next.

## Status

🔧 In progress — building the base environment.
