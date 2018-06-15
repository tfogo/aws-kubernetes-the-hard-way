# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Virtual Machines

```
KUBERNETES_HOSTS=(controller0 controller1 controller2 etcd0 etcd1 etcd2 worker0 worker1 worker2)
```

```
for instance in ip-10-240-0-10 ip-10-240-0-11 ip-10-240-0-12 ip-10-240-0-20 ip-10-240-0-21 ip-10-240-0-22; do
  INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" | \
    jq -j '.Reservations[].Instances[].InstanceId')
    
  aws ec2 terminate-instances --instance-ids ${INSTANCE_ID}
done
```

## IAM

```
aws iam remove-role-from-instance-profile \
  --instance-profile-name kubernetes \
  --role-name kubernetes
```

```
aws iam delete-instance-profile \
  --instance-profile-name kubernetes
```

```
aws iam delete-role-policy \
  --role-name kubernetes \
  --policy-name kubernetes
```

```
aws iam delete-role --role-name kubernetes
```

## SSH Keys

```
aws ec2 delete-key-pair --key-name kubernetes
```

## Networking

Be sure to wait about a minute for all VMs to terminates to avoid the following errors:

```
An error occurred (DependencyViolation) when calling ...
```

Network resources cannot be deleted while VMs hold a reference to them.

### Load Balancers

```
aws elb delete-load-balancer \
  --load-balancer-name kubernetes
```

### Internet Gateways

```
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=kubernetes" | \
  jq -r '.Vpcs[].VpcId')
```

```
INTERNET_GATEWAY_ID=$(aws ec2 describe-internet-gateways \
  --filters "Name=tag:Name,Values=kubernetes" | \
  jq -r '.InternetGateways[].InternetGatewayId')
```

```
aws ec2 detach-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID} \
  --vpc-id ${VPC_ID}
```

```
aws ec2 delete-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID}
```

### Security Groups

```
SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
  --filters "Name=tag:Name,Values=kubernetes" | \
  jq -r '.SecurityGroups[].GroupId')
```

```
aws ec2 delete-security-group \
  --group-id ${SECURITY_GROUP_ID}
```

### Subnets

```
SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=kubernetes" | \
  jq -r '.Subnets[].SubnetId')
```

```
aws ec2 delete-subnet --subnet-id ${SUBNET_ID}
```

### Route Tables

```
ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=kubernetes" | \
  jq -r '.RouteTables[].RouteTableId')
```

```
aws ec2 delete-route-table --route-table-id ${ROUTE_TABLE_ID}
```

### VPC

```
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=kubernetes" | \
  jq -r '.Vpcs[].VpcId')
```

```
aws ec2 delete-vpc --vpc-id ${VPC_ID}
```
