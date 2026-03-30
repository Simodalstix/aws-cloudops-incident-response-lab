# ALB / ELB Operations CLI Playbook

Focus: traffic routing, target health, service availability

---

## Load Balancers

    # list ALBs
    aws elbv2 describe-load-balancers

    # specific ALB
    aws elbv2 describe-load-balancers \
      --names my-alb

---

## Listeners

    # ports / protocols / rules (requires ALB ARN)
    aws elbv2 describe-listeners \
      --load-balancer-arn <alb-arn>

---

## Target Groups

    # list target groups
    aws elbv2 describe-target-groups

    # specific target group
    aws elbv2 describe-target-groups \
      --names my-tg

---

## Target Health (critical)

    # health per target (primary check when service is down)
    aws elbv2 describe-target-health \
      --target-group-arn <tg-arn>

states:
    healthy     → receiving traffic
    unhealthy   → failing checks
    initial     → warming up
    unused      → not registered

---

## Registration

    # add instance to target group
    aws elbv2 register-targets \
      --target-group-arn <tg-arn> \
      --targets Id=<instance-id>

    # remove instance
    aws elbv2 deregister-targets \
      --target-group-arn <tg-arn> \
      --targets Id=<instance-id>

---

## Health Check Config

    # verify health check path / port / protocol
    aws elbv2 describe-target-groups \
      --target-group-arns <tg-arn>

---

## Quick Flow

    # service down?
    # → check target health first

    # unhealthy:
    # → go EC2 (app / port / instance issue)

    # healthy:
    # → check listener rules / app logic

---

## Key Reminders

    # ALB → target group → instance
    # health checks control routing
    # running instance != serving traffic
