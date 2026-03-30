# Foundations and IAM CLI Playbook

Focus: identity, roles, instance profiles, and access validation

---

## identity

```bash
# who am I (account / ARN)
aws sts get-caller-identity

# list available CLI profiles (local)
aws configure list-profiles

# current CLI config
aws configure list
```

---

## roles

```bash
# list roles
aws iam list-roles \
  --query 'Roles[].RoleName' \
  --output table

# inspect role (trust policy)
aws iam get-role \
  --role-name ec2-ssm-role

# create role (trust policy must exist locally)
aws iam create-role \
  --role-name ec2-ssm-role \
  --assume-role-policy-document file://trust-policy.json

# attach managed policy (SSM access)
aws iam attach-role-policy \
  --role-name ec2-ssm-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

---

## policies

```bash
# managed policies attached to role
aws iam list-attached-role-policies \
  --role-name ec2-ssm-role

# inline policies (if any)
aws iam list-role-policies \
  --role-name ec2-ssm-role

# inspect inline policy
aws iam get-role-policy \
  --role-name ec2-ssm-role \
  --policy-name <policy-name>
```

---

## instance profile

```bash
# create instance profile
aws iam create-instance-profile \
  --instance-profile-name ec2-ssm-profile

# attach role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name ec2-ssm-profile \
  --role-name ec2-ssm-role

# inspect instance profile (role linkage)
aws iam get-instance-profile \
  --instance-profile-name ec2-ssm-profile

# list instance profiles
aws iam list-instance-profiles \
  --query 'InstanceProfiles[].InstanceProfileName' \
  --output table
```

---

## simulate permissions

```bash
# test if role can perform action
aws iam simulate-principal-policy \
  --policy-source-arn <role-arn> \
  --action-names ssm:StartSession
```

---

## quick checks

```bash
# EC2 cannot access AWS:
# → instance profile attached?
# → correct role in profile?
# → role has required policies?
# → trust policy allows EC2?

# AccessDenied:
# → sts get-caller-identity
# → check attached policies
# → simulate action
```

---

## reminders

```bash
# EC2 uses instance profiles, not roles directly
# trust policy ≠ permissions policy
# implicit deny is default
# IAM issues often appear as service failures
```