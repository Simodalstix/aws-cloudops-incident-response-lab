# ALB / ELB Operations CLI Playbook

Focus: traffic routing, target health, service availability

---

## load balancers

# list ALBs
aws elbv2 describe-load-balancers

# specific ALB
aws elbv2 describe-load-balancers \
  --names my-alb

---

## listeners

# ports / protocols / rules
# requires load balancer ARN (from describe-load-balancers)
aws elbv2 describe-listeners \
  --load-balancer-arn <alb-arn>

---

## target groups

# list target groups
aws elbv2 describe-target-groups

# specific target group
aws elbv2 describe-target-groups \
  --names my-tg

---

## target health (critical)

# health status per target
aws elbv2 describe-target-health \
  --target-group-arn <tg-arn>

# states:
# healthy → receiving traffic
# unhealthy → failing checks
# initial → warming up
# unused → not registered

---

## registration

# add instance to target group
aws elbv2 register-targets \
  --target-group-arn <tg-arn> \
  --targets Id=<instance-id>

# remove instance from target group
aws elbv2 deregister-targets \
  --target-group-arn <tg-arn> \
  --targets Id=<instance-id>

---

## health check config

# verify path / port / protocol
aws elbv2 describe-target-groups \
  --target-group-arns <tg-arn>

---

## quick patterns

service down:
- check target-health first
- unhealthy → EC2 issue
- healthy → app / listener issue

targets unhealthy:
- app not running
- wrong port
- bad health check path
- security group blocking ALB

no traffic:
- instance not registered
- wrong target group
- listener misconfigured

---

## reminders

- ALB → target group → instance
- health checks control traffic flow
- "running instance" ≠ "serving traffic"
