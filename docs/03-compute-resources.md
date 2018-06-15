# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [compute zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones).

> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Introduction.html) (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom VPC network:

```
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.240.0.0/16 | \
  jq -r '.Vpc.VpcId')
```

```
aws ec2 create-tags \
  --resources ${VPC_ID} \
  --tags Key=Name,Value=kubernetes-the-hard-way
```

```
aws ec2 modify-vpc-attribute \
  --vpc-id ${VPC_ID} \
  --enable-dns-support '{"Value": true}'
```

```
aws ec2 modify-vpc-attribute \
  --vpc-id ${VPC_ID} \
  --enable-dns-hostnames '{"Value": true}'
```

A [subnet](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-hard-way` VPC network:

```
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.240.0.0/24 | \
  jq -r '.Subnet.SubnetId')
```

```
aws ec2 create-tags \
  --resources ${SUBNET_ID} \
  --tags Key=Name,Value=kubernetes
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Internet Gateways

```
INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway | \
  jq -r '.InternetGateway.InternetGatewayId')
```

```
aws ec2 create-tags \
  --resources ${INTERNET_GATEWAY_ID} \
  --tags Key=Name,Value=kubernetes
```

```
aws ec2 attach-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID} \
  --vpc-id ${VPC_ID}
```

### Route Tables

```
ROUTE_TABLE_ID=$(aws ec2 create-route-table \
  --vpc-id ${VPC_ID} | \
  jq -r '.RouteTable.RouteTableId')
```

```
aws ec2 create-tags \
  --resources ${ROUTE_TABLE_ID} \
  --tags Key=Name,Value=kubernetes
```

```
aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID}
```

```
aws ec2 create-route \
  --route-table-id ${ROUTE_TABLE_ID} \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id ${INTERNET_GATEWAY_ID}
```

### Firewall Rules

```
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name kubernetes \
  --description "Kubernetes security group" \
  --vpc-id ${VPC_ID} | \
  jq -r '.GroupId')
```

```
aws ec2 create-tags \
  --resources ${SECURITY_GROUP_ID} \
  --tags Key=Name,Value=kubernetes
```

Create a firewall rule that allows internal communication across all protocols:

```
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol all \
  --port 0-65535 \
  --cidr 10.240.0.0/24
```

Create a firewall rule that allows external SSH, ICMP, and HTTPS:

```
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

```
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 6443 \
  --cidr 0.0.0.0/0
```

```
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol icmp \
  --port="-1" \
  --cidr 0.0.0.0/0
```

> An [external load balancer](https://cloud.google.com/compute/docs/load-balancing/network/) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `kubernetes-the-hard-way` VPC network:

```
aws ec2 describe-security-groups --group-ids=${SECURITY_GROUP_ID} | \ 
jq -r '(.SecurityGroups[0].IpPermissions[] | [.IpRanges[].CidrIp, .ToPort, .IpProtocol]) | @tsv'
```

> output

```
0.0.0.0/0       6443    tcp
10.240.0.0/24           -1
0.0.0.0/0       22      tcp
0.0.0.0/0       -1      icmp
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
aws elb create-load-balancer \
  --load-balancer-name kubernetes-the-hard-way \
  --listeners "Protocol=TCP,LoadBalancerPort=6443,InstanceProtocol=TCP,InstancePort=6443" \
  --subnets ${SUBNET_ID} \
  --security-groups ${SECURITY_GROUP_ID}
```

Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

```
aws elb describe-load-balancers \
  --load-balancer-name kubernetes-the-hard-way | \
  jq -r '.LoadBalancerDescriptions[].DNSName'
```

> output

```
kubernetes-the-hard-way-804965586.us-west-2.elb.amazonaws.com
```

### Create Instance IAM Policies

```
cat > kubernetes-iam-role.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Principal": { "Service": "ec2.amazonaws.com"}, "Action": "sts:AssumeRole"}
  ]
}
EOF
```

```
aws iam create-role \
  --role-name kubernetes \
  --assume-role-policy-document file://kubernetes-iam-role.json
```

```
cat > kubernetes-iam-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": ["ec2:*"], "Resource": ["*"]},
    {"Effect": "Allow", "Action": ["elasticloadbalancing:*"], "Resource": ["*"]},
    {"Effect": "Allow", "Action": ["route53:*"], "Resource": ["*"]},
    {"Effect": "Allow", "Action": ["ecr:*"], "Resource": "*"}
  ]
}
EOF
```

```
aws iam put-role-policy \
  --role-name kubernetes \
  --policy-name kubernetes \
  --policy-document file://kubernetes-iam-policy.json
```

```
aws iam create-instance-profile \
  --instance-profile-name kubernetes 
```

```
aws iam add-role-to-instance-profile \
  --instance-profile-name kubernetes \
  --role-name kubernetes
```

### Chosing an Image

Use the [Ubuntu Amazon EC2 AMI Locator](https://cloud-images.ubuntu.com/locator/ec2/) to find the right image-id for your zone. This guide assumes the `us-west-2` zone.

```
IMAGE_ID="ami-04f8bb7c"
```


### Generate A SSH Key Pair

```
aws ec2 create-key-pair --key-name kubernetes | \
  jq -r '.KeyMaterial' > ~/.ssh/kubernetes_the_hard_way
```

```
chmod 600 ~/.ssh/kubernetes_the_hard_way
```

```
ssh-add ~/.ssh/kubernetes_the_hard_way 
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
CONTROLLER_0_INSTANCE_ID=$(aws ec2 run-instances \
  --associate-public-ip-address \
  --iam-instance-profile 'Name=kubernetes' \
  --image-id ${IMAGE_ID} \
  --count 1 \
  --key-name kubernetes \
  --security-group-ids ${SECURITY_GROUP_ID} \
  --instance-type t2.small \
  --private-ip-address 10.240.0.10 \
  --subnet-id ${SUBNET_ID} | \
  jq -r '.Instances[].InstanceId')
```

```
aws ec2 modify-instance-attribute \
  --instance-id ${CONTROLLER_0_INSTANCE_ID} \
  --no-source-dest-check
```

```
aws ec2 create-tags \
  --resources ${CONTROLLER_0_INSTANCE_ID} \
  --tags Key=Name,Value=ip-10-240-0-10
```

```
CONTROLLER_1_INSTANCE_ID=$(aws ec2 run-instances \
  --associate-public-ip-address \
  --iam-instance-profile 'Name=kubernetes' \
  --image-id ${IMAGE_ID} \
  --count 1 \
  --key-name kubernetes \
  --security-group-ids ${SECURITY_GROUP_ID} \
  --instance-type t2.small \
  --private-ip-address 10.240.0.11 \
  --subnet-id ${SUBNET_ID} | \
  jq -r '.Instances[].InstanceId')
```

```
aws ec2 modify-instance-attribute \
  --instance-id ${CONTROLLER_1_INSTANCE_ID} \
  --no-source-dest-check
```

```
aws ec2 create-tags \
  --resources ${CONTROLLER_1_INSTANCE_ID} \
  --tags Key=Name,Value=ip-10-240-0-11
```

```
CONTROLLER_2_INSTANCE_ID=$(aws ec2 run-instances \
  --associate-public-ip-address \
  --iam-instance-profile 'Name=kubernetes' \
  --image-id ${IMAGE_ID} \
  --count 1 \
  --key-name kubernetes \
  --security-group-ids ${SECURITY_GROUP_ID} \
  --instance-type t2.small \
  --private-ip-address 10.240.0.12 \
  --subnet-id ${SUBNET_ID} | \
  jq -r '.Instances[].InstanceId')
```

```
aws ec2 modify-instance-attribute \
  --instance-id ${CONTROLLER_2_INSTANCE_ID} \
  --no-source-dest-check
```

```
aws ec2 create-tags \
  --resources ${CONTROLLER_2_INSTANCE_ID} \
  --tags Key=Name,Value=ip-10-240-0-12
``` 

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `user-data` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
WORKER_0_INSTANCE_ID=$(aws ec2 run-instances \
  --associate-public-ip-address \
  --iam-instance-profile 'Name=kubernetes' \
  --image-id ${IMAGE_ID} \
  --count 1 \
  --key-name kubernetes \
  --security-group-ids ${SECURITY_GROUP_ID} \
  --instance-type t2.small \
  --private-ip-address 10.240.0.20 \
  --user-data 10.200.0.0/24 \
  --subnet-id ${SUBNET_ID} | \
  jq -r '.Instances[].InstanceId')
```

```
aws ec2 modify-instance-attribute \
  --instance-id ${WORKER_0_INSTANCE_ID} \
  --no-source-dest-check
```

```
aws ec2 create-tags \
  --resources ${WORKER_0_INSTANCE_ID} \
  --tags Key=Name,Value=ip-10-240-0-20
```

```
WORKER_1_INSTANCE_ID=$(aws ec2 run-instances \
  --associate-public-ip-address \
  --iam-instance-profile 'Name=kubernetes' \
  --image-id ${IMAGE_ID} \
  --count 1 \
  --key-name kubernetes \
  --security-group-ids ${SECURITY_GROUP_ID} \
  --instance-type t2.small \
  --private-ip-address 10.240.0.21 \
  --user-data 10.200.1.0/24 \
  --subnet-id ${SUBNET_ID} | \
  jq -r '.Instances[].InstanceId')
```

```
aws ec2 modify-instance-attribute \
  --instance-id ${WORKER_1_INSTANCE_ID} \
  --no-source-dest-check
```

```
aws ec2 create-tags \
  --resources ${WORKER_1_INSTANCE_ID} \
  --tags Key=Name,Value=ip-10-240-0-21
```

```
WORKER_2_INSTANCE_ID=$(aws ec2 run-instances \
  --associate-public-ip-address \
  --iam-instance-profile 'Name=kubernetes' \
  --image-id ${IMAGE_ID} \
  --count 1 \
  --key-name kubernetes \
  --security-group-ids ${SECURITY_GROUP_ID} \
  --instance-type t2.small \
  --private-ip-address 10.240.0.22 \
  --user-data 10.200.2.0/24 \
  --subnet-id ${SUBNET_ID} | \
  jq -r '.Instances[].InstanceId')
```

```
aws ec2 modify-instance-attribute \
  --instance-id ${WORKER_2_INSTANCE_ID} \
  --no-source-dest-check
```

```
aws ec2 create-tags \
  --resources ${WORKER_2_INSTANCE_ID} \
  --tags Key=Name,Value=ip-10-240-0-22
```

### Verification

List the compute instances in your default compute zone:

```
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" | \
  jq -j '.Reservations[].Instances[] | .InstanceId, "  ", .Placement.AvailabilityZone, "  ", .PrivateIpAddress, "  ", .PublicIpAddress, "\n"'
```

> output

```
i-0378673d23a0f85d1  us-west-2b  10.240.0.20  XX.XXX.XX.XXX
i-07e72695e8b10b470  us-west-2b  10.240.0.11  XX.XXX.XX.XXX
i-0403631bc22541d0b  us-west-2b  10.240.0.22  XX.XXX.XX.XXX
i-03b91b46203ecdcec  us-west-2b  10.240.0.10  XX.XXX.XX.XXX
i-0de274d9230c236ad  us-west-2b  10.240.0.21  XX.XXX.XX.XXX
i-06b52e9f1d38d3d42  us-west-2b  10.240.0.12  XX.XXX.XX.XXX
```

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. Once the virtual machines are created you'll be able to login into each machine using ssh like this:

```
WORKER_0_PUBLIC_IP_ADDRESS=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=ip-10-240-0-20" | \
    jq -j '.Reservations[].Instances[].PublicIpAddress')
```

> The instance public IP address can also be obtained from the EC2 console. Each node will be tagged with a unique name.

```
ssh ubuntu@${WORKER_0_PUBLIC_IP_ADDRESS}
```

You'll then be logged into the `ip-10-240-0-20` instance:

```
Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-1010-aws x86_64)

...

Last login: Sun May 13 14:34:27 2018 from XX.XXX.XXX.XX
```

Type `exit` at the prompt to exit the `ip-10-240-0-20` compute instance:

```
ubuntu@ip-10-240-0-20:~$ exit
```
> output

```
logout
Connection to XX.XXX.XXX.XXX closed
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
