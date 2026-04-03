# Auto Scaling Groups CLI Playbook

Focus: launch templates, ASG creation, scaling policies, health, and failure simulation

Troubleshoot in order: ASG state → launch template → health checks → scaling policy → CloudWatch → IAM

---

## launch template

```bash
# create launch template
aws ec2 create-launch-template \
  --launch-template-name cloudops-lt \
  --version-description "v1" \
  --launch-template-data '{
    "ImageId": "<ami-id>",
    "InstanceType": "t2.micro",
    "IamInstanceProfile": {"Name": "ec2-ssm-profile"},
    "SecurityGroupIds": ["<sg-id>"],
    "UserData": "<base64-encoded-script>"
  }'

# list launch templates
aws ec2 describe-launch-templates \
  --query 'LaunchTemplates[].{Name:LaunchTemplateName,Id:LaunchTemplateId,Version:DefaultVersionNumber}' \
  --output table

# inspect launch template (default version)
aws ec2 describe-launch-template-versions \
  --launch-template-name cloudops-lt \
  --versions '$Default'
```

---

## create ASG

```bash
# create ASG across multiple subnets (multi-AZ)
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name cloudops-asg \
  --launch-template LaunchTemplateName=cloudops-lt,Version='$Latest' \
  --min-size 1 \
  --max-size 4 \
  --desired-capacity 2 \
  --vpc-zone-identifier "<subnet-az-a>,<subnet-az-b>" \
  --target-group-arns <tg-arn> \
  --health-check-type ELB \
  --health-check-grace-period 120

# note:
# health-check-type ELB is required for app-level replacement
# grace period gives instance time to pass initial health check
```

---

## inspect ASG

```bash
# full ASG details
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-asg

# compact view
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-asg \
  --query 'AutoScalingGroups[].{Min:MinSize,Max:MaxSize,Desired:DesiredCapacity,Instances:Instances[].InstanceId,Status:Status}' \
  --output table

# instance state within ASG
aws autoscaling describe-auto-scaling-instances \
  --query 'AutoScalingInstances[?AutoScalingGroupName==`cloudops-asg`]'

# scaling activity log
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name cloudops-asg
```

---

## scaling policies

```bash
# target tracking — CPU at 50%
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name cloudops-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 50.0
  }'

# target tracking — requests per target (ALB)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name cloudops-asg \
  --policy-name alb-request-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget",
      "ResourceLabel": "<alb-resource-label>/<tg-resource-label>"
    },
    "TargetValue": 1000.0
  }'

# list scaling policies
aws autoscaling describe-policies \
  --auto-scaling-group-name cloudops-asg \
  --query 'ScalingPolicies[].{Name:PolicyName,Type:PolicyType,Status:PolicyStatus}' \
  --output table
```

---

## manual scaling

```bash
# set desired capacity directly
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name cloudops-asg \
  --desired-capacity 3 \
  --honor-cooldown

# update min/max
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name cloudops-asg \
  --min-size 2 \
  --max-size 6
```

---

## instance health

```bash
# mark instance unhealthy (simulate failure)
aws autoscaling set-instance-health \
  --instance-id <instance-id> \
  --health-status Unhealthy

# confirm replacement triggered
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name cloudops-asg

# ALB target health (what the LB sees)
aws elbv2 describe-target-health \
  --target-group-arn <tg-arn>
```

---

## instance protection

```bash
# protect instance from scale-in termination
aws autoscaling set-instance-protection \
  --instance-ids <instance-id> \
  --auto-scaling-group-name cloudops-asg \
  --protected-from-scale-in

# remove protection
aws autoscaling set-instance-protection \
  --instance-ids <instance-id> \
  --auto-scaling-group-name cloudops-asg \
  --no-protected-from-scale-in
```

---

## cloudwatch integration

```bash
# ASG-level metrics (require explicit enable)
aws autoscaling enable-metrics-collection \
  --auto-scaling-group-name cloudops-asg \
  --granularity "1Minute"

# list available ASG metrics
aws autoscaling describe-metric-collection-types

# useful metrics once enabled:
# GroupMinSize, GroupMaxSize, GroupDesiredCapacity
# GroupInServiceInstances
# GroupPendingInstances
# GroupTerminatingInstances
```

---

## Scenario: ASG Not Replacing Unhealthy Instance

**Trigger:** Mark instance unhealthy or stop app (with ELB health check type set).

```bash
aws autoscaling set-instance-health \
  --instance-id <instance-id> \
  --health-status Unhealthy
```

**Detection:** Instance count in InService drops. Scaling activity log shows replacement.

```bash
aws autoscaling describe-auto-scaling-instances
aws autoscaling describe-scaling-activities --auto-scaling-group-name cloudops-asg
```

**Investigation:**
- Is health check type ELB (not just EC2)?
- Is health check grace period long enough for instance to start?
- Is the launch template still valid (AMI exists, SG valid)?

**Resolution:** Fix health check type or grace period. Validate launch template.

**Lesson:** EC2 health check only detects instance failure — ELB health check detects application failure.

---

## Scenario: Scale-Out Not Triggering

**Trigger:** Generate load, confirm CPU above target, but ASG does not add capacity.

**Detection:** CloudWatch metrics show breach. Desired capacity unchanged.

```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-asg \
  --query 'AutoScalingGroups[].{Desired:DesiredCapacity,Max:MaxSize}'

aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name cloudops-asg
```

**Investigation:**
- Is desired already at max?
- Is cooldown active (check activity log)?
- Is the scaling policy attached and enabled?
- Are ASG-level metrics enabled?

```bash
aws autoscaling describe-policies \
  --auto-scaling-group-name cloudops-asg
```

**Resolution:** Increase max size, wait for cooldown, or fix policy configuration.

**Lesson:** ASG cannot scale beyond max. Always check max before suspecting the policy.

---

## Scenario: New Instances Launch but Stay Unhealthy

**Trigger:** Launch template references bad AMI, misconfigured user data, or broken SG.

**Detection:** ASG keeps launching and terminating instances. Activity log shows rapid cycling.

```bash
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name cloudops-asg

aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

**Investigation:**
- Does the AMI still exist?
- Is user data bootstrapping correctly? (check console output)
- Does SG allow ALB health check traffic?

```bash
aws ec2 get-console-output --instance-id <instance-id>
aws ec2 describe-launch-template-versions --launch-template-name cloudops-lt
```

**Resolution:** Fix launch template. Validate by launching a standalone instance first.

**Lesson:** Validate launch template changes with a manual EC2 launch before attaching to ASG.

---

## Scenario: Scale-In Terminates Wrong Instance

**Trigger:** Default termination policy removes an instance you want to keep.

**Detection:** Active session or in-flight request terminated unexpectedly.

**Investigation:**
- What is the termination policy set to?
- Is instance protection enabled on critical instances?

```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names cloudops-asg \
  --query 'AutoScalingGroups[].TerminationPolicies'
```

**Resolution:** Enable instance protection, or use lifecycle hooks to drain before termination.

**Lesson:** Default policy is OldestInstance — understand your termination policy before it matters.

---

## Scenario: ASG Not Rebalancing Across AZs

**Trigger:** One AZ loses capacity or instances accumulate in a single AZ.

**Detection:** Describe instances shows uneven AZ distribution.

```bash
aws autoscaling describe-auto-scaling-instances \
  --query 'AutoScalingInstances[?AutoScalingGroupName==`cloudops-asg`].AvailabilityZone'
```

**Investigation:**
- Are multiple subnets configured?
- Is the AZ available?
- Is AvailabilityZoneRebalancing enabled?

**Resolution:** Add subnets across AZs, or trigger rebalance manually by updating desired capacity.

**Lesson:** Rebalancing is automatic but requires valid capacity in each AZ. AZ impairment will pin instances.

---

## quick checks

```bash
# ASG not scaling:
# → at max? cooling down? policy attached?

# instances launching and terminating:
# → launch template valid?
# → health check grace period too short?
# → SG blocking ALB health check?

# scale-in removing wrong instance:
# → termination policy?
# → instance protection enabled?

# no metrics visible:
# → metrics collection enabled on ASG?
# → correct namespace (AWS/AutoScaling)?
```

---

## reminders

```bash
# min / max / desired — know all three before touching anything
# ELB health check type required for app-level replacement
# target tracking creates its own CloudWatch alarms — don't duplicate manually
# cooldown ≠ warmup — understand the difference
# validate launch templates standalone before attaching to ASG
# ASG activity log is your primary debugging source
```
