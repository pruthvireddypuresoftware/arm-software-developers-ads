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

## Generate Access keys (Access key ID and Secret access key)

The installation of Terraform on your desktop or laptop needs to communicate with AWS. Thus, Terraform needs to be able to authenticate with AWS. For authentication, generate access keys (Access key ID and Secret access key). These access keys are used by Terraform for making programmatic calls to AWS via AWS CLI.

 Go to **Security Credentials**

![image](https://user-images.githubusercontent.com/92078754/217739255-cdbc372f-203c-45ee-b280-317eb4685447.png)

On Your **Security Credentials** page, click on **Create access keys** (Access key ID and Secret access key)

![image](https://user-images.githubusercontent.com/87687468/190137925-c725359a-cdab-468f-8195-8cce9c1be0ae.png)

Copy the **Access key ID** and **Secret access key** 

![image](https://user-images.githubusercontent.com/87687468/190138349-7cc0007c-def1-48b7-ad1e-4ee5b97f4b90.png)

## Generate key-pair(public key, private key) using ssh keygen

Before using Terraform, first generate the key-pair (public key, private key) using `ssh-keygen`. Then associate both public and private keys with AWS EC2 instances.

Generate the **key-pair** using the following command.

```console
ssh-keygen -t rsa -b 2048
```
       
By default, the above command will generate the public as well as private key at location **$HOME/.ssh**. You can override the end destination with a custom path.

Output when a key pair is generated.

![image](https://user-images.githubusercontent.com/92078754/217774946-7bd230c2-3a22-407e-a65b-71a83a14d30f.png)
      
**NOTE:** Use the public key **task2-key.pub** inside the Terraform file to provision/start the instance and private key task2-key to connect to the instance.

## Deploy EC2 instance via Terraform

After generating the public and private keys, we have to create an EC2 instance. Then we will push our public key to the **authorized_keys** folder in **~/.ssh**. We will also create a security group that opens inbound ports **22**(ssh) and **5432**(PSQL). Below is a Terraform file called **main.tf**.


```console

// instance creation
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXXX"
  secret_key  = "AXXXXXXXXXXXXXXXXXXXX"
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
output "Master_public_IP" {
  value = [aws_instance.PSQL_TEST.public_ip]
}
 resource "aws_key_pair" "deployer" {
         key_name   = "task2-key"
         public_key = "ssh-rsaxxxxxxxxxxxxxx"
  }
// Generate inventory file
resource "local_file" "inventory" {
    depends_on= [aws_instance.PSQL_TEST]
    filename = "(your_current_directory)/hosts"
    content = <<EOF
          [db_master]
          ${aws_instance.PSQL_TEST.public_ip}         
          [all:vars]
          ansible_connection=ssh
          ansible_user=ubuntu
          EOF
}
```
**NOTE:-** Replace `public_key`, `access_key`, `secret_key`, `key_name` and `filename` with respective values. You can check your current directory using `pwd` command.

Now, use the  Terraform commands below to deploy the **main.tf** file.
### Terraform Commands

#### Initialize Terraform

```console
terraform init
```
Run `terraform init` to initialize the Terraform deployment. This command is responsible for downloading all dependencies which are required for the AWS provider.

![image](https://user-images.githubusercontent.com/92078754/216525708-4742761b-1e7f-4a2d-a1da-3dbac9a11d81.png)

#### Create a Terraform execution plan

Run `terraform plan` to create an execution plan.

```console
terraform plan
```
![image](https://user-images.githubusercontent.com/92078754/216525501-ecfc05b8-dfc3-4aee-a4d5-ea8e6e3773f3.png)

**NOTE:** The `terraform plan` command is optional. You can directly run `terraform apply` command. But it is always better to check the created resources.

#### Apply a Terraform execution plan

Run `terraform apply` to apply the execution plan to your cloud infrastructure. The below command creates all required infrastructure.

```console
terraform apply
```      
![image](https://user-images.githubusercontent.com/92078754/217154441-c292c420-8a51-41da-a9dc-49714704cf1c.png)



## Configure PostgreSQL through Ansible
Ansible is a software tool that provides simple but powerful automation for cross-platform computer support.
Ansible allows you to configure not just one computer, but potentially a whole network of computers at once.
To run Ansible create a **.yml** file, which is also known as **Ansible-Playbook**. The playbook contains a collection of tasks.

Here is the complete YML file of Ansible-Playbook
```console
---
- hosts: {{ your_node_ip }}
  become: yes
  become_method: sudo

  vars_files:
    - vars.yml

  pre_tasks:
    - name: Update the Machine
      shell: apt-get update -y
    - name : LSB (Linux Standard Base) and Distribution information
      shell: sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
    - name: Fetch & add key
      shell: wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
    - name: Install postgres
      shell: sudo apt-get install postgresql -y
    - name: Start postgres server
      shell: sudo systemctl start postgresql
    - name: Check the status
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
         - acl 
    - name: Install Python pip
      apt: name={{ item }} update_cache=true state=present force_apt_get=yes
      with_items:
      - python3-pip
      become: true
    - name: Install Python packages
      pip: name={{ item }}
      with_items:
      - psycopg2-binary
      become: true
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
       src: /tmp/dump.sql
       dest: /tmp
    - name: "Add some dummy data to our database"
      become: true
      become_user: postgres
      shell: psql "{{ db_name }}" < /tmp/dump.sql
  handlers:
    - name: restart postgres
      service: name=postgresql state=restarted

```
**NOTE:** Replace `db_name` , `db_user` and `db_password` with your database name, user and password respectively or you can add all these variables in the [vars.yml](https://github.com/puppetlabs/pdk-docker/files/10739641/vars.txt) file. 

In our case, the hosts(inventory) file is generating automatically after the terraform apply command. 
We are using [dump.sql](https://github.com/puppetlabs/pdk-docker/files/10728905/dump.txt) file to create a table and insert values into the database. 

#### Ansible Commands

To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_hosts_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace `{{ your_yml_file }}`, `{your_hosts_file}` and `{path_to_private_key}` with your respective values.

![image](https://user-images.githubusercontent.com/92078754/218668084-4a8bc7c7-3fa5-46f5-825d-e50242177e56.png)

Here is the output after successful execution of the **ansible-playbook** command.

![image](https://user-images.githubusercontent.com/92078754/218667894-46e16245-8656-46e1-8392-7e43deaeb8db.png)

## Connect to Database 

To connect to the database, we need the `host` (public-ip of the node) where PostgreSQL is deployed. 

```console
ssh -i ~/.ssh/private_key username@host
```
**NOTE:-** Replace `{private_key}`,`{host}` and `username` with your respective values.

![image](https://user-images.githubusercontent.com/92078754/218686378-904788ec-6fb7-43b8-9c64-b0a245d6c4be.png)

Next, log into the postgres by using the below commands.
```console
cd ~postgres/
sudo su postgres -c psql
```
![image](https://user-images.githubusercontent.com/92078754/218687608-ad1f1a09-52e3-4bbe-91b0-0dcff9fa59c0.png)

Use the below command to show databases and tables.

```console
 \l;
```
![image](https://user-images.githubusercontent.com/92078754/218688208-abaa4da6-ef1a-45bd-a00c-e2e1502f80a3.png)

Use the below command to use existing databases.
```console
 \c testdb;
```
![image](https://user-images.githubusercontent.com/92078754/218688529-8bdbb62f-3ac8-49b1-a3d0-7647b2bff50a.png)

Use the below command to show the tables.

```console
 \dt;
```
![image](https://user-images.githubusercontent.com/92078754/218688649-679ace9f-1711-4181-a0b7-aa25e8a9ae8e.png)

Use the below command to access the content of the table.

```console
select * from teachers;
```
![image](https://user-images.githubusercontent.com/92078754/218688811-e9294095-ebe5-4c0f-b74f-770dcee777f5.png)
