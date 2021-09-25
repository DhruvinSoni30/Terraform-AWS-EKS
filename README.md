# Deploy an AWS EKS cluster using Terraform

### What is AWS EKS?

Amazon Elastic Kubernetes Service (Amazon EKS) is a managed Kubernetes service provided by AWS. Through AWS EKS we can run Kubernetes without installing and operating a Kubernetes control plane or worker nodes. AWS EKS helps you provide highly available and secure clusters and automates key tasks such as patching, node provisioning, and updates.

![1](https://github.com/DhruvinSoni30/Terraform-EKS-Ansible/blob/main/1.png)

### What is Terraform?

Terraform is a free and open-source infrastructure as code (IAC) that can help to automate the deployment, configuration, and management of the remote servers. Terraform can manage both existing service providers and custom in-house solutions.

![2](https://github.com/DhruvinSoni30/Terraform-EKS-Ansible/blob/main/2.png)

In this tutorial, I will be going to create an AWS EKS cluster with the help of Terraform scripts.

### Prerequisites:

* AWS Account
* Basic understanding of **AWS, Terraform & Kubernetes**

Now, let's start creating terraform scripts for the Kubernetes cluster.

**Step 1:- Create `.tf` file for storing environment variables**

* Create `vars.tf` file and add below content in it
  ```
  variable "access_key" {
    default = "<Your-AWS-Access-Key>"
  }
  variable "secret_key" {
    default = "<Your-AWS-Secret-Key>"
  }
  ```
 
**Step 2:- Create `.tf` file for AWS Configuration**

* Create `main.tf` file and add below content in it
  ```
  provider "aws" {
    region = "us-east-1"
    access_key = "${var.access_key}"
    secret_key = "${var.secret_key}"
  }
  data "aws_availability_zones" "azs" {
    state = "available"
  }
  ```
* data `"aws_availability_zones"` `"azs"` will provide the list of availability zone for the us-east-1 region

**Step 3:- Create .tf file for AWS VPC**

* Create `vpc.tf` file for VPC and add below content in it

  ```
  variable "region" {
    default = "us-east-1"
  }
  data "aws_availability_zones" "available" {}
  locals {
    cluster_name = "EKS-Cluster"
  }
  module vpc {
    source = "terraform-aws-modules/vpc/aws"
    version = "3.2.0"
    name = "Demo-VPC"
    cidr = "10.0.0.0/16"
    azs = data.aws_availability_zones.available.names
    private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
    public_subnets =  ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]
    enable_nat_gateway = true
    single_nat_gateway = true
    enable_dns_hostname = true
  tags = {
    "Name" = "Demo-VPC"
  }
  public_subnet_tags = {
    "Name" = "Demo-Public-Subnet"
  }
  private_subnet_tags = {
    "Name" = "Demo-Private-Subnet"
  }
  }
  ```
* We are using the AWS VPC module for VPC creation
* The above code will create the AWS VPC of `10.0.0.0/16` CIDR range in `us-east-1` region
* The VPC will have 3 public and private subnets
* `data "aws_availability_zones"` `"azs"` will provide the list of availability zone for the `us-east-1` region
* I have enabled the NAT Gateway & DNS Hostname

**Step 4:- Create .tf file for AWS Security Group**

* Create `security.tf` file for AWS Security Group and add below content in it

  ```
  resource "aws_security_group" "worker_group_mgmt_one" {
    name_prefix = "worker_group_mgmt_one"
    vpc_id = module.vpc.vpc_id
    ingress {
        from_port = 22
        to_port = 22
        protocol = "tcp"
    cidr_blocks = [
            "10.0.0.0/8"
        ]
    }
  }
  resource "aws_security_group" "worker_group_mgmt_two" {
    name_prefix = "worker_group_mgmt_two"
    vpc_id = module.vpc.vpc_id
 
    ingress {
        from_port = 22
        to_port = 22
        protocol = "tcp"
    cidr_blocks = [
            "10.0.0.0/8"
        ]
    }
  }
  resource "aws_security_group" "all_worker_mgmt" {
    name_prefix = "all_worker_management"
    vpc_id = module.vpc.vpc_id
  ingress {
        from_port = 22
        to_port = 22
        protocol = "tcp"
  cidr_blocks = [
            "10.0.0.0/8"
        ]
    }
  }
  ```
  
* We are creating 2 security groups for 2 worker node group
* We are allowing only 22 port for the SSH connection
* We are restricting the SSH access for `10.0.0.0/8` CIDR Block

**Step 5:- Create .tf file for the EKS Cluster**

* Create `eks.tf` file for VPC and add below content in it

  ```
  module "eks"{
    source = "terraform-aws-modules/eks/aws"
    version = "17.1.0"
    cluster_name = local.cluster_name
    cluster_version = "1.20"
    subnets = module.vpc.private_subnets
  tags = {
        Name = "Demo-EKS-Cluster"
    }
  vpc_id = module.vpc.vpc_id
    workers_group_defaults = {
        root_volume_type = "gp2"
    }
  workers_group = [
        {
            name = "Worker-Group-1"
            instance_type = "t2.micro"
            asg_desired_capacity = 2
            additional_security_group_ids = [aws_security_group.worker_group_mgmt_one.id]
        },
        {
            name = "Worker-Group-2"
            instance_type = "t2.micro"
            asg_desired_capacity = 1
            additional_security_group_ids = [aws_security_group.worker_group_mgmt_two.id]
        },
    ]
  }
  data "aws_eks_cluster" "cluster" {
    name = module.eks.cluster_id
  }
  data "aws_eks_cluster_auth" "cluster" {
    name = module.eks.cluster_id
  }
  ```
* For EKS Cluster creation we are using the terraform AWS EKS module
* The below code will create 2 worker groups with the desired capacity of 3 instances of type t2.micro
* We are attaching the recently created security group to both the worker node groups

  ```
  workers_group = [
        {
            name = "Worker-Group-1"
            instance_type = "t2.micro"
            asg_desired_capacity = 2
            additional_security_group_ids = [aws_security_group.worker_group_mgmt_one.id]
        },
        {
            name = "Worker-Group-2"
            instance_type = "t2.micro"
            asg_desired_capacity = 1
            additional_security_group_ids = [aws_security_group.worker_group_mgmt_two.id]
        },
    ]
  ```

**Step 6:- Create .tf file for terraform Kubernetes provider**

* Create `kubernetes.tf` file and add below content in it

  ```
  provider "kubernetes" {
    host = data.aws_eks_cluster.cluster.endpoint
    token = data.aws_eks_cluster_auth.cluster.token
    cluster_ca_certificate = base64encode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  }
  ```
  
* In the above code, we are using a recently created cluster as the host and authentication token as token
* We are using the cluster_ca_certificate for the CA certificate

**Step 7:- Create .tf file for outputs**

* Create `outputs.tf` file and add below content in it

  ```
  output "cluster_id" {
    value = module.eks.cluster_id
  }
  output "cluster_endpoint" {
    value = module.eks.cluster_endpoint
  }
  ```

* The above code will give output the name of our cluster and expose the endpoint of our cluster.

**Step 8:- Initialize the working directory**

* Run `terraform init` command in the working directory. It will download all the necessary providers and all the modules

**Step 9:- Create a terraform plan**

* Run `terraform plan` command in the working directory. It will give the execution plan

  ```
  Plan: 50 to add, 0 to change, 0 to destroy.
  Changes to Outputs:
  + cluster_endpoint = (known after apply)
  + cluster_id       = (known after apply)
  ```

**Step 10:- Create the cluster on AWS**

* Run `terraform apply` command in the working directory. It will be going to create the Kubernetes cluster on AWS
* Terraform will create the below resources on AWS

* VPC
* Route Table
* IAM Role
* NAT Gateway
* Security Group
* Public & Private Subnets
* EKS Cluster

**Step 11:- Verify the resources on AWS**

* Navigate to your AWS account and verify the resources

1. EKS Cluster:
![6](https://github.com/DhruvinSoni30/Terraform-EKS-Ansible/blob/main/6.png)
![7](https://github.com/DhruvinSoni30/Terraform-EKS-Ansible/blob/main/7.png)

2. VPC & other resources:
![8](https://github.com/DhruvinSoni30/Terraform-EKS-Ansible/blob/main/8.png)

3. Subnets:
![9](https://github.com/DhruvinSoni30/Terraform-EKS-Ansible/blob/main/9.png)

4. Security Group:
![10](https://github.com/DhruvinSoni30/Terraform-EKS-Ansible/blob/main/10.png)

5. IAM Role:
![11](https://github.com/DhruvinSoni30/Terraform-EKS-Ansible/blob/main/11.png)

6. Auto Scaling Groups:
![12](https://github.com/DhruvinSoni30/Terraform-EKS-Ansible/blob/main/12.png)

7. EC2 Instances:
![13](https://github.com/DhruvinSoni30/Terraform-EKS-Ansible/blob/main/13.png)

That's it now, you have learned how to create the AWS EKS cluster using Terraform. You can now play with it and modify it accordingly.
