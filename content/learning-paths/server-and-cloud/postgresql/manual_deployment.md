---
# User change
title: "Deploy a 3-node PostgreSQL cluster with two hot standby servers"

weight: 2 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy 3 node of PostgreSQL cluster

## Prerequisites

* An [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [AWS IAM authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)
* [Terraform](https://github.com/zachlas/arm-software-developers-ads/blob/main/content/install-tools/terraform.md)

## Generate Access keys, (access key ID and secret access key)

The installation of Terraform on your desktop or laptop needs to communicate with AWS. Thus, Terraform needs to be able to authenticate with AWS. For authentication, generate access keys (access key ID and secret access key). These access keys are used by Terraform for making programmatic calls to AWS via AWS CLI.

Go to **My Security Credentials**

![image](https://user-images.githubusercontent.com/92078754/217739255-cdbc372f-203c-45ee-b280-317eb4685447.png)

On Your **Security Credentials** page click on **create access keys**, (access key ID and secret access key)

![image](https://user-images.githubusercontent.com/87687468/190137925-c725359a-cdab-468f-8195-8cce9c1be0ae.png)

Copy the **Access Key ID** and **Secret Access Key** 

![image](https://user-images.githubusercontent.com/87687468/190138349-7cc0007c-def1-48b7-ad1e-4ee5b97f4b90.png)

## Generate key-pair(public key, private key) using ssh keygen

Before using Terraform, first generate the **key-pair** (public key, private key) using `ssh-keygen`. Then associate both public and private keys with AWS EC2 instances.

Generate the **key-pair** using the following command:

```console
ssh-keygen -t rsa -b 2048
```
       
By default, the above command will generate the public as well as private key at location **$HOME/.ssh**. You can override the end destination with a custom path.

Output when a key pair is generated:

![image](https://user-images.githubusercontent.com/92078754/217732720-96b77cd2-d434-4f1c-a231-0f1a0d4019a0.png)
      
**NOTE:** Use the public key task2-key.pub inside the Terraform file to provision/start the instance and private key task2-key to connect to the instance.

## Deploy EC2 instance via Terraform

After generating the public and private keys, we have to create an EC2 instance. Then we will push our public key to the **authorized_keys** folder in **~/.ssh**. We will also create a security group that opens inbound ports **22**(ssh) and **5432**(PSQL). Below is a Terraform file called **main.tf** performs the above process.


```console
// instance creation

provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXX"
  secret_key   = "AXXXXXXXXXXXXXXXXXX"
}
resource "aws_instance" "PSQL_TEST" {
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity.name]
  key_name = "task2-key"
  
  tags = {
    Name = "PSQL_TEST"
  }
}
resource "aws_instance" "replica-PSQL_TEST" {
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity.name]
  key_name = "task2-key"
 
  tags = {
    Name = "replica-PSQL_TEST"
  }
}
resource "aws_instance" "replica1-PSQL_TEST" {
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity.name]
  key_name = "task2-key"
  
  tags = {
    Name = "replica1-PSQL_TEST"
  }
}
resource "aws_default_vpc" "main" {
  tags = {
    Name = "main"
  }
}
resource "aws_security_group" "Terraformsecurity" {
  name        = "Terraformsecurity"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_default_vpc.main.id
  ingress {
    description      = "TLS from VPC"
    from_port        = 5432
    to_port          = 5432
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}
  ingress {
    description      = "TLS from VPC"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}
  ingress {
    description      = "TLS from VPC"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  tags = {
    Name = "Terraformsecurity"
  }
 }
 resource "aws_key_pair" "deployer" {
         key_name   = "task2-key"
         public_key = "ssh-rsaxxxxxxxxxxxxxx"
  }

output "Master_public_IP" {
  value = [aws_instance.PSQL_TEST.public_ip, aws_instance.replica-PSQL_TEST.public_ip, aws_instance.replica1-PSQL_TEST.public_ip]

}
```
**NOTE:-** Replace `public_key`, `access_key`, `secret_key`, and `key_name` with your values.

Now, use the below Terraform commands to deploy the `main.tf` file.

## Terraform Commands

#### Initialize Terraform

Run `terraform init` to initialize the Terraform deployment. This command is responsible for downloading all dependencies which are required for the AWS provider.

```console
terraform init
```

![Screenshot (320)](https://user-images.githubusercontent.com/92315883/213113408-91133eef-645c-44ed-9136-f48cce40e220.png)

#### Create a Terraform execution plan

Run `terraform plan` to create an execution plan.

```console
terraform plan
```
![image](https://user-images.githubusercontent.com/92078754/215394355-e4715e1f-95d9-4446-acdb-ab7116b1f34a.png)

**NOTE:** The **terraform plan** command is optional. You can directly run **terraform apply** command. But it is always better to check created resources.

#### Apply a Terraform execution plan

Run `terraform apply` to apply the execution plan to your cloud infrastructure. The below command creates all required infrastructure.

```console
terraform apply
```      

![image](https://user-images.githubusercontent.com/92078754/216567557-5b7b79ff-5726-4383-9165-91d13a03c961.png)

## Manual configuration of master-slave setup

Here are the three nodes deployed by Terraform.

**Primary node:** IP: 3.142.184.72

**Replica node:** IP: 18.218.199.25 (hot standby server that is read-only)

**Replica1 node:** IP: 3.16.21.58 (hot standby server that is read-only)

 **Install PostgreSQL Server**

The first step is to install PostgreSQL on the Primary and both the Replica nodes. 

**NOTE:** You need to install the same version of PostgreSQL on all three nodes for logical replication.

First log into your nodes via SSH `ssh ubuntu@{{ your_nodes_ip }}`. And then follow below command for Postgres installation.
```console
sudo apt-get update
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install postgresql-9.6  
```

#### Configure Primary Node

SSH to the primary node(3.142.184.72) and follow the steps below to make configuration changes.
 
```console
ssh ubuntu@{{ primary_node_ip }}
```
Next, you need to tweak the main configuration file **/etc/postgresql/9.6/main/postgresql.conf** using your editor.
With the file open, locate the `listen_addresses` directive. The directive specifies the host under which the PostgreSQL database server listens to connections. Uncomment the directive by removing the `#` symbol then replace localhost with `'*'` in single quotation marks as shown:

![image](https://user-images.githubusercontent.com/92078754/215722631-7ec6ac62-7726-4fee-821c-ad1149699efd.png)

Next, go to pg_hba.conf file in this location **/etc/postgresql/9.6/main/pg_hba.conf**. To access your instance using SSH change the address from `127.0.0.1/32` (localhost) to `0.0.0.0/0` to enable all IPv4 addresses and change the address of IPv6 from `::1/128` to `::/0` to enable all IPv6 address.

![image](https://user-images.githubusercontent.com/92078754/217788571-697413fe-141a-4266-8800-b6b6c82a7dbd.png) 

Next, log into the Postgres by the following commands.
```console
cd ~postgres/
sudo su postgres -c psql
```
Then run the following command to create the replication user and assign replication privileges. In this command, replication is the replication user while the password is the user’s password.

```console
CREATE ROLE replication WITH REPLICATION PASSWORD 'password' LOGIN;
```
![image](https://user-images.githubusercontent.com/92078754/215955679-50b6cb30-1f4e-4ca1-90d3-4758b1a69de7.png)

Then logout from the PostgreSQL prompt.

![image](https://user-images.githubusercontent.com/92078754/215955930-590628a4-463b-4090-b2d2-12defed9aeb0.png)

Next, need to stop the postgres by this command `sudo systemctl stop postgresql`

Next, locate the `wal_level` directive in the **/etc/postgresql/9.6/main/postgresql.conf file**, the setting specifies the amount of information to be written to the Write Ahead Log (WAL) file.
Uncomment the line and set it to hot_standby as shown below.

![image](https://user-images.githubusercontent.com/92078754/215723032-7e1486d7-8ac5-4eee-8be8-206c8a18eb24.png)

Next, locate the `max_wal_sender` and `wal_keep_segments`. These settings control the behavior of the built-in streaming replication feature. These parameters would be set on the primary server that is to send replication data to one or more standby servers.

![image](https://user-images.githubusercontent.com/92078754/215723543-ece14cf8-f235-4a47-8966-0d6cbcb9e7da.png)

Next, locate the `archive_mode` by default, it is set to off when set to on, it will store the backup of replicas. Also, add `archive_command` while storing the data.

![image](https://user-images.githubusercontent.com/92078754/217772707-5b8d51fc-ed75-46d3-9593-4b74e72d96e7.png)


These changes are required in this configuration file. Save the changes and exit.

Next, create an archive directory and grant permission to it by following below commands.
```console
sudo mkdir /var/lib/postgresql/9.6/archive
sudo chown postgres.postgres /var/lib/postgresql/9.6/archive
```
Next, access the **/etc/postgresql/9.6/main/pg_hba.conf** configuration file.
Append this line at the end of the configuration file. This allows the replica and replica1 **ip-adresses** to connect with the master node using replication.

![image](https://user-images.githubusercontent.com/92078754/216566702-892e09b8-ba53-4d9e-b8ba-aac5a68adfdc.png)

Save the changes and close this file. Then restart PostgreSQL service.

```console
sudo systemctl restart postgresql
```
#### Configure Replica Node

Before the replica node starts replicating data from the primary node, you need to create a copy of the primary node’s data directory to the replica’s data directory. To achieve this first, stop the PostgreSQL service on the replica node using below command.

```console
sudo systemctl stop postgresql 
```
Next, remove all files in the replica’s data directory in order to start on a clean state and make room for the primary node data directory.

```console
sudo rm -rf /var/lib/postgresql/9.6/main/*
```

Now run the pg_basebackup utility as shown to copy data from the primary node to the replica node using below command.

```console
pg_basebackup -h {{ host_server_ip }} -D /var/lib/postgresql/9.6/main/ -P -U {{ replication_user }}
```
![image](https://user-images.githubusercontent.com/92078754/217457056-08ace6cf-4608-4d2f-b969-186ace92fd65.png)

Now we must modify **/etc/postgresql/9.6/main/postgresql.conf** changed here as `hot_standby=off` to `hot_standby=on`.

![image](https://user-images.githubusercontent.com/92078754/215724525-3efb4088-2118-4ba9-9138-41b50f076a66.png)

Last we need to create a **recovery.conf** file in the data directory **/var/lib/postgresql/9.6/main/**. Else, replication will not happen.

Add the following code in the **recovery.conf** file.

```console
standby_mode = 'on'
primary_conninfo = 'host=3.142.184.72 port=5432 user=replication password=password'
trigger_file = '/var/lib/postgresql/9.6/trigger'
restore_command = 'cp /var/lib/postgresql/9.6/archive/%f "%p"'
```
Here, we are telling that when stand_by mode is on then save our connection info with the host address and replication as credentials.

**NOTE:** In primary conf_info, you can replace `host={primary_server_ip}`, `user={rplication_name}` and `password={replication_role_password}`.

Now, start the PostgreSQL server. The replica will now be running in hot standby mode.
```console
sudo systemctl start postgresql
```
#### Configure Replica1 Node

**NOTE:** All steps are same as the Replica setup **(Configure Replica Node)** for PostgreSQL installation.
```console
sudo systemctl stop postgresql
sudo rm -rv /var/lib/postgresql/9.6/main/*
pg_basebackup -h {{ host_server_ip }} -D /var/lib/postgresql/9.6/main/ -P -U replication 
sudo vim /etc/postgresql/9.6/main/postgresql.conf ## hot_standby=on
sudo vim /var/lib/postgresql/9.6/main/recovery.conf
sudo systemctl start postgresql
```

#### Test Replication Setup

In **primary node**, create a database with database name postgresql.

![image](https://user-images.githubusercontent.com/92078754/217457571-e2cfd18c-f27b-4ac8-9c96-dc38d81d5970.png)

In **replica node** the database **postgresql** is created in the primary node will replicate on the replica node. And produces below error while writing something here.

![image](https://user-images.githubusercontent.com/92078754/217457990-ceedf971-1334-483d-906f-2a005f7e13f3.png)

**Replica1:** Here, the data from primary node is also replicated. And produces below error while writing something here.

![image](https://user-images.githubusercontent.com/92078754/217460213-91bf664f-f498-4b8d-b817-b5476954273b.png)











