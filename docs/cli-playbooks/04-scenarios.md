# CloudOps Scenarios

Focus: simulate incidents → detect → investigate → resolve → learn

Troubleshoot in order: DNS → ALB/listener → target group → instance/app → IAM → monitoring

---

## Scenario: DNS Not Resolving

**Trigger:** Remove Route 53 record or use wrong hosted zone / delegation.

**Detection:** Domain does not resolve. Client cannot reach service by name.

```bash
dig app.example.com
dig +short app.example.com
```

**Investigation:**
- Does DNS return NXDOMAIN or no answer?
- Is the domain pointing at the expected ALB?
- If using Route 53, confirm the hosted zone / record exists.

**Resolution:** Recreate correct record or fix delegation / alias target.

**Lesson:** If DNS fails, traffic never reaches the load balancer.

---

## Scenario: DNS Resolves to Wrong Destination

**Trigger:** Point DNS record to wrong ALB or stale endpoint.

**Detection:** Domain resolves, but traffic goes to wrong service or environment.

```bash
dig app.example.com
dig +short app.example.com

aws elbv2 describe-load-balancers
```

**Investigation:**
- Compare resolved DNS target with expected ALB DNS name.
- Confirm the record points to the correct load balancer.

**Resolution:** Update Route 53 record / alias target.

**Lesson:** DNS can succeed technically while still routing users to the wrong place.

---

## Scenario: Service Down

**Trigger:** Stop application or stop instance.

```bash
aws ec2 stop-instances --instance-ids <instance-id>
```

**Detection:** ALB shows unhealthy targets. CloudWatch alarm triggers (if configured).

```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn>
aws cloudwatch describe-alarms
```

**Investigation:** Is instance running? Is it healthy? Is it registered?

```bash
aws ec2 describe-instances --instance-ids <instance-id>
aws ec2 describe-instance-status --instance-ids <instance-id>
aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

**Resolution:** Start instance or fix app.

```bash
aws ec2 start-instances --instance-ids <instance-id>
```

**Lesson:** Running ≠ serving traffic. ALB health is the primary signal.

---

## Scenario: Listener Misrouting Traffic

**Trigger:** Forward listener to wrong target group or break listener rules.

**Detection:** ALB is reachable, but traffic does not reach expected backend.

```bash
aws elbv2 describe-listeners --load-balancer-arn <alb-arn>

# if using rules:
aws elbv2 describe-rules --listener-arn <listener-arn>
```

**Investigation:**
- Is the default action forwarding to the correct target group?
- Are path / host rules matching as expected?
- Is the listener using the correct port / protocol?

**Resolution:** Update listener default action or fix listener rules.

**Lesson:** Healthy targets don't help if listener logic sends traffic to the wrong place.

---

## Scenario: Target Not Receiving Traffic

**Trigger:** Deregister instance or break health check.

```bash
aws elbv2 deregister-targets \
  --target-group-arn <tg-arn> \
  --targets Id=<instance-id>
```

**Detection:** ALB shows unused / unhealthy target.

```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

**Investigation:** Is instance registered? Is health check passing?

```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn>
aws elbv2 describe-target-groups --target-group-arns <tg-arn>
```

**Resolution:** Re-register instance or fix health endpoint.

```bash
aws elbv2 register-targets \
  --target-group-arn <tg-arn> \
  --targets Id=<instance-id>
```

**Lesson:** Traffic depends on health checks, not just instance state.

---

## Scenario: ALB Cannot Reach Target

**Trigger:** Break instance security group rule. Remove inbound app port from ALB security group source.

**Detection:** Target becomes unhealthy. Instance may still be running.

```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn>
aws ec2 describe-security-groups --group-ids <instance-sg-id>
```

**Investigation:**
- Does instance SG allow app port from ALB security group?
- Is the application port open?
- Is the target group using the expected port?

```bash
aws elbv2 describe-target-groups --target-group-arns <tg-arn>
```

**Resolution:** Restore inbound rule from ALB security group to app port.

**Lesson:** Instance running ≠ reachable from ALB.

---

## Scenario: Health Check Path or Port Mismatch

**Trigger:** Set wrong health check path or port. Stop Nginx / app while instance remains running.

**Detection:** Target becomes unhealthy even though instance is up.

```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn>
aws elbv2 describe-target-groups --target-group-arns <tg-arn>
```

**Investigation:**
- Is health check path correct?
- Is app listening on expected port?
- Does the endpoint return 200?

**Resolution:** Fix health check path / port or restore application service.

**Lesson:** Target health reflects application behaviour, not only compute state.

---

## Scenario: High CPU

**Trigger:** Generate load manually inside instance.

**Detection:** CloudWatch alarm or metric spike.

```bash
aws cloudwatch describe-alarms

aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=<instance-id> \
  --statistics Average Maximum \
  --start-time <start> \
  --end-time <end> \
  --period 300
```

**Investigation:** Confirm instance still responsive. Optionally connect via SSM and inspect processes.

```bash
aws ec2 describe-instance-status --instance-ids <instance-id>
```

**Resolution:** Reduce load, restart service, or reboot instance.

```bash
aws ec2 reboot-instances --instance-ids <instance-id>
```

**Lesson:** Metrics confirm behaviour, not guesswork.

---

## Scenario: Instance Not Reachable

**Trigger:** Break security group, remove public access, or break SSM path.

**Detection:** Cannot connect through SSM. App unreachable.

**Investigation:**

```bash
# is instance running?
aws ec2 describe-instances --instance-ids <instance-id>

# health checks
aws ec2 describe-instance-status --instance-ids <instance-id>

# security groups
aws ec2 describe-security-groups --group-ids <sg-id>

# SSM visibility
aws ssm describe-instance-information
```

**Resolution:** Fix security group, restore SSM / network access.

**Lesson:** Most connectivity issues come from SG, IAM, SSM agent, or routing problems.

---

## Scenario: IAM Access Failure

**Trigger:** Remove policy or break role permissions.

**Detection:** AccessDenied errors. SSM or other AWS operations fail.

**Investigation:**

```bash
# who am I?
aws sts get-caller-identity

# inspect role policies
aws iam list-attached-role-policies --role-name ec2-ssm-role

# simulate permission
aws iam simulate-principal-policy \
  --policy-source-arn <role-arn> \
  --action-names ssm:StartSession
```

**Resolution:** Reattach policy or fix permissions.

```bash
aws iam attach-role-policy \
  --role-name ec2-ssm-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

**Lesson:** IAM issues often appear as service failures rather than obvious auth problems.

---

## Scenario: CloudWatch Alarm Not Firing

**Trigger:** Misconfigure threshold, namespace, dimension, or evaluation period.

**Detection:** Service is degraded, but no alert is triggered.

```bash
aws cloudwatch describe-alarms
```

**Investigation:**
- Does the alarm watch the correct metric?
- Are dimensions correct?
- Does the threshold match current behaviour?

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=<instance-id> \
  --statistics Average Maximum \
  --start-time <start> \
  --end-time <end> \
  --period 300
```

**Resolution:** Correct alarm metric, dimension, threshold, or evaluation settings.

**Lesson:** Missing alerts can be as dangerous as the incident itself.

---

## Scenario: SNS Not Delivering Alerts

**Trigger:** Use unconfirmed subscription or wrong topic / policy setup.

**Detection:** CloudWatch alarm changes state, but no notification received.

```bash
aws sns list-topics
aws sns list-subscriptions-by-topic --topic-arn <topic-arn>
```

**Investigation:**
- Is the subscription confirmed?
- Is CloudWatch alarm linked to the expected SNS topic?

```bash
aws cloudwatch describe-alarms
```

**Resolution:** Confirm subscription or attach correct SNS topic to alarm actions.

**Lesson:** Monitoring is incomplete if alert delivery path is broken.

---

## Expansion Ideas

- Disk full
- Memory pressure
- Nginx service stopped
- Wrong app port
- Multi-instance imbalance (one healthy / one unhealthy)
- Autoscaling response checks
- Route 53 failover / health check scenarios

---

## Reminders

- Detect → investigate → resolve → learn
- Always confirm with CLI, not assumptions
- Troubleshoot in order: DNS → ALB/listener → target group → instance/app → IAM → monitoring
- Practice repeatedly until the flow is natural
