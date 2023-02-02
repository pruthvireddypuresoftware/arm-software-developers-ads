---
# User change
title: "Deploy a 3-node PostgreSQL cluster with two hot standby servers"

weight: 2 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy 3 node of PostgreSQL 

## Prerequisites

* An [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [AWS IAM authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)
* [Terraform](https://github.com/zachlas/arm-software-developers-ads/blob/main/content/install-tools/terraform.md)

## Generate Access keys (access key ID and secret access key)

The installation of Terraform on your desktop or laptop needs to communicate with AWS. Thus, Terraform needs to be able to authenticate with AWS. For authentication, generate access keys (access key ID and secret access key). These access keys are used by Terraform for making programmatic calls to AWS via the AWS CLI.

### Go to My Security Credentials

![image](https://user-images.githubusercontent.com/87687468/190137370-87b8ca2a-0b38-4732-80fc-3ea70c72e431.png)

### On Your Security Credentials page click on create access keys (access key ID and secret access key)

![image](https://user-images.githubusercontent.com/87687468/190137925-c725359a-cdab-468f-8195-8cce9c1be0ae.png)

### Copy the Access Key ID and Secret Access Key 

![image](https://user-images.githubusercontent.com/87687468/190138349-7cc0007c-def1-48b7-ad1e-4ee5b97f4b90.png)

## Generate key-pair(public key, private key) using ssh keygen

### Generate the public key and private key

Before using Terraform, first generate the key-pair (public key, private key) using ssh-keygen. Then associate both public and private keys with AWS EC2 instances.

Generate the key-pair using the following command:

```console
ssh-keygen -t rsa -b 2048
```

By default, the above command will generate the public as well as private key at location **$HOME/.ssh**. You can override the end destination with a custom path.

Output when a key pair is generated:

![image](https://user-images.githubusercontent.com/92078754/215745442-7d9c0295-1cb0-48d1-bc72-4c30cbff276c.png)


**Note:** Use the public key id_rsa.pub inside the Terraform file to provision/start the instance and private key id_rsa to connect to the instance. Add the below code in main.tf file, we do not need to generate public key every time we run terrform apply.

```console
// ssh-key gen
resource "tls_private_key" task1_p_key  {
    algorithm = "RSA"
}
resource "aws_key_pair" "task1-key" {
    key_name    = "task1-key"
    public_key = tls_private_key.task1_p_key.public_key_openssh
  }
resource "local_file" "public_key" {
    depends_on = [
      tls_private_key.task1_p_key,
    ]
    filename = pathexpand("~/.ssh/id_rsa.pub")
    content  = tls_private_key.task1_p_key.public_key_openssh
    file_permission = "400"
}
resource "local_file" "private_key" {
    depends_on = [
      tls_private_key.task1_p_key,
    ]
    filename = pathexpand("~/.ssh/id_rsa")
    content  = tls_private_key.task1_p_key.private_key_openssh
    file_permission = "400"
}

```



## Deploy EC2 instance via Terraform

After generating the public and private keys, we have to create an EC2 instance. Then we will push our public key to the **authorized_keys** folder in `~/.ssh`. We will also create a security group that opens inbound ports `22`(ssh) and `5432`(PSQL). Below is a Terraform file called `main.tf` which will do this for us.



```console
// ssh-key gen
resource "tls_private_key" task1_p_key  {
    algorithm = "RSA"
}
resource "aws_key_pair" "task1-key" {
    key_name    = "task1-key"
    public_key = tls_private_key.task1_p_key.public_key_openssh
  }
resource "local_file" "public_key" {
    depends_on = [
      tls_private_key.task1_p_key,
    ]
    filename = pathexpand("~/.ssh/id_rsa.pub")
    content  = tls_private_key.task1_p_key.public_key_openssh
    file_permission = "400"
}
resource "local_file" "private_key" {
    depends_on = [
      tls_private_key.task1_p_key,
    ]
    filename = pathexpand("~/.ssh/id_rsa")
    content  = tls_private_key.task1_p_key.private_key_openssh
    file_permission = "400"
}

// instance creation
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXX"
  secret_key   = "AXXXXXXXXXXXXXXXX"
}

resource "aws_instance" "PSQL_TEST" {
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity10.name]
  key_name = "task1-key"
  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> private_ips.txt && echo ${self.public_ip} >> public_ips.txt && echo ${self.public_dns} >> public_ips.txt"
  }
  tags = {
    Name = "PSQL_TEST"
  }
}
resource "aws_instance" "replica-PSQL_TEST" {
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity10.name]
  key_name = "task1-key"
  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> private_ips.txt && echo ${self.public_ip} >> public_ips.txt && echo ${self.public_dns} >> public_ips.txt"
  }
  tags = {
    Name = "replica-PSQL_TEST"
  }                              
}
resource "aws_instance" "replica1-PSQL_TEST" {
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity10.name]
  key_name = "task1-key"
  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> private_ips.txt && echo ${self.public_ip} >> public_ips.txt && echo ${self.public_dns} >> public_ips.txt"
  }
  tags = {
    Name = "replica1-PSQL_TEST"
  }
}
resource "aws_default_vpc" "main" {
  tags = {
    Name = "main"
  }
}

resource "aws_security_group" "Terraformsecurity10" {
  name        = "Terraformsecurity10"
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

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  tags = {
    Name = "Terraformsecurity10"
  }
 }

output "Master_public_IP" {
  value = [aws_instance.PSQL_TEST.public_ip, aws_instance.replica-PSQL_TEST.public_ip, aws_instance.replica1-PSQL_TEST.public_ip]  
}


```
**NOTE:-** Replace `public_key`, `access_key`, `secret_key`, and `key_name` with your values.

Now, use the below Terraform commands to deploy the `main.tf` file.


### Terraform Commands

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

**NOTE:** The **terraform plan** command is optional. You can directly run **terraform apply** command. But it is always better to check the resources about to be created.

#### Apply a Terraform execution plan

Run `terraform apply` to apply the execution plan to your cloud infrastructure. The below command creates all required infrastructure.

```console
terraform apply
```      
![terraformaapl](https://user-images.githubusercontent.com/92078754/215390110-2514da83-ac67-4a28-99ed-020d68b6c71c.jpg)

## For configuration of master-slave setup manually, follow the below steps on all the nodes

Here are the three nodes deployed by Terraform.

**Primary node:** IP: 3.131.162.244

**Replica node:** IP: 52.15.37.65 (hot standby server that are read-only)

**Replica1 node:** IP: 3.17.146.121 (hot standby server that are read-only)

### Install PostgreSQL Server

The first step is to install PostgreSQL on the Primary and both the Replica nodes. Note that you need to install the same version of PostgreSQL on all 3 nodes for logical replication.

First, log in to your server via SSH and refresh the repositories and the follow below command for Postgres installation.

```console
sudo apt-get update
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-  key add -
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install postgresql-9.6  
```

### Configure Primary Node
 
 Next, log in to the primary node (3.131.162.244) as a Postgres user, the default user created with every new PostgreSQL installation.
 
```console
sudo -u postgres psql
```
Therefore, run the following command to create the replication user and assign replication privileges. In this command, replication is the replication user while the password is the user’s password.

![image](https://user-images.githubusercontent.com/92078754/215955679-50b6cb30-1f4e-4ca1-90d3-4758b1a69de7.png)

Then log out from the PostgreSQL prompt.

![image](https://user-images.githubusercontent.com/92078754/215955930-590628a4-463b-4090-b2d2-12defed9aeb0.png)


Next, you need to tweak the main configuration file **sudo vi /etc/postgresql/9.6/main/pg_hba.conf**.
With the file open, scroll down and locate the listen_addresses directive. The directive specifies the host under which the PostgreSQL database server listens for connections. Uncomment the directive by removing the # symbol and replace localhost with localhost ‘*’ in single quotation marks as shown:

![image](https://user-images.githubusercontent.com/92078754/215722631-7ec6ac62-7726-4fee-821c-ad1149699efd.png)

Next, locate the wal_level directive. The setting specifies the amount of information to be written to the Write Ahead Log (WAL) file.
Uncomment the line and set it to hot_standby as shown.

![image](https://user-images.githubusercontent.com/92078754/215723032-7e1486d7-8ac5-4eee-8be8-206c8a18eb24.png)

Next, locate the max_wal_sender and wal_keep_segments.

![image](https://user-images.githubusercontent.com/92078754/215723543-ece14cf8-f235-4a47-8966-0d6cbcb9e7da.png)

Next, locate the archive mode By default, it is set to off when set to on it will store the backup of replicas. Also, add archive_command when we are storing the data.

![image](https://user-images.githubusercontent.com/92078754/215724031-0e3813bd-7248-4372-981d-8601da395ae0.png)

That’s all for the changes needed in this configuration file. Save the changes and exit.

Next, create an archive directory.
```console
sudo mkdir /var/lib/postgresql/9.6/archive
```

Next, access the **/etc/postgresql/9.6/main/pg_hba.conf** configuration file.
Append this line at the end of the configuration file. This allows the replica and replica1({{ replica_ipv4.address }}, {{ replica1_ipv4.address }}) to connect with the master node using the replication.

![image](https://user-images.githubusercontent.com/92078754/216284502-2fad2d1b-f746-422e-9875-7626439f2720.png)

Save the changes and close the file. The restart PostgreSQL service.

```console
sudo systemctl restart postgresql
```
### Configure Replica Node

Before the replica node can start replicating data from the master node, you need to create a copy of the primary node’s data directory to the replica’s data directory. To achieve this, first, stop the PostgreSQL service on the replica node.

```console
sudo systemctl stop PostgreSQL 
```
Next, remove all files in the replica’s data directory in order to start on a clean slate and make room for the primary node data directory.

```console
sudo rm -rf /var/lib/postgresql/9.6/main/*
```

Now run the pg_basebackup utility as shown to copy data from the primary node to the replica node.

![image](https://user-images.githubusercontent.com/92078754/215405568-b7820bf2-92d1-4565-a305-d395455cf72f.png)

Now we shall modify **sudo vim /etc/postgresql/9.6/main/pg_hba.conf** changed here hot_standby=off to hot_standby=on.

![image](https://user-images.githubusercontent.com/92078754/215724525-3efb4088-2118-4ba9-9138-41b50f076a66.png)

Last we need to create a recovery.conf file on our data directory. Else replication will not happen.

```console
sudo vim /var/lib/postgresql/9.6/main/recovery.conf
```
![image](https://user-images.githubusercontent.com/92078754/215959419-edc5e377-e146-49a1-b79c-f1e1034aaf6a.png)

**NOTE:** In primary conf info you can replace host={with your public_ip}, user={with your rplication_name} and password={with you role_password}.

Here we are telling that stand_by mode is on then we will save our connection info with the host address and replication user and password.

Now, start the PostgreSQL server. The replica will now be running in hot standby mode.

```console
sudo systemctl start postgresql
```

### Configure Replica1 Node

**Note:** In here all steps are same as the Replica setup (Configure Replica Node) for PostgreSQL installation.

```console

sudo systemctl stop PostgreSQL
sudo rm -rv /var/lib/postgresql/9.6/main/*
pg_basebackup -h 3.131.162.244 -D /var/lib/postgresql/9.6/main/ -P -U replication 
sudo vim /etc/postgresql/9.6/main/pg_hba.conf ## hot_standby=on
sudo vim /var/lib/postgresql/9.6/main/recovery.conf
sudo systemctl start PostgreSQL

```

### Test Replication Setup

In the **primary node**, created a database with database name psql.

![image](https://user-images.githubusercontent.com/92078754/215960314-3c9da65d-7cfd-4006-abcd-694c00e35768.png)


In the **replica node** the database psql we created in the primary node will replicate on the replica node. And when we are trying to write something here then it is giving below error.

![image](https://user-images.githubusercontent.com/92078754/215960687-57d9efe3-79b2-41d0-81b7-6125f57be74f.png)


In **Replica1:** Here also the data from the primary node is replicated and when we are trying to write something here then it is giving below error.

![image](https://user-images.githubusercontent.com/92078754/215962383-b64bcfec-471b-4aeb-9917-a89b7c2eb475.png)









