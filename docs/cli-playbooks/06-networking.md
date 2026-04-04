# Networking CLI Playbook

Focus: VPC, subnets, security groups, route tables, internet access, and SSM connectivity

Troubleshoot in order: VPC → subnet → route table → IGW → security group → NACL → endpoint

---

## vpc

```bash
# list all VPCs
aws ec2 describe-vpcs \
  --query 'Vpcs[].{ID:VpcId,CIDR:CidrBlock,Default:IsDefault,Name:Tags[?Key==`Name`]|[0].Value}' \
  --output table

# inspect specific VPC
aws ec2 describe-vpcs \
  --vpc-ids <vpc-id>

# confirm default VPC exists
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[].{VpcId:VpcId,CIDR:CidrBlock}' \
  --output table
```

---

## subnets

```bash
# list all subnets
aws ec2 describe-subnets \
  --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Public:MapPublicIpOnLaunch,VPC:VpcId}' \
  --output table

# list subnets in a specific VPC
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Public:MapPublicIpOnLaunch}' \
  --output table

# list default subnets (one per AZ)
aws ec2 describe-subnets \
  --filters "Name=defaultForAz,Values=true" \
  --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}' \
  --output table

# check available IPs in subnet
aws ec2 describe-subnets \
  --subnet-ids <subnet-id> \
  --query 'Subnets[].{ID:SubnetId,AvailableIPs:AvailableIpAddressCount,CIDR:CidrBlock}'
```

---

## internet gateway

```bash
# list internet gateways
aws ec2 describe-internet-gateways \
  --query 'InternetGateways[].{ID:InternetGatewayId,VPC:Attachments[0].VpcId,State:Attachments[0].State}' \
  --output table

# confirm IGW is attached to VPC
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=<vpc-id>" \
  --query 'InternetGateways[].{ID:InternetGatewayId,State:Attachments[0].State}'
```

> No IGW = no public internet access. Common cause of "cannot reach instance" in new VPCs.

---

## route tables

```bash
# list route tables
aws ec2 describe-route-tables \
  --query 'RouteTables[].{ID:RouteTableId,VPC:VpcId,Main:Associations[?Main==`true`]|[0].Main}' \
  --output table

# inspect route table (check for default route to IGW)
aws ec2 describe-route-tables \
  --route-table-ids <rtb-id>

# show routes only
aws ec2 describe-route-tables \
  --route-table-ids <rtb-id> \
  --query 'RouteTables[].Routes[]'

# find route table associated with a subnet
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=<subnet-id>" \
  --query 'RouteTables[].{ID:RouteTableId,Routes:Routes}'
```

> A public subnet needs: route 0.0.0.0/0 → igw-xxxxxx
> If this route is missing, the instance has no path to the internet.

---

## security groups

```bash
# list security groups
aws ec2 describe-security-groups \
  --query 'SecurityGroups[].{ID:GroupId,Name:GroupName,VPC:VpcId}' \
  --output table

# inspect full rules for one group
aws ec2 describe-security-groups \
  --group-ids <sg-id>

# show inbound rules only
aws ec2 describe-security-groups \
  --group-ids <sg-id> \
  --query 'SecurityGroups[].IpPermissions'

# show outbound rules only
aws ec2 describe-security-groups \
  --group-ids <sg-id> \
  --query 'SecurityGroups[].IpPermissionsEgress'

# find security group by name
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=<name>" \
  --query 'SecurityGroups[].{ID:GroupId,Name:GroupName}'

# add inbound rule (e.g. allow HTTP from anywhere)
aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# add inbound rule from another security group (preferred pattern)
aws ec2 authorize-security-group-ingress \
  --group-id <instance-sg-id> \
  --protocol tcp \
  --port 80 \
  --source-group <alb-sg-id>

# remove inbound rule
aws ec2 revoke-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

> Security groups are stateful — return traffic is allowed automatically.
> Source-group rules are preferred over CIDR in production — they don't break on IP changes.

---

## network acls

```bash
# list NACLs
aws ec2 describe-network-acls \
  --query 'NetworkAcls[].{ID:NetworkAclId,VPC:VpcId,Default:IsDefault}' \
  --output table

# inspect NACL rules
aws ec2 describe-network-acls \
  --network-acl-ids <nacl-id>
```

> NACLs are stateless — return traffic must be explicitly allowed.
> Default NACL allows all traffic. Custom NACLs are deny-by-default.
> NACLs apply at subnet level, security groups at instance level.

---

## vpc endpoints (ssm without internet)

```bash
# list VPC endpoints
aws ec2 describe-vpc-endpoints \
  --query 'VpcEndpoints[].{ID:VpcEndpointId,Service:ServiceName,State:State,Type:VpcEndpointType}' \
  --output table

# check if SSM endpoints exist (required for SSM in private subnets)
aws ec2 describe-vpc-endpoints \
  --filters "Name=service-name,Values=com.amazonaws.ap-southeast-2.ssm" \
  --query 'VpcEndpoints[].{ID:VpcEndpointId,State:State}'

# SSM requires three endpoints for full functionality:
# com.amazonaws.<region>.ssm
# com.amazonaws.<region>.ssmmessages
# com.amazonaws.<region>.ec2messages

# create SSM endpoint (interface type)
aws ec2 create-vpc-endpoint \
  --vpc-id <vpc-id> \
  --service-name com.amazonaws.ap-southeast-2.ssm \
  --vpc-endpoint-type Interface \
  --subnet-ids <subnet-id> \
  --security-group-ids <sg-id> \
  --private-dns-enabled
```

> Without VPC endpoints: instance needs outbound 443 to internet for SSM.
> With VPC endpoints: SSM works in fully private subnets with no internet access.

---

## connectivity troubleshooting

```bash
# reachability analyser (trace path between two resources)
aws ec2 start-network-insights-analysis \
  --network-insights-path-id <path-id>

# create a path first (e.g. from instance to internet gateway)
aws ec2 create-network-insights-path \
  --source <instance-id> \
  --destination <igw-id> \
  --protocol TCP

# get analysis result
aws ec2 describe-network-insights-analyses \
  --network-insights-analysis-ids <analysis-id> \
  --query 'NetworkInsightsAnalyses[].{Status:Status,Reachable:NetworkPathFound,ExplanationCodes:Explanations[].ExplanationCode}'
```

---

## quick checks

```bash
# instance has no internet access:
# → IGW attached to VPC?
# → route table has 0.0.0.0/0 → igw?
# → subnet has public IP auto-assign enabled?
# → security group allows outbound 443?

# SSM not connecting (private subnet):
# → VPC endpoints for ssm, ssmmessages, ec2messages exist?
# → endpoints are in correct subnet and SG allows 443?
# → instance profile has AmazonSSMManagedInstanceCore?

# ALB cannot reach instances:
# → instance SG allows inbound on app port from ALB SG?
# → instance is in a subnet routable from ALB?

# traffic allowed by SG but still blocked:
# → check NACL — it may have an explicit deny
# → NACLs are stateless — both inbound and outbound rules apply

# security group rule added but still failing:
# → rules take effect immediately — recheck ports and protocol
# → confirm you're inspecting the right SG (instance vs ALB)
```

---

## reminders

```bash
# security groups = stateful, instance level
# NACLs = stateless, subnet level — rarely touched but can silently block traffic
# route table controls where traffic goes — IGW required for internet
# VPC endpoints = private access to AWS services without internet
# for SSM in private subnets: need ssm + ssmmessages + ec2messages endpoints
# source-group SG rules preferred over CIDR in production
# reachability analyser is the fastest way to diagnose complex path issues
```
