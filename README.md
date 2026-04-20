# AWS CloudOps CLI Lab

A hands-on CLI reference for provisioning and operating a web-serving stack on AWS. Built as a practical exercise to develop AWS CLI fluency — no IaC, no console.

## What's Covered

| Playbook | Topics |
|---|---|
| 00 — Foundations & IAM | Regions, profiles, IAM roles, instance profiles |
| 01 — EC2 | Launch, connect, AMI lookup, key pairs, security groups |
| 02 — ALB & Target Groups | Load balancer setup, target registration, health checks |
| 03 — CloudWatch & SNS | Alarms, log groups, metrics, SNS topics and subscriptions |
| 04 — Scenarios | Common failure patterns and how to diagnose them |
| 05 — Autoscaling | Launch templates, ASGs, scaling policies |
| 06 — Networking | VPC, subnets, route tables, internet gateways, NAT |
| 07 — VPC from Scratch | Full VPC build walkthrough from a blank account |

## Prerequisites

- AWS account with admin access
- AWS CLI v2 configured (`aws configure`)

## Approach

Everything is done via the AWS CLI. Each command is verified before moving to the next. The goal was exposure and repetition — understanding what the API actually does rather than letting a tool abstract it away.

## Status

Complete. This repo served its purpose as a CLI familiarisation exercise.
