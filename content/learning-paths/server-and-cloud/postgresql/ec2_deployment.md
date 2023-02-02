---
# User change
title: "Deploy a single instance of PostgreSQL"

weight: 2 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy single instance of PostgreSQL 

## Prerequisites

* An [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [AWS IAM authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
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


**Note:** Use the public key id_rsa.pub inside the Terraform file to provision/start the instance and the private key id_rsa to connect to the instance. Add the below code in the main.tf file, we do not need to generate a public key every time we run terraform apply.

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
  access_key  = "AXXXXXXXXXXXXXXXXXXXX"
  secret_key  = "AXXXXXXXXXXXXXXXXXXXX"
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
  value = [aws_instance.PSQL_TEST.public_ip]
}
// Generate inventory file
resource "local_file" "inventory" {
    depends_on= [aws_instance.PSQL_TEST]
    filename = "/home/ubuntu/xxx/demo/hosts"
    content = <<EOF
          [db_master]
          ${aws_instance.PSQL_TEST.public_ip}         
          [all:vars]
          ansible_connection=ssh
          ansible_user=ubuntu
          EOF
}
```
**NOTE:-** Replace `public_key`, `access_key`, `secret_key`, and `key_name` with your values.

Now, use the below Terraform commands to deploy the `main.tf` file.


### Terraform Commands

#### Initialize Terraform



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



## Configure PostgreSQL through Ansible
Ansible is a software tool that provides simple but powerful automation for cross-platform computer support.
Ansible allows you to configure not just one computer, but potentially a whole network of computers at once.
To run Ansible, we have to create a `.yml` file, which is also known as `Ansible-Playbook`. The playbook contains a collection of tasks.

### Here is the complete YML file of Ansible-Playbook
```console
---
- hosts: 3.133.100.152
  become: yes
  become_method: sudo

  vars_files:
    - vars.yml

  pre_tasks:
    - name: Update the Machine
      shell: apt-get update -y
    - name : add lsb_release
      shell: sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
    - name: fetch & add key
      shell: wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
    - name: install postgres
      shell: sudo apt-get install postgresql -y
    - name: start postgres server
      shell: sudo systemctl start postgresql
    - name: check the status
      shell: sudo systemctl status postgresql
    - name: Update apt repo and cache on all Debian/Ubuntu boxes
      apt: update_cache=yes force_apt_get=yes cache_valid_time=3600
      become: true
    - name: Upgrade all apt packages
      apt: upgrade=yes force_apt_get=yes
      become: true
    - name: Common- Install PostgreSQL packages
      package:
        name:
         - wget
         - nano
    - name: Install Python pip
      apt: name={{ item }} update_cache=true state=present force_apt_get=yes
      with_items:
      - python-pip
      - python3-pip
      become: true
    - name: Install Python packages
      pip: name={{ item }}
      with_items:
      - psycopg2-binary
      become: true
    - name: Install required packaged
      apt:
        name: "{{ item }}"
        state: present
      with_items:
        - acl
        - python3-pip
  tasks:
    - name: "Find out if PostgreSQL is initialized"
      ansible.builtin.stat:
        path: "/var/lib/pgsql/main/pg_hba.conf"
      register: init_status

    - name: "Start and enable services"
      service: "name={{ item }} state=started enabled=yes"
      with_items:
        - postgresql
    - name: "Create app database"
      postgresql_db:
        state: present
        name: "{{ db_name }}"
      become: yes
      become_user: postgres

    - name: "Create db user"
      postgresql_user:
        state: present
        name: "{{ db_user }}"
        password: "{{ db_password }}"
      become: yes
      become_user: postgres

    - name: "Allow md5 connection for the db user"
      postgresql_pg_hba:
        dest: "~/14/main/pg_hba.conf"
        contype: host
        databases: all
        method: md5
        users: "{{ db_user }}"
        create: true
      become: yes
      become_user: postgres
      notify: restart postgres
    - name: Copy database dump file
      copy:
       src: /home/ubuntu/xxx/demo/dump.sql
       dest: /tmp
    - name: "Add some dummy data to our database"
      become: true
      become_user: postgres
      shell: psql {{ db_name }} < /tmp/dump.sql
  handlers:
    - name: restart postgres
      service: name=postgresql state=restarted

```
**NOTE:** Replace {{db_name}} with your database name, {{ db_user }} with your user, and {{ db_password }} with your password or you can add all these variables in the vars.yml file. In our case, the inventory file will generate automatically after the terraform apply command. We have used the `dump.sql` file to create a table and insert values into the database.

```console
CREATE TABLE IF NOT EXISTS test (
  message varchar(255) NOT NULL
);
INSERT INTO test(message) VALUES('Ansible is fun');
ALTER TABLE test OWNER TO "admin";
CREATE TABLE IF NOT EXISTS teachers (id INT PRIMARY KEY, first_name VARCHAR, last_name VARCHAR, subject VARCHAR, grade_level int);
INSERT INTO teachers VALUES (001, 'Rohan', 'Sharma', 'Hindi', 01), (002, 'Nitin', 'malik', 'stat', 02);
```


### Ansible Commands
To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i hosts
```
**NOTE:-** Replace `{your_yml_file}` with your values.

![image](https://user-images.githubusercontent.com/92078754/216302251-0b148a11-ed09-4527-ab68-e1ed3592873a.png)

Here is the output after the successful execution of the `ansible-playbook` command.

![image](https://user-images.githubusercontent.com/92078754/216256702-5fbab32c-2286-4ffa-9ff3-7d8fc9e54d9f.png)


## Connect to Database using EC2 instance

To connect to the database, we need the `public-ip` of the instance where PostgreSQL is deployed. 

```console
ssh ubuntu@3.133.100.152
```
**NOTE:-** Replace `{public_ip of an instance where Postgresql deployed}` which we have created through the `.yml` file.  
```console
sudo su postgres psql
```
![image](https://user-images.githubusercontent.com/92078754/215392554-9b99d2bb-8598-4af0-a2c8-47e44fddef95.png)

We can use the below commands to show our databases and tables.

```console
postgres# \l;
```
![image](https://user-images.githubusercontent.com/92078754/215725329-e1deae49-f608-4cba-8a5b-8a1acea87103.png)

Use the below command to use existing databases.
```console
postgres=# \c testdb;
```
![image](https://user-images.githubusercontent.com/92078754/215393022-56f6f41a-a115-41a5-b108-beccc8374fde.png)

Use the below commands to show the tables.

```console
testdb=# \dt;
```
![image](https://user-images.githubusercontent.com/92078754/215393252-9f25a09a-1c9a-4814-93c9-0643e9dcaf68.png)

Use the below command to access the content of the table.

```console
select * from teachers;
```

![image](https://user-images.githubusercontent.com/92078754/215393384-763ede3a-1ede-4318-ab1c-f9dce55a4165.png)




