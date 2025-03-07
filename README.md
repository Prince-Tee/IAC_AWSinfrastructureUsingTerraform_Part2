# Automate Infrastructure With IaC using Terraform- PART 2

Welcome to the world of Infrastructure as Code (IaC) with Terraform! In this guide, we will walk you through the process of automating the creation of AWS resources using Terraform.

## Table of Contents
1. [Introduction](#introduction)
2. [Networking](#networking)
   - [Private Subnets & Best Practices](#private-subnets--best-practices)
   - [Tagging](#tagging)
   - [Internet Gateways & `format()` Function](#internet-gateways--format-function)
   - [NAT Gateways](#nat-gateways)
   - [AWS Routes](#aws-routes)
3. [AWS Identity and Access Management (IAM)](#aws-identity-and-access-management-iam)
   - [IAM Roles](#iam-roles)
   - [IAM Policies](#iam-policies)
   - [Instance Profiles](#instance-profiles)
4. [Security Groups](#security-groups)
5. [Certificates with AWS Certificate Manager](#certificates-with-aws-certificate-manager)
6. [Application Load Balancers (ALB)](#application-load-balancers-alb)
   - [External ALB](#external-alb)
   - [Internal ALB](#internal-alb)
7. [Auto Scaling Groups (ASG)](#auto-scaling-groups-asg)
   - [Launch Templates](#launch-templates)
   - [Auto Scaling Groups](#auto-scaling-groups)
8. [Storage and Database](#storage-and-database)
   - [Elastic File System (EFS)](#elastic-file-system-efs)
   - [Relational Database Service (RDS)](#relational-database-service-rds)
9. [Variables and Outputs](#variables-and-outputs)
10. [Conclusion](#conclusion)

## Introduction

Terraform is an open-source tool that allows you to define and provision infrastructure using a high-level configuration language. With Terraform, you can manage cloud resources such as virtual machines, networks, and databases in a declarative way.

In this guide, we will continue from where we left off in the previous project and create more AWS resources, including private subnets, NAT gateways, IAM roles, security groups, and more.

## Networking

### Vpc creation
⚠️ NOTE: All the files we will be creating should be in the same directory/folder.
Create a new file provider.tf and populate it with the following:

```hcl
provider "aws" {
  region = var.region
}
```
(screenshot)

Create a new file vpc.tf, populate it with this content:
```hcl
# Get list of availability zones in the region
data "aws_availability_zones" "available" {
    state          = "available"
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  enable_dns_support = true
  enable_dns_hostnames = true

  tags = merge(
    var.tags,
    {
      Name = "Production-VPC"
    }
  )

}
```
(screenshot)

### Private Subnets & Best Practices

In AWS, subnets are subdivisions of a VPC (Virtual Private Cloud) that allow you to isolate resources. Private subnets are typically used for resources that should not be directly accessible from the internet.

#### Creating Private Subnets

Let's create 4 private subnets following these principles:

We are going to be utilizing;

Dynamic AZ Utilization
```hcl
# Use variables or length() function for AZ count
count = length(data.aws_availability_zones.available.names)
```

Automated CIDR Allocation
```hcl
# Use cidrsubnet() function for subnet ranges
cidr_block = cidrsubnet(var.vpc_cidr, 4, count.index)
```

To create private subnets, we will use the `aws_subnet` resource. We will also use variables and the `cidrsubnet()` function to allocate IP addresses dynamically. Create a new file subnets.tf. This a sample of my terraform code for creating 4 private subnet and 2 public subnets:

```hcl
# Dynamically Create Private Subnets
resource "aws_subnet" private {
    count = var.preferred_number_of_private_subnets == null? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets
    vpc_id = aws_vpc.main.id
    cidr_block = cidrsubnet(var.vpc_cidr, 4, count.index)
    availability_zone = data.aws_availability_zones.available.names[count.index % length(data.aws_availability_zones.available.names)]
    map_public_ip_on_launch = false
    
    tags = merge(
        var.tags,
        {
            Name = format("Private-Subnet-%s", count.index + 1)
            
        }
    )
}

# Dynamically Create Public Subnets
resource "aws_subnet" public {
    count = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
    vpc_id = aws_vpc.main.id
    cidr_block = cidrsubnet(var.vpc_cidr, 4, count.index + 4) # We added `+4` to the index to avoid overlapping with the private subnets CIDR blocks 
    map_public_ip_on_launch = true
    availability_zone = data.aws_availability_zones.available.names[count.index]

    tags = merge(
        var.tags,
        {
            Name = format("Public-Subnet-%s", count.index + 1)
            
        }
    )
}

```
(screenshot)

### **Line-by-Line Explanation for dynamically Creating Private Subnets**

1. **`resource "aws_subnet" "private" {`**  
   - Declares a Terraform resource of type `aws_subnet` named `private`. This will create private subnets in AWS.

2. **`count = var.preferred_number_of_private_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_private_subnets`**  
   - **`count`**: Determines how many private subnets to create.
   - **Ternary Operator (`? :`)**:
     - If `var.preferred_number_of_private_subnets` is `null`, the number of subnets created will equal the number of available Availability Zones (AZs) in the region (`length(data.aws_availability_zones.available.names)`).
     - If `var.preferred_number_of_private_subnets` is not `null`, the specified number of subnets will be created.

3. **`vpc_id = aws_vpc.main.id`**  
   - Associates the subnet with the VPC created earlier (`aws_vpc.main`).

4. **`cidr_block = cidrsubnet(var.vpc_cidr, 4, count.index)`**  
   - **`cidrsubnet()`**: Dynamically calculates the CIDR block for each subnet.
     - **`var.vpc_cidr`**: The CIDR block of the VPC.
     - **`4`**: The number of additional bits to add to the VPC CIDR for subnetting.
     - **`count.index`**: The index of the subnet being created (used to ensure unique CIDR blocks).

5. **`availability_zone = data.aws_availability_zones.available.names[count.index % length(data.aws_availability_zones.available.names)]`**  
   - Assigns an Availability Zone (AZ) to each subnet.
   - **`data.aws_availability_zones.available.names`**: A list of available AZs in the region.
   - **`count.index % length(data.aws_availability_zones.available.names)`**: Ensures AZs are reused in a round-robin fashion if the number of subnets exceeds the number of AZs.

6. **`map_public_ip_on_launch = false`**  
   - Ensures that instances launched in this subnet do **not** automatically receive a public IP address (since this is a private subnet).

7. **`tags = merge(var.tags, { Name = format("Private-Subnet-%s", count.index + 1) })`**  
   - Adds tags to the subnet.
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).
   - **`format("Private-Subnet-%s", count.index + 1)`**: Dynamically generates a unique name for each subnet (e.g., `Private-Subnet-1`, `Private-Subnet-2`).


### **Line-by-Line Explanation for dynamically Creating Public Subnets**

1. **`resource "aws_subnet" "public" {`**  
   - Declares a Terraform resource of type `aws_subnet` named `public`. This will create public subnets in AWS.

2. **`count = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets`**  
   - **`count`**: Determines how many public subnets to create.
   - **Ternary Operator (`? :`)**:
     - If `var.preferred_number_of_public_subnets` is `null`, the number of subnets created will equal the number of available AZs in the region (`length(data.aws_availability_zones.available.names)`).
     - If `var.preferred_number_of_public_subnets` is not `null`, the specified number of subnets will be created.

3. **`vpc_id = aws_vpc.main.id`**  
   - Associates the subnet with the VPC created earlier (`aws_vpc.main`).

4. **`cidr_block = cidrsubnet(var.vpc_cidr, 4, count.index + 4)`**  
   - **`cidrsubnet()`**: Dynamically calculates the CIDR block for each subnet.
     - **`var.vpc_cidr`**: The CIDR block of the VPC.
     - **`4`**: The number of additional bits to add to the VPC CIDR for subnetting.
     - **`count.index + 4`**: Adds an offset of `4` to the index to avoid overlapping CIDR blocks with private subnets.

5. **`map_public_ip_on_launch = true`**  
   - Ensures that instances launched in this subnet **automatically receive a public IP address** (since this is a public subnet).

6. **`availability_zone = data.aws_availability_zones.available.names[count.index]`**  
   - Assigns an Availability Zone (AZ) to each subnet.
   - **`data.aws_availability_zones.available.names`**: A list of available AZs in the region.
   - **`count.index`**: Ensures each subnet is created in a unique AZ.

7. **`tags = merge(var.tags, { Name = format("Public-Subnet-%s", count.index + 1) })`**  
   - Adds tags to the subnet.
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).
   - **`format("Public-Subnet-%s", count.index + 1)`**: Dynamically generates a unique name for each subnet (e.g., `Public-Subnet-1`, `Public-Subnet-2`).

Now let's make sure to create a variables.tf file and let it have the below content:

```hcl
variable "region"{
    default = "eu-west-2"
}

variable "vpc_cidr"{
    default = "172.16.0.0/16"
}

variable "enable_dns_support"{
    default = "true"
}

variable "enable_dns_hostnames"{
    default = "true"
}

variable "preferred_number_of_private_subnets"{
    default = null
}

variable "tags"{
    description = "Tags to be applied to the resources"
    type        = map(string)
    default = {
        Environment = "production"
        Owner       = "I.T. Admin"
        Terraform   = "true"
        Project     = "PBL"
    }
}
```
(screenshot)
### **Variables Explained**

1. **`region`**  
   - Specifies the AWS region where resources will be deployed. By default, it is set to `"eu-west-2"` (London).

2. **`vpc_cidr`**  
   - Defines the CIDR block for the VPC (Virtual Private Cloud). The default value is `"172.16.0.0/16"`, which provides a large IP address range for your VPC.

3. **`enable_dns_support`**  
   - Enables DNS support for the VPC. When set to `"true"`, instances in the VPC can use DNS for communication.

4. **`enable_dns_hostnames`**  
   - Enables DNS hostnames for the VPC. When set to `"true"`, instances in the VPC will automatically receive DNS hostnames.

5. **`preferred_number_of_private_subnets`**  
   - Specifies the number of private subnets to create. If set to `null`, the number of subnets will match the number of Availability Zones (AZs) in the region.

6. **`tags`**  
   - Defines a set of tags to be applied to all resources. Tags are key-value pairs that help organize and manage resources in AWS. The default tags include:  
     - `Environment = "production"`  
     - `Owner = "I.T. Admin"`  
     - `Terraform = "true"`  
     - `Project = "PBL"`

Then let us go ahead to create a terraform.tfvars file with contents like the below:
```hcl
region = "eu-west-2"
vpc_cidr = "172.16.0.0/16"
enable_dns_support = "true"
enable_dns_hostnames = "true"
preferred_number_of_private_subnets = 4
preferred_number_of_public_subnets = 2
tags = {
    Environment = "production"
    Owner       = "taiwo@adebiyi.com"
    Terraform   = "true"
    Project     = "PBL"
}
```
(screenshot)
### **Variable Values Explained**

1. **`region`**  
   - The AWS region where resources will be deployed is set to `"eu-west-2"` (London).

2. **`vpc_cidr`**  
   - The CIDR block for the VPC is set to `"172.16.0.0/16"`, providing a large IP address range for the VPC.

3. **`enable_dns_support`**  
   - DNS support is enabled for the VPC, allowing instances to use DNS for communication.

4. **`enable_dns_hostnames`**  
   - DNS hostnames are enabled for the VPC, so instances will automatically receive DNS hostnames.

5. **`preferred_number_of_private_subnets`**  
   - The number of private subnets to create is set to `4`.

6. **`preferred_number_of_public_subnets`**  
   - The number of public subnets to create is set to `2`.

7. **`tags`**  
   - Tags are applied to all resources for better organization and management. The tags include:  
     - `Environment = "production"`  
     - `Owner = "taiwo@adebiyi.com"`  
     - `Terraform = "true"`  
     - `Project = "PBL"`

Now let's run terraform apply and see what gets created from our aws console.

(screenshot)
(screenshot)

### Tagging

Tagging is a powerful concept in AWS that helps you organize and manage resources efficiently. Tags are key-value pairs that you can attach to resources.

#### Example of Tagging

```hcl
tags = merge(
  var.tags,
  {
    Name = "Name of the resource"
  },
)
```

- **merge()**: This function combines multiple maps of tags into a single map.
- **var.tags**: This is a variable that contains default tags.

### Internet Gateways & `format()` Function

An Internet Gateway (IGW) allows resources in a VPC to connect to the internet.

#### Creating an Internet Gateway
Create a new file internet_gateway.tf with below content:

```hcl
resource "aws_internet_gateway" "igw" {
    vpc_id = aws_vpc.main.id

    tags = merge(
        var.tags,
        {
            Name = format("%s-%s", aws_vpc.main.tags["Name"], "IGW")
        }
    )
}
```
(screenshot)
### **Explanation of the above code**

1. **`resource "aws_internet_gateway" "igw" {`**  
   - Declares a Terraform resource of type `aws_internet_gateway` named `igw`. This will create an Internet Gateway in AWS.

2. **`vpc_id = aws_vpc.main.id`**  
   - Attaches the IGW to the VPC created earlier (`aws_vpc.main`). This links the IGW to the VPC, enabling internet access for resources within the VPC.

3. **`tags = merge(var.tags, { Name = format("%s-%s", aws_vpc.main.tags["Name"], "IGW") })`**  
   - Adds tags to the IGW for better organization and management.  
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).  
   - **`format()`**: Dynamically generates a unique name for the IGW by combining the VPC's name (from its tags) with the string `"IGW"`.

Terraform Plan and apply Showing IGW Creation
(screenshot)

Internet Gateway Created in AWS Console
(screenshot)

### NAT Gateways

A NAT Gateway allows instances in a private subnet to connect to the internet while preventing the internet from initiating connections with those instances.

#### Creating Elastic IP and NAT Gateway
Create a new file natgateway.tf:

```hcl
# Create Elastic IP for NAT Gateway
resource "aws_eip" "nat_eip" {
  domain = "vpc"
  depends_on = [aws_internet_gateway.igw]

  tags = merge(
    var.tags,
    {
      Name = format("%s-EIP", var.name)
    },
  )
}

# Create NAT Gateway
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = element(aws_subnet.public.*.id, 0)
  depends_on    = [aws_internet_gateway.igw]

  tags = merge(
    var.tags,
    {
      Name = format("%s-Nat", var.name)
    },
  )
}
```
(screenshot)
### create an **Elastic IP (EIP)** and a **NAT Gateway** in AWS.

1. **Elastic IP (EIP)**  
   - **`domain = "vpc"`**: The EIP is created for use within the VPC.  
   - **`depends_on = [aws_internet_gateway.igw]`**: Ensures the Internet Gateway (IGW) is created before the EIP.  
   - **`tags`**: Tags are applied to the EIP for better organization and management. The `format()` function dynamically generates a unique name for the EIP using the `var.name` variable.

2. **NAT Gateway**  
   - **`allocation_id = aws_eip.nat_eip.id`**: Associates the NAT Gateway with the EIP created earlier.  
   - **`subnet_id = element(aws_subnet.public.*.id, 0)`**: Places the NAT Gateway in the first public subnet.  
   - **`depends_on = [aws_internet_gateway.igw]`**: Ensures the Internet Gateway (IGW) is created before the NAT Gateway.  
   - **`tags`**: Tags are applied to the NAT Gateway for better organization and management. The `format()` function dynamically generates a unique name for the NAT Gateway using the `var.name` variable.

NAT Gateway and EIP Configuration in Terraform
(screenshot)

Elastic IP Created in AWS Console
(screenshot)

NAT Gateway Created in AWS Console
(screenshot)

### AWS Routes

Routes define how traffic is directed within a VPC. We will create route tables for both public and private subnets.

#### Creating Route Tables

```hcl
resource "aws_route_table" "private-rtb" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-Private-Route-Table", var.name)
    },
  )
}

resource "aws_route_table_association" "private-subnets-assoc" {
  count          = length(aws_subnet.private[*].id)
  subnet_id      = element(aws_subnet.private[*].id, count.index)
  route_table_id = aws_route_table.private-rtb.id
}
```

- **aws_route_table**: This creates a route table for private subnets.
- **aws_route_table_association**: This associates the private subnets with the route table.

## AWS Identity and Access Management (IAM)

IAM allows you to manage access to AWS services and resources securely.

### IAM Roles

An IAM role is an identity that you can assume to gain temporary access to AWS resources.

#### Creating an IAM Role

```hcl
resource "aws_iam_role" "ec2_instance_role" {
  name = "ec2_instance_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })

  tags = merge(
    var.tags,
    {
      Name = "aws assume role"
    },
  )
}
```

- **assume_role_policy**: This policy allows EC2 instances to assume the role.

### IAM Policies

IAM policies define permissions for actions that can be performed on AWS resources.

#### Creating an IAM Policy

```hcl
resource "aws_iam_policy" "policy" {
  name        = "ec2_instance_policy"
  description = "A test policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "ec2:Describe*",
        ]
        Effect   = "Allow"
        Resource = "*"
      },
    ]
  })

  tags = merge(
    var.tags,
    {
      Name = "aws assume policy"
    },
  )
}
```

- **policy**: This policy allows the EC2 instance to describe EC2 resources.

### Instance Profiles

An instance profile is a container for an IAM role that you can use to pass role information to an EC2 instance.

#### Creating an Instance Profile

```hcl
resource "aws_iam_instance_profile" "ip" {
  name = "aws_instance_profile_test"
  role = aws_iam_role.ec2_instance_role.name
}
```

## Security Groups

Security groups act as virtual firewalls for your instances to control inbound and outbound traffic.

#### Creating Security Groups

```hcl
resource "aws_security_group" "ext-alb-sg" {
  name        = "ext-alb-sg"
  vpc_id      = aws_vpc.main.id
  description = "Allow TLS inbound traffic"

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    var.tags,
    {
      Name = "ext-alb-sg"
    },
  )
}
```

- **ingress**: This allows inbound HTTP traffic.
- **egress**: This allows all outbound traffic.

## Certificates with AWS Certificate Manager

AWS Certificate Manager (ACM) allows you to provision, manage, and deploy SSL/TLS certificates.

#### Creating a Certificate

```hcl
resource "aws_acm_certificate" "oyindamola" {
  domain_name       = "*.oyindamola.gq"
  validation_method = "DNS"
}
```

- **domain_name**: This is the domain name for which the certificate is issued.
- **validation_method**: This specifies the method used to validate the domain ownership.

## Application Load Balancers (ALB)

An Application Load Balancer (ALB) distributes incoming application traffic across multiple targets, such as EC2 instances.

### External ALB

An external ALB is internet-facing and routes traffic from the internet to your application.

#### Creating an External ALB

```hcl
resource "aws_lb" "ext-alb" {
  name     = "ext-alb"
  internal = false
  security_groups = [aws_security_group.ext-alb-sg.id]
  subnets  = [aws_subnet.public[0].id, aws_subnet.public[1].id]

  tags = merge(
    var.tags,
    {
      Name = "ACS-ext-alb"
    },
  )
}
```

- **internal**: This is set to `false` to make the ALB internet-facing.
- **security_groups**: This associates the ALB with a security group.

### Internal ALB

An internal ALB routes traffic within your VPC.

#### Creating an Internal ALB

```hcl
resource "aws_lb" "ialb" {
  name     = "ialb"
  internal = true
  security_groups = [aws_security_group.int-alb-sg.id]
  subnets  = [aws_subnet.private[0].id, aws_subnet.private[1].id]

  tags = merge(
    var.tags,
    {
      Name = "ACS-int-alb"
    },
  )
}
```

- **internal**: This is set to `true` to make the ALB internal.

## Auto Scaling Groups (ASG)

Auto Scaling Groups (ASG) allow you to automatically adjust the number of EC2 instances in response to changes in demand.

### Launch Templates

A launch template defines the configuration for the EC2 instances that will be launched by the ASG.

#### Creating a Launch Template

```hcl
resource "aws_launch_template" "bastion-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.bastion_sg.id]

  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
  }

  key_name = var.keypair

  tag_specifications {
    resource_type = "instance"

    tags = merge(
      var.tags,
      {
        Name = "bastion-launch-template"
      },
    )
  }

  user_data = filebase64("${path.module}/bastion.sh")
}
```

- **image_id**: This is the AMI ID for the EC2 instance.
- **user_data**: This is a script that runs when the instance is launched.

### Auto Scaling Groups

An Auto Scaling Group (ASG) automatically adjusts the number of EC2 instances based on demand.

#### Creating an ASG

```hcl
resource "aws_autoscaling_group" "bastion-asg" {
  name                      = "bastion-asg"
  max_size                  = 2
  min_size                  = 1
  health_check_grace_period = 300
  health_check_type         = "ELB"
  desired_capacity          = 1

  vpc_zone_identifier = [aws_subnet.public[0].id, aws_subnet.public[1].id]

  launch_template {
    id      = aws_launch_template.bastion-launch-template.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "bastion-launch-template"
    propagate_at_launch = true
  }
}
```

- **max_size**: This is the maximum number of instances in the ASG.
- **min_size**: This is the minimum number of instances in the ASG.
- **desired_capacity**: This is the desired number of instances in the ASG.

## Storage and Database

### Elastic File System (EFS)

Elastic File System (EFS) provides scalable file storage for use with AWS Cloud services and on-premises resources.

#### Creating an EFS

```hcl
resource "aws_efs_file_system" "ACS-efs" {
  encrypted  = true
  kms_key_id = aws_kms_key.ACS-kms.arn

  tags = merge(
    var.tags,
    {
      Name = "ACS-efs"
    },
  )
}
```

- **encrypted**: This enables encryption for the EFS.
- **kms_key_id**: This specifies the KMS key used for encryption.

### Relational Database Service (RDS)

Amazon RDS makes it easy to set up, operate, and scale a relational database in the cloud.

#### Creating an RDS Instance

```hcl
resource "aws_db_instance" "ACS-rds" {
  allocated_storage      = 20
  storage_type           = "gp2"
  engine                 = "mysql"
  engine_version         = "5.7"
  instance_class         = "db.t2.micro"
  name                   = "daviddb"
  username               = var.master-username
  password               = var.master-password
  parameter_group_name   = "default.mysql5.7"
  db_subnet_group_name   = aws_db_subnet_group.ACS-rds.name
  skip_final_snapshot    = true
  vpc_security_group_ids = [aws_security_group.datalayer-sg.id]
  multi_az               = "true"
}
```

- **allocated_storage**: This is the amount of storage allocated to the RDS instance.
- **engine**: This specifies the database engine (e.g., MySQL).
- **multi_az**: This enables Multi-AZ deployment for high availability.

## Variables and Outputs

### Variables

Variables allow you to customize your Terraform configuration without modifying the code.

#### Example of Variables

```hcl
variable "region" {
  type        = string
  description = "The region to deploy resources"
}

variable "vpc_cidr" {
  type        = string
  description = "The VPC CIDR"
}
```

### Outputs

Outputs allow you to extract information about your infrastructure after it has been deployed.

#### Example of Outputs

```hcl
output "alb_dns_name" {
  value = aws_lb.ext-alb.dns_name
}

output "alb_target_group_arn" {
  value = aws_lb_target_group.nginx-tgt.arn
}
```

## Conclusion

Congratulations! You have successfully created a comprehensive infrastructure using Terraform. This guide covered everything from networking and IAM to security groups, load balancers, and databases. Remember to always plan and apply your Terraform configurations carefully, and destroy resources when they are no longer needed to avoid unnecessary costs.

Happy Terraforming! 🚀