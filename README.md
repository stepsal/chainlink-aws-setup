# chainlink-aws-setup
Automated Chainlink node setup guide for Amazon Web Services.
These instructions illustrate how to bootstrap a new EC2 instance and automatically provision a dockerized Chainlink node running on Ropsten testnet.
This configuration is for demo purposes only and should not be considered as a suitable node configuration for mainnet.

## Node Configuration

Parameter | Value | Description
--------- | ----- | ------------
EC2 Instance Type | t3.micro | 2x vCPUs + 1GB RAM
EC2 Image Id | ami-09f0b8b3e41191524 | Ubuntu 16.04 LTS Xenial
Image\OS Hardening | None | Ubuntu out of the box
Chainlink Node Type | Docker | 
Root Volume | EBS | 8 GB
Root Volume Encryption | No | My linkies are gone
Delete Instance on Termination | Disabled | Instance can only be deleted manually via the AWS console
Incoming Open Ports | 22 (0.0.0.0/0) | SSH Access restricted to .pem key-file login
Outgoing Open Ports | All (0.0.0.0/0) |
Failover Support | None |  No Chainlink Container, VPS or Volume redundancy.
Protocol/Port | HTTP:6688 | 
TLS/SSL | No | I dont love you enough
Web Access | http://localhost:6688 | Web login is configured via port forwarding from your local machine to the EC2 instance
External Adapters | None | 

## Pre-requisites

* You are running on Linux or Mac OS (Linux VM should work on Windows).

* You have signed up for a [Amazon Web Services](https://aws.amazon.com/) account.

* The [AWS CLI Tool](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) (and optionally git) has been installed on your host machine. The CLI Tool has been correctly [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html#cli-quick-configuration) with the output format set to JSON.


* You have a [Fiews.io](https://fiews.io/) or similar Ethereum node as a service (EaaS) account.

## Installation Steps

### Download the repo

With git installed:

```
git clone https://github.com/stepsal/chainlink-aws-setup.git && cd chainlink-aws-setup
```

Without git:

```
wget https://github.com/stepsal/chainlink-aws-setup/archive/master.zip && unzip master.zip && cd chainlink-aws-setup-master
```

### Prepare configuration

Modify the node_setup.bsh script in a text editor.
Replace the placeholders `ETHNODE_ADDRESS`, `WALLET_PASSWORD`, `API_USER` and `API_PASSWORD` with your [desired config](#configuration-variables).

Create a default security group for the chainlink node.

```
aws ec2 create-security-group --group-name chainlink-node --description "Chainlink Node Security Group"
```

Allow SSH access to the node on port 22. (PasswordAuthentication is disabled by default,
 access can only be obtained using the key pair created in the following step)

```
aws ec2 authorize-security-group-ingress --group-name chainlink-node --protocol tcp --port 22 --cidr 0.0.0.0/0
```

Create a new key-pair in your local home directory for connecting to the node.

```
aws ec2 create-key-pair --key-name chainlinknode-key --query "KeyMaterial" --output text > ~/chainlinknode-key.pem
```

Update the key-pair permissions
chmod 400 ~/chainlinknode-key.pem



### Provision the EC2 Instance 

Provision the EC2 Instance and automatically install the Chainlink node.
This command will return the <instance_id> of the newly created node.

```
aws ec2 run-instances \
    --image-id ami-09f0b8b3e41191524 \
    --count 1 --instance-type t3.micro \
    --key-name chainlinknode-key \
    --security-groups chainlink-node \
    --user-data file://node_setup.sh \
    --query "Instances[0].InstanceId"
```

Get the PublicDNSName of the node.
Substitute <instance_id> with the value returned from the previous step.

```
aws ec2 describe-instances --instance-id <instance_id> --query "Reservations[0].Instances[0].[PublicDnsName]"
```

Monitor the installation.
Substitute <PublicDNSName> with the value returned from the previous step.

```
ssh -i ~/chainlinknode-key.pem ubuntu@<PublicDNSName> "tail -f /var/log/cloud-init-output.log"
```

Type Yes to add te key to the known hosts file and you will be viewing the (short) installation .
Once you see "****Chainlink Node Installed Successfully******" in the output press ctrl+z to exit.


Configure port forwarding from port 6688 on your local machine to port 6688 on the node.
Substiture <PublicDNSName> in the command with your PublicDNSName.

```
ssh -L 6688:localhost:6688 -i ~/chainlinknode-key.pem ubuntu@<PublicDNSName>
```

You can now login to your (Ropsten) Chainlink node via thw url http://localhost:6688

On reboot the SSH port forwarding will be disabled, so you will have to re-run the command.
You can create an alias in your bash file to simplify things. Once again substitute your nodes <PublicDNSName>

```
echo "alias link_port='ssh -L 6688:localhost:6688 -i ~/chainlinknode-key.pem ubuntu@<PublicDNSName>'" >> ~/.bashrc
```

After a reboot you can just run ``link_port`` from the command line to setup port forwarding and connect to your node.


## Post-installation
If you dont want your passwords in plain text on the node you can remove the passwords file after installation. Make sure you have them written down and stored safely somewhere

```
ssh -i ~/chainlinknode-key.pem ubuntu@<PublicDNSName>  'rm -rf /var/chainlink-ropsten/.api /var/chainlink-ropsten/.password'
```

## Configuration variables

Variable | Description | Example
-------- | ----------- | -------
`ETHNODE_ADDRESS` | The WS endpoint URL for your Ethereum node. | Fiews: `wss://cl-ropsten.fiews.io/v1/yourapikey` LinkPool: `wss://ropsten-rpc.linkpool.io/ws`
`API_USER` | The email you want to use to sign in to your CL node. | `you@example.com`
`API_PASSWORD` | The password you want to use to sign in to your CL node. | `yourpassword123`
`WALLET_PASSWORD` | A (secure) password for your Ethereum wallet. | `U!^926*KmBqsj68RpcI$*!w9$YpSTJK!#T`





