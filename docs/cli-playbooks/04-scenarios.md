# CloudOps Scenarios

Focus: simulate incidents → detect → investigate → resolve → learn

---

## scenario: service down

```bash
# trigger:
# stop application OR stop instance

aws ec2 stop-instances \
  --instance-ids <instance-id>

# detection:
# ALB shows unhealthy targets
# CloudWatch alarm triggers (if configured)

aws elbv2 describe-target-health \
  --target-group-arn <tg-arn>

aws cloudwatch describe-alarms

# investigation:
# is instance running?
# is it healthy?
# is it registered?

aws ec2 describe-instances \
  --instance-ids <instance-id>

aws ec2 describe-instance-status \
  --instance-ids <instance-id>

aws elbv2 describe-target-health \
  --target-group-arn <tg-arn>

# resolution:
# start instance OR fix app

aws ec2 start-instances \
  --instance-ids <instance-id>

# lesson:
# running != serving traffic
# ALB health is primary signal
```

---

## scenario: high cpu

```bash
# trigger:
# generate load manually (inside instance)

# detection:
# CloudWatch alarm OR metric spike

aws cloudwatch describe-alarms

aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=<instance-id> \
  --statistics Average Maximum \
  --start-time <start> \
  --end-time <end> \
  --period 300

# investigation:
# confirm instance still responsive

aws ec2 describe-instance-status \
  --instance-ids <instance-id>

# optional:
# connect via SSM and inspect processes

# resolution:
# reduce load OR restart service OR reboot

aws ec2 reboot-instances \
  --instance-ids <instance-id>

# lesson:
# metrics confirm behaviour, not guesswork
```

---

## scenario: instance not reachable

```bash
# trigger:
# break security group OR remove public access

# detection:
# cannot connect (SSM / app unreachable)

# investigation:

# is instance running?
aws ec2 describe-instances \
  --instance-ids <instance-id>

# health checks
aws ec2 describe-instance-status \
  --instance-ids <instance-id>

# security groups
aws ec2 describe-security-groups \
  --group-ids <sg-id>

# SSM visibility
aws ssm describe-instance-information

# resolution:
# fix security group OR restore access

# lesson:
# most connectivity issues = SG or routing
```

---

## scenario: target not receiving traffic

```bash
# trigger:
# deregister instance OR break health check

aws elbv2 deregister-targets \
  --target-group-arn <tg-arn> \
  --targets Id=<instance-id>

# detection:
# ALB shows unused / unhealthy

aws elbv2 describe-target-health \
  --target-group-arn <tg-arn>

# investigation:
# is instance registered?
# is health check passing?

aws elbv2 describe-target-health \
  --target-group-arn <tg-arn>

aws elbv2 describe-target-groups \
  --target-group-arns <tg-arn>

# resolution:
# re-register instance OR fix health endpoint

aws elbv2 register-targets \
  --target-group-arn <tg-arn> \
  --targets Id=<instance-id>

# lesson:
# traffic depends on health checks, not instance state
```

---

## scenario: iam access failure

```bash
# trigger:
# remove policy OR break role

# detection:
# AccessDenied errors

# investigation:

# who am I?
aws sts get-caller-identity

# inspect role policies
aws iam list-attached-role-policies \
  --role-name ec2-ssm-role

# simulate permission
aws iam simulate-principal-policy \
  --policy-source-arn <role-arn> \
  --action-names ssm:StartSession

# resolution:
# reattach policy OR fix permissions

aws iam attach-role-policy \
  --role-name ec2-ssm-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# lesson:
# IAM issues surface as service failures
```

---

## expansion ideas

```bash
# add more scenarios as lab grows:

# disk full
# memory pressure
# broken health endpoint
# SNS not delivering alerts
# CloudWatch alarm misconfigured
# SSM not connecting
# multi-instance / load balancing issues
```

---

## reminders

```bash
# detect → investigate → resolve → learn
# always confirm with CLI, not assumptions
# start at ALB (traffic) → EC2 → IAM → CloudWatch
# practice repeatedly until flow is natural
```