# VPC Build from Scratch — CLI Playbook

Focus: build a production-style VPC with public/private subnets, IGW, NAT Gateway, route tables, and security groups

Region: ap-southeast-2 — follow top to bottom, each section builds on the last

---

## design decisions

| Decision | Choice | Why |
|---|---|---|
| CIDR | 10.0.0.0/16 | RFC1918, 65k IPs, doesn't conflict with 172.31 (old default) |
| AZs | 2a, 2b, 2c | Full AZ coverage for ap-southeast-2 |
| Public subnets | /24 each | 251 usable IPs — enough for ALBs, NAT, bastion |
| Private subnets | /24 each | Workloads live here — scale up if needed |
| NAT Gateway | 1 (in 2a) | Cost-conscious for lab; production uses one per AZ |
| DNS | Enabled | Required for SSM endpoints and service discovery |

```bash
# IP layout
10.0.0.0/16
├── Public
│   ├── 10.0.0.0/24   ap-southeast-2a
│   ├── 10.0.1.0/24   ap-southeast-2b
│   └── 10.0.2.0/24   ap-southeast-2c
└── Private
    ├── 10.0.10.0/24  ap-southeast-2a
    ├── 10.0.11.0/24  ap-southeast-2b
    └── 10.0.12.0/24  ap-southeast-2c
```

---

## step 1 — create the vpc

```bash
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[
    {Key=Name,Value=cloudops-lab-vpc},
    {Key=Env,Value=lab},
    {Key=ManagedBy,Value=cli}
  ]'
```

> Output includes VpcId — save it, you'll use it in every subsequent command.

Enable DNS resolution and hostnames (required for SSM and service discovery):

```bash
aws ec2 modify-vpc-attribute \
  --vpc-id <vpc-id> \
  --enable-dns-support

aws ec2 modify-vpc-attribute \
  --vpc-id <vpc-id> \
  --enable-dns-hostnames
```

Check:

```bash
aws ec2 describe-vpcs \
  --vpc-ids <vpc-id> \
  --query 'Vpcs[].{ID:VpcId,CIDR:CidrBlock,DNS:EnableDnsSupport,Hostnames:EnableDnsHostnames,State:State}'
```

---

## step 2 — create subnets

**Public subnets:**

```bash
# 2a
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 10.0.0.0/24 \
  --availability-zone ap-southeast-2a \
  --tag-specifications 'ResourceType=subnet,Tags=[
    {Key=Name,Value=cloudops-lab-public-2a},
    {Key=Tier,Value=public},
    {Key=AZ,Value=ap-southeast-2a}
  ]'

# 2b
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-southeast-2b \
  --tag-specifications 'ResourceType=subnet,Tags=[
    {Key=Name,Value=cloudops-lab-public-2b},
    {Key=Tier,Value=public},
    {Key=AZ,Value=ap-southeast-2b}
  ]'

# 2c
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-southeast-2c \
  --tag-specifications 'ResourceType=subnet,Tags=[
    {Key=Name,Value=cloudops-lab-public-2c},
    {Key=Tier,Value=public},
    {Key=AZ,Value=ap-southeast-2c}
  ]'
```

**Private subnets:**

```bash
# 2a
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 10.0.10.0/24 \
  --availability-zone ap-southeast-2a \
  --tag-specifications 'ResourceType=subnet,Tags=[
    {Key=Name,Value=cloudops-lab-private-2a},
    {Key=Tier,Value=private},
    {Key=AZ,Value=ap-southeast-2a}
  ]'

# 2b
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 10.0.11.0/24 \
  --availability-zone ap-southeast-2b \
  --tag-specifications 'ResourceType=subnet,Tags=[
    {Key=Name,Value=cloudops-lab-private-2b},
    {Key=Tier,Value=private},
    {Key=AZ,Value=ap-southeast-2b}
  ]'

# 2c
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 10.0.12.0/24 \
  --availability-zone ap-southeast-2c \
  --tag-specifications 'ResourceType=subnet,Tags=[
    {Key=Name,Value=cloudops-lab-private-2c},
    {Key=Tier,Value=private},
    {Key=AZ,Value=ap-southeast-2c}
  ]'
```

Enable auto-assign public IP on public subnets:

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id <public-2a-subnet-id> \
  --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
  --subnet-id <public-2b-subnet-id> \
  --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
  --subnet-id <public-2c-subnet-id> \
  --map-public-ip-on-launch
```

> Do NOT enable this on private subnets — instances there should never have public IPs.

Check:

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'Subnets[].{Name:Tags[?Key==`Name`]|[0].Value,ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,PublicIP:MapPublicIpOnLaunch}' \
  --output table
```

> Expected: 3 subnets with PublicIP=True, 3 with PublicIP=False.

---

## step 3 — internet gateway

```bash
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[
    {Key=Name,Value=cloudops-lab-igw}
  ]'

# attach to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id <igw-id> \
  --vpc-id <vpc-id>
```

Check:

```bash
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=<vpc-id>" \
  --query 'InternetGateways[].{ID:InternetGatewayId,State:Attachments[0].State}'
```

> Expected: State=available.

---

## step 4 — nat gateway

NAT Gateway requires an Elastic IP and must sit in a public subnet.

```bash
# allocate EIP
aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[
    {Key=Name,Value=cloudops-lab-nat-eip}
  ]'

# create NAT Gateway in public-2a (use AllocationId from above)
aws ec2 create-nat-gateway \
  --subnet-id <public-2a-subnet-id> \
  --allocation-id <eip-allocation-id> \
  --tag-specifications 'ResourceType=natgateway,Tags=[
    {Key=Name,Value=cloudops-lab-nat}
  ]'
```

> NAT Gateway takes 1-2 minutes to reach available state — check before proceeding.

Check:

```bash
aws ec2 describe-nat-gateways \
  --filter "Name=vpc-id,Values=<vpc-id>" \
  --query 'NatGateways[].{ID:NatGatewayId,State:State,Subnet:SubnetId,EIP:NatGatewayAddresses[0].PublicIp}'
```

> Wait until State=available before continuing.

---

## step 5 — route tables

Two route tables are needed: public (routes to IGW) and private (routes to NAT Gateway).

The main/default route table AWS creates is left unmodified and unassociated — production best practice.

**Public route table:**

```bash
aws ec2 create-route-table \
  --vpc-id <vpc-id> \
  --tag-specifications 'ResourceType=route-table,Tags=[
    {Key=Name,Value=cloudops-lab-rtb-public}
  ]'

# add default route to IGW
aws ec2 create-route \
  --route-table-id <public-rtb-id> \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id <igw-id>

# associate with all 3 public subnets
aws ec2 associate-route-table \
  --route-table-id <public-rtb-id> \
  --subnet-id <public-2a-subnet-id>

aws ec2 associate-route-table \
  --route-table-id <public-rtb-id> \
  --subnet-id <public-2b-subnet-id>

aws ec2 associate-route-table \
  --route-table-id <public-rtb-id> \
  --subnet-id <public-2c-subnet-id>
```

**Private route table:**

```bash
aws ec2 create-route-table \
  --vpc-id <vpc-id> \
  --tag-specifications 'ResourceType=route-table,Tags=[
    {Key=Name,Value=cloudops-lab-rtb-private}
  ]'

# add default route to NAT Gateway
aws ec2 create-route \
  --route-table-id <private-rtb-id> \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id <nat-gw-id>

# associate with all 3 private subnets
aws ec2 associate-route-table \
  --route-table-id <private-rtb-id> \
  --subnet-id <private-2a-subnet-id>

aws ec2 associate-route-table \
  --route-table-id <private-rtb-id> \
  --subnet-id <private-2b-subnet-id>

aws ec2 associate-route-table \
  --route-table-id <private-rtb-id> \
  --subnet-id <private-2c-subnet-id>
```

Check routes:

```bash
# public RT — should show local + 0.0.0.0/0 → igw
aws ec2 describe-route-tables \
  --route-table-ids <public-rtb-id> \
  --query 'RouteTables[].Routes[]'

# private RT — should show local + 0.0.0.0/0 → nat
aws ec2 describe-route-tables \
  --route-table-ids <private-rtb-id> \
  --query 'RouteTables[].Routes[]'
```

Check associations:

```bash
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'RouteTables[].{Name:Tags[?Key==`Name`]|[0].Value,ID:RouteTableId,Associations:Associations[].SubnetId}' \
  --output table
```

> Expected: public RT associated with 3 subnets, private RT associated with 3 subnets.

---

## step 6 — security groups

Create purpose-specific security groups — never use the default SG for workloads.

**Bastion / SSH access:**

```bash
aws ec2 create-security-group \
  --group-name cloudops-lab-bastion-sg \
  --description "Bastion host - SSH inbound" \
  --vpc-id <vpc-id> \
  --tag-specifications 'ResourceType=security-group,Tags=[
    {Key=Name,Value=cloudops-lab-bastion-sg}
  ]'

# allow SSH from your IP only
aws ec2 authorize-security-group-ingress \
  --group-id <bastion-sg-id> \
  --protocol tcp \
  --port 22 \
  --cidr <your-ip>/32
```

**Private instance (general workload):**

```bash
aws ec2 create-security-group \
  --group-name cloudops-lab-private-sg \
  --description "Private instances - inbound from bastion only" \
  --vpc-id <vpc-id> \
  --tag-specifications 'ResourceType=security-group,Tags=[
    {Key=Name,Value=cloudops-lab-private-sg}
  ]'

# allow SSH from bastion SG only (SG-to-SG reference, not CIDR)
aws ec2 authorize-security-group-ingress \
  --group-id <private-sg-id> \
  --protocol tcp \
  --port 22 \
  --source-group <bastion-sg-id>
```

> SG-to-SG references are preferred over CIDR in production — they don't break when IPs change and they express intent clearly.

Check:

```bash
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'SecurityGroups[].{Name:GroupName,ID:GroupId,InboundRules:IpPermissions}' \
  --output json
```

---

## final verification

Run after everything is built to confirm the full topology is correct:

```bash
# all subnets with tier and AZ
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'Subnets[].{Name:Tags[?Key==`Name`]|[0].Value,AZ:AvailabilityZone,CIDR:CidrBlock,Public:MapPublicIpOnLaunch}' \
  --output table

# IGW attached
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=<vpc-id>" \
  --query 'InternetGateways[].{ID:InternetGatewayId,State:Attachments[0].State}'

# NAT Gateway available
aws ec2 describe-nat-gateways \
  --filter "Name=vpc-id,Values=<vpc-id>" \
  --query 'NatGateways[].{ID:NatGatewayId,State:State,EIP:NatGatewayAddresses[0].PublicIp}'

# route tables and their subnet associations
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'RouteTables[].{Name:Tags[?Key==`Name`]|[0].Value,ID:RouteTableId,Subnets:Associations[].SubnetId,Routes:Routes[].DestinationCidrBlock}' \
  --output table

# security groups
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'SecurityGroups[].{Name:GroupName,ID:GroupId}' \
  --output table
```

---

## what you have when done

```
cloudops-lab-vpc (10.0.0.0/16)
├── cloudops-lab-igw (attached)
├── Public subnets (auto-assign public IP ON)
│   ├── cloudops-lab-public-2a  10.0.0.0/24
│   ├── cloudops-lab-public-2b  10.0.1.0/24
│   └── cloudops-lab-public-2c  10.0.2.0/24
│       └── cloudops-lab-rtb-public → 0.0.0.0/0 → igw
├── Private subnets (auto-assign public IP OFF)
│   ├── cloudops-lab-private-2a  10.0.10.0/24
│   ├── cloudops-lab-private-2b  10.0.11.0/24
│   └── cloudops-lab-private-2c  10.0.12.0/24
│       └── cloudops-lab-rtb-private → 0.0.0.0/0 → nat
├── cloudops-lab-nat (in public-2a, EIP attached)
├── cloudops-lab-bastion-sg (SSH from your IP)
└── cloudops-lab-private-sg (SSH from bastion SG only)
```

---

## common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| NAT GW in private subnet | Private instances can't reach internet | Move NAT to public subnet |
| No default route in public RT | Instances unreachable externally | Add 0.0.0.0/0 → igw |
| Subnets not associated with RT | Traffic hits main RT (no IGW route) | Explicitly associate subnets |
| Auto-assign public IP not enabled | Public subnet instances have no public IP | modify-subnet-attribute |
| DNS hostnames disabled | SSM and service discovery fail | modify-vpc-attribute both DNS flags |

---

## next steps

- Launch a bastion EC2 into a public subnet and SSH test
- Launch a private EC2 and verify it can reach the internet via NAT (curl, yum update)
- Add SSM VPC endpoints and remove the bastion — preferred pattern for production
- Layer in ALB, target groups, and auto scaling when ready
