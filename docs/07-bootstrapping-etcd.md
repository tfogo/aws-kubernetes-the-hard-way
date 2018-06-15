# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/coreos/etcd). In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

The commands in this lab must be run on each controller instance: `ip-10-240-0-10`, `ip-10-240-0-11`, and `ip-10-240-0-12`. Login to each controller instance using `ssh`:

```
CONTROLLER_0_PUBLIC_IP_ADDRESS=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ip-10-240-0-10" "Name=instance-state-name,Values=running" | \
  jq -j '.Reservations[].Instances[].PublicIpAddress')
    
ssh ubuntu@${CONTROLLER_0_PUBLIC_IP_ADDRESS}
```

```
CONTROLLER_1_PUBLIC_IP_ADDRESS=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ip-10-240-0-11" "Name=instance-state-name,Values=running" | \
  jq -j '.Reservations[].Instances[].PublicIpAddress')

ssh ubuntu@${CONTROLLER_1_PUBLIC_IP_ADDRESS}
```

```
CONTROLLER_2_PUBLIC_IP_ADDRESS=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ip-10-240-0-12" "Name=instance-state-name,Values=running" | \
  jq -j '.Reservations[].Instances[].PublicIpAddress')
    
ssh ubuntu@${CONTROLLER_2_PUBLIC_IP_ADDRESS}
```

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Bootstrapping an etcd Cluster Member

### Download and Install the etcd Binaries

Download the official etcd release binaries from the [coreos/etcd](https://github.com/coreos/etcd) GitHub project:

```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/coreos/etcd/releases/download/v3.3.5/etcd-v3.3.5-linux-amd64.tar.gz"
```

Extract and install the `etcd` server and the `etcdctl` command line utility:

```
{
  tar -xvf etcd-v3.3.5-linux-amd64.tar.gz
  sudo mv etcd-v3.3.5-linux-amd64/etcd* /usr/local/bin/
}
```

### Configure the etcd Server

```
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
}
```

The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address for the current compute instance:

```
INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
```

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance:

```
ETCD_NAME=$(hostname -s)
```

Create the `etcd.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster ip-10-240-0-10=https://10.240.0.10:2380,ip-10-240-0-11=https://10.240.0.11:2380,ip-10-240-0-12=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the etcd Server

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

> Remember to run the above commands on each controller node: `ip-10-240-0-10`, `ip-10-240-0-11`, and `ip-10-240-0-12`.

## Verification

List the etcd cluster members:

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> output

```
3a57933972cb5131, started, ip-10-240-0-12, https://10.240.0.12:2380, https://10.240.0.12:2379
f98dc20bce6225a0, started, ip-10-240-0-10, https://10.240.0.10:2380, https://10.240.0.10:2379
ffed16798470cab5, started, ip-10-240-0-11, https://10.240.0.11:2380, https://10.240.0.11:2379
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
