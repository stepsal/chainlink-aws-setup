# chainlink-aws-setup
Automated Chainlink node setup guide for Amazon Web Services.
These instructions illustrate how to bootstrap a new EC2 instance and automatically provision a dockerized Chainlink node running on Ropsten testnet.
This configuration is for demo purposes only and should not be considered as suitable for mainnet.


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


Create a new security-group.

```
aws ec2 create-security-group --group-name chainlink-node --description "Chainlink Node Security Group"
```


Open port 22.

SSH PasswordAuthentication is disabled by default, access can only be obtained using a key-pair

```
aws ec2 authorize-security-group-ingress --group-name chainlink-node --protocol tcp --port 22 --cidr 0.0.0.0/0
```


Create a new key-pair.

The key-pair is created in your local home directory and used to SSH to the node.

```
aws ec2 create-key-pair --key-name chainlinknode-key --query "KeyMaterial" --output text > ~/chainlinknode-key.pem
```


Update the key-pair permissions
```
chmod 400 ~/chainlinknode-key.pem
```


### Provision the EC2 Instance 

Provision the EC2 Instance and automatically install the Chainlink node.
This command will return the INSTANCE_ID of the created node.

```
aws ec2 run-instances \
    --image-id ami-00035f41c82244dab \
    --count 1 --instance-type t3.micro \
    --key-name chainlinknode-key \
    --security-groups chainlink-node \
    --user-data file://node_setup.sh \
    --query "Instances[0].InstanceId"
```


Get the PublicDNSName.

Use your INSTANCE_ID to retrieve the PUBLIC_DNS_NAME

```
aws ec2 describe-instances --instance-id INSTANCE_ID --query "Reservations[0].Instances[0].[PublicDnsName]"
```


Connect to the node and Monitor the installation.

Substitute PUBLIC_DNS_NAME with the value returned from the previous step.
When prompted add the key to the known hosts file.
Once you see "Chainlink Node Installed Successfully" press ```ctrl+z``` to exit the SSH session.

```
ssh -i ~/chainlinknode-key.pem ubuntu@PUBLIC_DNS_NAME "tail -f /var/log/cloud-init-output.log"
```


Configure port forwarding.

Setup port forwarding from localhost port 6688 to port 6688 of the AWS instance.

```
ssh -N -L 6688:localhost:6688 -i ~/chainlinknode-key.pem ubuntu@PUBLIC_DNS_NAME
```
You can now login to your (Ropsten) Chainlink node via http://localhost:6688

On a shutdown SSH port forwarding will be lost and will have to re-enabled after you restart your machine
You can create a shortcut alias to simplify this. You can run ``link_port`` in the shell anytime to re-enable port-forwarding. Make sure to substitute PUBLIC_DNS_NAME
 

Backup your .bashrc file
```
 cp ~/.bashrc ~/.bashrc_backup
```
Create the link-port alias
```
echo "alias link_port='ssh -L 6688:localhost:6688 -i ~/chainlinknode-key.pem ubuntu@PUBLIC_DNS_NAME'" >> ~/.bashrc
```

## Post-installation

If you dont want to leave your passwords in plain text on the node you can remove the passwords file with the below command. 
Make sure to backup you API and Wallet passwords locally before excuting the step.

```
ssh -i ~/chainlinknode-key.pem ubuntu@PUBLIC_DNS_NAME 'rm -rf /var/chainlink-ropsten/.api /var/chainlink-ropsten/.password'
```

## Configuration variables

Variable | Description | Example
-------- | ----------- | -------
`ETHNODE_ADDRESS` | The WS endpoint URL for your Ethereum node. | Fiews: `wss://cl-ropsten.fiews.io/v1/yourapikey` LinkPool: `wss://ropsten-rpc.linkpool.io/ws`
`API_USER` | The email you want to use to sign in to your CL node. | `you@example.com`
`API_PASSWORD` | The password you want to use to sign in to your CL node. | `yourpassword123`
`WALLET_PASSWORD` | A (secure) password for your Ethereum wallet. | `U!^926*KmBqsj68RpcI$*!w9$YpSTJK!#T`


## Node Description

Parameter | Value | Description
--------- | ----- | ------------
EC2 Instance Type | t3.micro | 2x vCPUs + 1GB RAM
EC2 Image Id | ami-00035f41c82244dab| Ubuntu 18.04 LTS
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





