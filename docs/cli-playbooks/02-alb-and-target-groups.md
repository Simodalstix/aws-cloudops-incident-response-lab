# ALB and Target Groups CLI Playbook

Focus: create, inspect, and operate traffic routing + health

---

## target group

```bash
# create target group
aws elbv2 create-target-group \
  --name my-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id <vpc-id>

# list target groups
aws elbv2 describe-target-groups

# inspect target group
aws elbv2 describe-target-groups \
  --names my-tg
```

---

## load balancer

```bash
# create ALB
aws elbv2 create-load-balancer \
  --name my-alb \
  --subnets <subnet-1> <subnet-2> \
  --security-groups <sg-id>

# list ALBs
aws elbv2 describe-load-balancers

# inspect ALB
aws elbv2 describe-load-balancers \
  --names my-alb
```

---

## listener

```bash
# create listener (forward to target group)
aws elbv2 create-listener \
  --load-balancer-arn <alb-arn> \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=<tg-arn>

# inspect listeners
aws elbv2 describe-listeners \
  --load-balancer-arn <alb-arn>
```

---

## registration

```bash
# register instance
aws elbv2 register-targets \
  --target-group-arn <tg-arn> \
  --targets Id=<instance-id>

# deregister instance
aws elbv2 deregister-targets \
  --target-group-arn <tg-arn> \
  --targets Id=<instance-id>
```

---

## target health (critical)

```bash
# check health per target
aws elbv2 describe-target-health \
  --target-group-arn <tg-arn>
```

# states:
# healthy     → receiving traffic
# unhealthy   → failing checks
# initial     → warming up
# unused      → not registered

---

## quick checks

```bash
# service down:
# → check target health first

# unhealthy:
# → EC2 issue (app / port / instance)

# healthy:
# → listener / routing / app issue

# no traffic:
# → not registered?
# → wrong target group?
# → listener misconfigured?
```

---

## reminders

```bash
# ALB → target group → instance
# health checks control routing
# instance can be running but not serving traffic
```