# CloudWatch and SNS CLI Playbook

Focus: metrics, alarms, alerting, and basic signal validation

---

## sns topic

```bash
# create topic
aws sns create-topic \
  --name cloudops-alerts

# list topics
aws sns list-topics
```

---

## sns subscription

```bash
# subscribe email endpoint
aws sns subscribe \
  --topic-arn <topic-arn> \
  --protocol email \
  --notification-endpoint you@example.com

# list subscriptions
aws sns list-subscriptions
```

---

## sns test

```bash
# publish test message
aws sns publish \
  --topic-arn <topic-arn> \
  --subject "cloudops test alert" \
  --message "test notification from aws cli"
```

---

## metrics

```bash
# list EC2 metrics
aws cloudwatch list-metrics \
  --namespace AWS/EC2

# list CPU metrics for one instance
aws cloudwatch list-metrics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=<instance-id>
```

---

## metric data

```bash
# CPU stats for last hour
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=<instance-id> \
  --statistics Average Maximum \
  --start-time 2026-03-30T09:00:00Z \
  --end-time 2026-03-30T10:00:00Z \
  --period 300
```

# note:
# adjust start / end time each run
# period is in seconds

---

## alarms

```bash
# create CPU alarm
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu-cloudops-lab \
  --alarm-description "High CPU on lab instance" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=<instance-id> \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions <topic-arn>

# list alarms
aws cloudwatch describe-alarms

# inspect one alarm
aws cloudwatch describe-alarms \
  --alarm-names high-cpu-cloudops-lab
```

---

## alarm state

```bash
# set alarm state manually (test only)
aws cloudwatch set-alarm-state \
  --alarm-name high-cpu-cloudops-lab \
  --state-value ALARM \
  --state-reason "manual test"
```

# states:
# OK
# ALARM
# INSUFFICIENT_DATA

---

## quick checks

```bash
# no alert received:
# → subscription confirmed?
# → correct topic ARN?
# → alarm-actions set?
# → alarm actually in ALARM state?

# app looks unhealthy:
# → check CPUUtilization first
# → check alarm state
# → then move to EC2 / ALB

# metric missing:
# → correct namespace?
# → correct dimension?
# → enough time range?
```

---

## reminders

```bash
# CloudWatch detects, SNS notifies
# alarm can exist without notifications wired correctly
# no data / bad dimensions often looks like "nothing is happening"
# use metrics to confirm suspicion before acting
```