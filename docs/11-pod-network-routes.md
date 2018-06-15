# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network.

Print the internal IP address and Pod CIDR range for each worker instance:

```
for instance in ip-10-240-0-20 ip-10-240-0-21 ip-10-240-0-22; do
  aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" "Name=instance-state-name,Values=running" | \
    jq -j '.Reservations[].Instances[].PrivateIpAddress," "'

  INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${instance}" | \
    jq -j '.Reservations[].Instances[].InstanceId')

  aws ec2 describe-instance-attribute \
    --instance-id ${INSTANCE_ID} \
    --attribute userData --output text --query "UserData.Value" | \ 
    base64 --decode && echo "\n"
done
```

> output

```
10.240.0.20 10.200.0.0/24
10.240.0.21 10.200.1.0/24
10.240.0.22 10.200.2.0/24
```

## Routes

Create network routes for each worker instance:

```
ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=kubernetes" | \
  jq -r '.RouteTables[].RouteTableId')

WORKER_0_INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ip-10-240-0-20" | \
  jq -j '.Reservations[].Instances[].InstanceId')

aws ec2 create-route \
  --route-table-id ${ROUTE_TABLE_ID} \
  --destination-cidr-block 10.200.0.0/24 \
  --instance-id ${WORKER_0_INSTANCE_ID}

WORKER_1_INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ip-10-240-0-21" | \
  jq -j '.Reservations[].Instances[].InstanceId')

aws ec2 create-route \
  --route-table-id ${ROUTE_TABLE_ID} \
  --destination-cidr-block 10.200.1.0/24 \
  --instance-id ${WORKER_1_INSTANCE_ID}

WORKER_2_INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ip-10-240-0-22" | \
  jq -j '.Reservations[].Instances[].InstanceId')

aws ec2 create-route \
  --route-table-id ${ROUTE_TABLE_ID} \
  --destination-cidr-block 10.200.2.0/24 \
  --instance-id ${WORKER_2_INSTANCE_ID}
```

List the routes in the `kubernetes-the-hard-way` VPC network:

```
aws ec2 describe-route-tables --route-table-ids ${ROUTE_TABLE_ID} | \ 
  jq -j '.RouteTables[].Routes[] | .DestinationCidrBlock, " ", .NetworkInterfaceId // .GatewayId, " ", .State, "\n"' 
```

> output

```
10.200.0.0/24 eni-ac25962e active
10.200.1.0/24 eni-362093b4 active
10.200.2.0/24 eni-82229100 active
10.240.0.0/16 local active
0.0.0.0/0 igw-090e5d71 active
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
