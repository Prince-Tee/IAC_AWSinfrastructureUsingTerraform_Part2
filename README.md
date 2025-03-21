# Automate Infrastructure With IaC using Terraform-PART2

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
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/CREATE%20VPC.PNG)

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
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/content%20of%20vpc%20tf.PNG)

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
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/creating%20subnets.PNG)

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
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/content%20of%20variable%20tf.PNG)
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
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/terraform%20tvfars.PNG)
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

![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/out%20whole%20network%20has%20been%20set%20up.PNG)

![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Subnets%20Created%20in%20AWS%20Console.PNG)

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
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/content%20of%20internet%20gateway.PNG)
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
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Terraform%20Plan%20and%20apply%20Showing%20IGW%20Creation.PNG)

Internet Gateway Created in AWS Console
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Internet%20Gateway%20Created%20in%20AWS%20Console.PNG)

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
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/content%20of%20nat%20gateway.PNG)
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
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/NAT%20Gateway%20and%20EIP%20Configuration%20in%20Terraform.PNG)

Elastic IP Created in AWS Console
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Elastic%20IP%20Created%20in%20AWS%20Console.PNG)

NAT Gateway Created in AWS Console
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/NAT%20Gateway%20Created%20in%20AWS%20Console.PNG)

### AWS Routes

Routes define how traffic is directed within a VPC. We will create route tables for both public and private subnets.

#### Creating Route Tables
Let's create route tables for both public and private subnets. Create a new file route_tables.tf:

```hcl
# Create Private Route Table
resource "aws_route_table" "private-rtb" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }
  tags = merge(
    var.tags,
    {
      Name = format("%s-Private-Route-Table", var.tags["Environment"])
    }
  )
}

# Private Subnet Associations
resource "aws_route_table_association" "private-subnets-assoc" {
  count          = length(aws_subnet.private[*].id)
  subnet_id      = element(aws_subnet.private[*].id, count.index)
  route_table_id = aws_route_table.private-rtb.id
}

# Public Route Table
resource "aws_route_table" "public-rtb" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-Public-Route-Table", var.name)
    },
  )
}

# Public Route
resource "aws_route" "public-rtb-route" {
  route_table_id         = aws_route_table.public-rtb.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

# Public Subnet Associations
resource "aws_route_table_association" "public-subnets-assoc" {
  count          = length(aws_subnet.public[*].id)
  subnet_id      = element(aws_subnet.public[*].id, count.index)
  route_table_id = aws_route_table.public-rtb.id
}
```

![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/content%20of%20route_tables%20tf.PNG)

This section explains the Terraform code used to create **private** and **public route tables** and associate them with subnets.

---

### **1. Private Route Table**

- **`vpc_id`**: Associates the route table with the VPC.  
- **`route`**: Adds a route to the route table.  
  - **`cidr_block = "0.0.0.0/0"`**: Routes all traffic (0.0.0.0/0) to the NAT Gateway.  
  - **`nat_gateway_id`**: Specifies the NAT Gateway for the route.  
- **`tags`**: Adds tags to the route table.  
  - **`format()`**: Dynamically generates a name for the route table using the `Environment` tag.

---

### **2. Private Subnet Associations**

- **`count`**: Creates an association for each private subnet.  
- **`subnet_id`**: Associates each private subnet with the private route table.  
- **`route_table_id`**: Specifies the private route table to associate with.

---

### **3. Public Route Table**

- **`vpc_id`**: Associates the route table with the VPC.  
- **`tags`**: Adds tags to the route table.  
  - **`format()`**: Dynamically generates a name for the route table using `var.name`.

---

### **4. Public Route**

- **`route_table_id`**: Specifies the public route table.  
- **`destination_cidr_block = "0.0.0.0/0"`**: Routes all traffic (0.0.0.0/0) to the Internet Gateway.  
- **`gateway_id`**: Specifies the Internet Gateway for the route.

---

### **5. Public Subnet Associations**

- **`count`**: Creates an association for each public subnet.  
- **`subnet_id`**: Associates each public subnet with the public route table.  
- **`route_table_id`**: Specifies the public route table to associate with.

Route Tables Configuration in Terraform
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Route%20Tables%20Configuration%20in%20Terraform.PNG)

Route Tables Created in AWS Console
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Route%20Tables%20Created%20in%20AWS%20Console.PNG)

### Public Route Table Details
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Public%20Route%20Table%20Details.PNG)
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Public%20Subnet%20Associations.PNG)

### Private Route Table Details
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Private%20Route%20Table%20Details.PNG)
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Private%20Subnet%20Associations.PNG)

## AWS Identity and Access Management (IAM)

IAM allows you to manage access to AWS services and resources securely.

### IAM Roles

An IAM role is an identity that you can assume to gain temporary access to AWS resources.

#### Creating an IAM Role
Create a new file roles-and-policy.tf:

```hcl
resource "aws_iam_role" "ec2_instance_role" {
  name = "ec2_instance_role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
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
Explaining the Terraform code used to create an **IAM Role** in AWS.


1. **`resource "aws_iam_role" "ec2_instance_role" {`**  
   - Declares a Terraform resource of type `aws_iam_role` named `ec2_instance_role`. This will create an IAM Role in AWS.

2. **`name = "ec2_instance_role"`**  
   - Specifies the name of the IAM Role as `ec2_instance_role`.

3. **`assume_role_policy = jsonencode({ ... })`**  
   - Defines the trust policy for the IAM Role using JSON.  
   - **`jsonencode()`**: Converts the policy into JSON format.

4. **`Version = "2012-10-17"`**  
   - Specifies the version of the policy language.

5. **`Statement = [ ... ]`**  
   - Defines the permissions for the IAM Role.  
   - **`Action = "sts:AssumeRole"`**: Allows the `ec2.amazonaws.com` service to assume this role.  
   - **`Effect = "Allow"`**: Grants permission to assume the role.  
   - **`Principal = { Service = "ec2.amazonaws.com" }`**: Specifies that EC2 instances can assume this role.

6. **`tags = merge(var.tags, { Name = "aws assume role" })`**  
   - Adds tags to the IAM Role for better organization and management.  
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).  
   - **`Name = "aws assume role"`**: Assigns a name tag to the role.

![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/creating%20aws%20iam%20assume%20role.PNG)

### IAM Policies

IAM policies define permissions for actions that can be performed on AWS resources.

#### Creating an IAM Policy Inside the same file

```hcl
# Create a resource role
resource  "aws_iam_role" "ec2_instance_role"{
    name = "ec2_instance_role"
    assume_role_policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
            {
                Action      = "sts:AssumeRole"
                Effect      = "Allow"
                Sid         = ""
                Principal   = {
                    Service = "ec2.amazonaws.com"
                }
            }
        ]
    })

    tags = merge(
        var.tags,
        {
            Name = "aws assume role"
        }
    )
}

# Create an IAM Policy
resource "aws_iam_policy" "ec2_policy" {
    name                    = "ec2_instance_policy"
    description             = "This is a policy to grant access to all ec2 resource(s)" 
    policy                  = jsonencode({
        Version             = "2012-10-17"
        Statement           =[
            {
                Action      = [
                    "ec2:Describe*"
                ]
                Effect      = "Allow"
                Resource    = "*"
            }
        ]
    })

    tags = merge(
        var.tags,
        {
            Name    = "aws assume policy"
        }
    )
}

# Create KMS decrypt policy
resource "aws_iam_role_policy" "efs_kms_decrypt" {
    name = "AllowEFSToDecryptKMSKey"
    role = aws_iam_role.ec2_instance_role.name
    
    policy = jsonencode({
        Version          = "2012-10-17"
        Statement        = [
            {
                Action   = "kms:Decrypt"
                Effect   = "Allow"
                Resource = aws_kms_key.project-kms.arn
            }
        ]
    })
}
```
Explaining the Terraform code used to create **IAM Policies** in AWS.

### **1. Create an IAM Policy for EC2**

1. **`resource "aws_iam_policy" "ec2_policy" {`**  
   - Declares a Terraform resource of type `aws_iam_policy` named `ec2_policy`. This will create an IAM Policy in AWS.

2. **`name = "ec2_instance_policy"`**  
   - Specifies the name of the IAM Policy as `ec2_instance_policy`.

3. **`description = "This is a policy to grant access to all ec2 resource(s)"`**  
   - Provides a description of the policy.

4. **`policy = jsonencode({ ... })`**  
   - Defines the permissions for the IAM Policy using JSON.  
   - **`jsonencode()`**: Converts the policy into JSON format.

5. **`Version = "2012-10-17"`**  
   - Specifies the version of the policy language.

6. **`Statement = [ ... ]`**  
   - Defines the permissions for the IAM Policy.  
   - **`Action = ["ec2:Describe*"]`**: Allows the `Describe*` action on all EC2 resources.  
   - **`Effect = "Allow"`**: Grants permission for the action.  
   - **`Resource = "*"`**: Applies the permission to all EC2 resources.

7. **`tags = merge(var.tags, { Name = "aws assume policy" })`**  
   - Adds tags to the IAM Policy for better organization and management.  
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).  
   - **`Name = "aws assume policy"`**: Assigns a name tag to the policy.

### **2. Create a KMS Decrypt Policy**

1. **`resource "aws_iam_role_policy" "efs_kms_decrypt" {`**  
   - Declares a Terraform resource of type `aws_iam_role_policy` named `efs_kms_decrypt`. This will attach a policy to an IAM Role.

2. **`name = "AllowEFSToDecryptKMSKey"`**  
   - Specifies the name of the policy as `AllowEFSToDecryptKMSKey`.

3. **`role = aws_iam_role.ec2_instance_role.name`**  
   - Attaches the policy to the IAM Role (`ec2_instance_role`).

4. **`policy = jsonencode({ ... })`**  
   - Defines the permissions for the policy using JSON.  
   - **`jsonencode()`**: Converts the policy into JSON format.

5. **`Version = "2012-10-17"`**  
   - Specifies the version of the policy language.

6. **`Statement = [ ... ]`**  
   - Defines the permissions for the policy.  
   - **`Action = "kms:Decrypt"`**: Allows the `Decrypt` action on the specified KMS key.  
   - **`Effect = "Allow"`**: Grants permission for the action.  
   - **`Resource = aws_kms_key.project-kms.arn`**: Specifies the KMS key that can be decrypted.
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/creating%20iam%20policy%20inside%20the%20same%20file.PNG)

### Attaching Policy to Role in the same File
```hcl
resource "aws_iam_role_policy_attachment" "test-attach" {
  role       = aws_iam_role.ec2_instance_role.name
  policy_arn = aws_iam_policy.ec2_policy.arn
}
```
Explaining the Terraform code used to attach an **IAM Policy** to an **IAM Role**.

1. **`resource "aws_iam_role_policy_attachment" "test-attach" {`**  
   - Declares a Terraform resource of type `aws_iam_role_policy_attachment` named `test-attach`. This will attach an IAM Policy to an IAM Role.

2. **`role = aws_iam_role.ec2_instance_role.name`**  
   - Specifies the IAM Role to which the policy will be attached.  
   - References the role named `ec2_instance_role`.

3. **`policy_arn = aws_iam_policy.ec2_policy.arn`**  
   - Specifies the ARN (Amazon Resource Name) of the IAM Policy to attach.  
   - References the policy named `policy`.
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/attaching%20policy%20to%20the%20role%20we%20created.PNG)

### Instance Profiles

An instance profile is a container for an IAM role that you can use to pass role information to an EC2 instance.

#### Creating an Instance Profile

```hcl
resource "aws_iam_instance_profile" "ip" {
  name = "aws_instance_profile_test"
  role = aws_iam_role.ec2_instance_role.name
}
```
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/instance%20profile.PNG)

IAM Roles and Policies in the Code Phase after tf-plan and tf-apply
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/IAM%20Roles%20and%20Policies%20in%20the%20Code%20Phase%20after%20tf-plan%20and%20tf-apply.PNG)

If you encounter and Error "aws_kms_key.project-kms Not Declared"
Solution:
Declare the aws_kms_key resource in your variables.tf configuration. For example:

```hcl
Copy
resource "aws_kms_key" "project-kms" {
  description = "KMS key for project"
  tags = merge(
    var.tags,
    {
      Name = "project-kms-key"
    },
  )
}
```
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/IAM%20Roles%20and%20Policies%20in%20the%20Code%20Phase%20after%20tf-plan%20and%20tf-apply2.PNG)
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/IAM%20Roles%20and%20Policies%20in%20AWS%20Console.PNG)

### IAM Best Practices
Principle of Least Privilege

Grant minimum permissions needed
Regularly review and revoke unused permissions
Use specific resource ARNs when possible
Role Usage

Use roles instead of access keys
Rotate credentials regularly
Never hardcode credentials

Security Considerations
```hcl
# Example of restricted policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::my-bucket/*"],
      "Condition": {
        "StringEquals": {"aws:PrincipalTag/Environment": "Production"}
      }
    }
  ]
}
```
## Security Groups

Security groups act as virtual firewalls for your instances to control inbound and outbound traffic.

#### Creating Security Groups
Create a new file security.tf:

#### External Load Balancer Security Group
```hcl
# Security group for external ALB
resource "aws_security_group" "ext-alb-sg" {
  name        = "ext-alb-sg"
  vpc_id      = aws_vpc.main.id
  description = "Allow HTTP/HTTPS inbound traffic"

  ingress {
    description = "Allow HTTP connections"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow SSH connections"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all traffic"
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

Explaining the Terraform code used to create a **KMS Key** for encryption and a **Security Group** for an external Application Load Balancer (ALB).
#### **2. Create Security Group for External ALB**

1. **`resource "aws_security_group" "ext-alb-sg" {`**  
   - Declares a Terraform resource of type `aws_security_group` named `ext-alb-sg`. This will create a security group in AWS.

2. **`name = "ext-alb-sg"`**  
   - Specifies the name of the security group.

3. **`vpc_id = aws_vpc.main.id`**  
   - Associates the security group with the VPC.

4. **`description = "Allow HTTP/HTTPS inbound traffic"`**  
   - Provides a description for the security group.

5. **`ingress { ... }`**  
   - Defines inbound rules for the security group.  
   - **`description = "Allow HTTP connections"`**: Allows HTTP traffic on port 80.  
   - **`description = "Allow SSH connections"`**: Allows SSH traffic on port 22.  
   - **`cidr_blocks = ["0.0.0.0/0"]`**: Allows traffic from any IP address.

6. **`egress { ... }`**  
   - Defines outbound rules for the security group.  
   - **`description = "Allow all traffic"`**: Allows all outbound traffic.  
   - **`protocol = "-1"`**: Applies to all protocols.  
   - **`cidr_blocks = ["0.0.0.0/0"]`**: Allows traffic to any IP address.

7. **`tags = merge(var.tags, { Name = "ext-alb-sg" })`**  
   - Adds tags to the security group.  
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).  
   - **`Name = "ext-alb-sg"`**: Assigns a name tag to the security group.

![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/External%20Load%20Balancer%20Security%20Group.PNG)

#### Bastion Host Security Group
```hcl
resource "aws_security_group" "bastion_sg" {
  name        = "bastion_sg"
  vpc_id      = aws_vpc.main.id
  description = "Security group for bastion host SSH access"

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
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
      Name = "Bastion-SG"
    },
  )
}
```

Explaining the Terraform code used to create a **Security Group** for a Bastion Host.

1. **`resource "aws_security_group" "bastion_sg" {`**  
   - Declares a Terraform resource of type `aws_security_group` named `bastion_sg`. This will create a security group in AWS.

2. **`name = "bastion_sg"`**  
   - Specifies the name of the security group.

3. **`vpc_id = aws_vpc.main.id`**  
   - Associates the security group with the VPC.

4. **`description = "Security group for bastion host SSH access"`**  
   - Provides a description for the security group.

5. **`ingress { ... }`**  
   - Defines inbound rules for the security group.  
   - **`description = "SSH"`**: Allows SSH traffic on port 22.  
   - **`cidr_blocks = ["0.0.0.0/0"]`**: Allows SSH access from any IP address.

6. **`egress { ... }`**  
   - Defines outbound rules for the security group.  
   - **`from_port = 0`**: Allows all outbound traffic.  
   - **`to_port = 0`**: Allows all outbound traffic.  
   - **`protocol = "-1"`**: Applies to all protocols.  
   - **`cidr_blocks = ["0.0.0.0/0"]`**: Allows traffic to any IP address.

7. **`tags = merge(var.tags, { Name = "Bastion-SG" })`**  
   - Adds tags to the security group.  
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).  
   - **`Name = "Bastion-SG"`**: Assigns a name tag to the security group.
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Bastion%20Host%20Security%20Group.PNG)

#### Nginx Security Group
```hcl
resource "aws_security_group" "nginx-sg" {
  name   = "nginx-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    var.tags,
    {
      Name = "nginx-SG"
    },
  )
}

# Nginx Security Group Rules
resource "aws_security_group_rule" "inbound-nginx-https" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.ext-alb-sg.id
  security_group_id        = aws_security_group.nginx-sg.id
}

resource "aws_security_group_rule" "inbound-bastion-ssh" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion_sg.id
  security_group_id        = aws_security_group.nginx-sg.id
}
```
Explaining the Terraform code used to create a **Security Group** for Nginx and define its inbound rules.

#### **1. Create Nginx Security Group**

1. **`resource "aws_security_group" "nginx-sg" {`**  
   - Declares a Terraform resource of type `aws_security_group` named `nginx-sg`. This will create a security group in AWS.

2. **`name = "nginx-sg"`**  
   - Specifies the name of the security group.

3. **`vpc_id = aws_vpc.main.id`**  
   - Associates the security group with the VPC.

4. **`egress { ... }`**  
   - Defines outbound rules for the security group.  
   - **`from_port = 0`**: Allows all outbound traffic.  
   - **`to_port = 0`**: Allows all outbound traffic.  
   - **`protocol = "-1"`**: Applies to all protocols.  
   - **`cidr_blocks = ["0.0.0.0/0"]`**: Allows traffic to any IP address.

5. **`tags = merge(var.tags, { Name = "nginx-SG" })`**  
   - Adds tags to the security group.  
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).  
   - **`Name = "nginx-SG"`**: Assigns a name tag to the security group.

### **2. Nginx Security Group Rules**

1. **`resource "aws_security_group_rule" "inbound-nginx-https" {`**  
   - Declares a Terraform resource of type `aws_security_group_rule` named `inbound-nginx-https`. This will create an inbound rule for the Nginx security group.

2. **`type = "ingress"`**  
   - Specifies that this is an inbound rule.

3. **`from_port = 443`**  
   - Allows traffic on port 443 (HTTPS).

4. **`to_port = 443`**  
   - Allows traffic on port 443 (HTTPS).

5. **`protocol = "tcp"`**  
   - Specifies the TCP protocol.

6. **`source_security_group_id = aws_security_group.ext-alb-sg.id`**  
   - Allows traffic from the external ALB security group.

7. **`security_group_id = aws_security_group.nginx-sg.id`**  
   - Associates this rule with the Nginx security group.


#### **3. Bastion Host SSH Access Rule**

1. **`resource "aws_security_group_rule" "inbound-bastion-ssh" {`**  
   - Declares a Terraform resource of type `aws_security_group_rule` named `inbound-bastion-ssh`. This will create an inbound rule for SSH access.

2. **`type = "ingress"`**  
   - Specifies that this is an inbound rule.

3. **`from_port = 22`**  
   - Allows traffic on port 22 (SSH).

4. **`to_port = 22`**  
   - Allows traffic on port 22 (SSH).

5. **`protocol = "tcp"`**  
   - Specifies the TCP protocol.

6. **`source_security_group_id = aws_security_group.bastion_sg.id`**  
   - Allows traffic from the Bastion Host security group.

7. **`security_group_id = aws_security_group.nginx-sg.id`**  
   - Associates this rule with the Nginx security group.
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Nginx%20Security%20Group.PNG)

#### Internal ALB Security Group
```hcl
resource "aws_security_group" "int-alb-sg" {
  name   = "int-alb-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    var.tags,
    {
      Name = "int-alb-sg"
    },
  )
}

resource "aws_security_group_rule" "inbound-ialb-https" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.nginx-sg.id
  security_group_id        = aws_security_group.int-alb-sg.id
}
```
Explaining the Terraform code used to create a **Security Group** for an Internal ALB and define its inbound rules.

#### **1. Create Internal ALB Security Group**

1. **`resource "aws_security_group" "int-alb-sg" {`**  
   - Declares a Terraform resource of type `aws_security_group` named `int-alb-sg`. This will create a security group in AWS.

2. **`name = "int-alb-sg"`**  
   - Specifies the name of the security group.

3. **`vpc_id = aws_vpc.main.id`**  
   - Associates the security group with the VPC.

4. **`egress { ... }`**  
   - Defines outbound rules for the security group.  
   - **`from_port = 0`**: Allows all outbound traffic.  
   - **`to_port = 0`**: Allows all outbound traffic.  
   - **`protocol = "-1"`**: Applies to all protocols.  
   - **`cidr_blocks = ["0.0.0.0/0"]`**: Allows traffic to any IP address.

5. **`tags = merge(var.tags, { Name = "int-alb-sg" })`**  
   - Adds tags to the security group.  
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).  
   - **`Name = "int-alb-sg"`**: Assigns a name tag to the security group.

#### **2. Internal ALB Security Group Rule**

1. **`resource "aws_security_group_rule" "inbound-ialb-https" {`**  
   - Declares a Terraform resource of type `aws_security_group_rule` named `inbound-ialb-https`. This will create an inbound rule for the Internal ALB security group.

2. **`type = "ingress"`**  
   - Specifies that this is an inbound rule.

3. **`from_port = 443`**  
   - Allows traffic on port 443 (HTTPS).

4. **`to_port = 443`**  
   - Allows traffic on port 443 (HTTPS).

5. **`protocol = "tcp"`**  
   - Specifies the TCP protocol.

6. **`source_security_group_id = aws_security_group.nginx-sg.id`**  
   - Allows traffic from the Nginx security group.

7. **`security_group_id = aws_security_group.int-alb-sg.id`**  
   - Associates this rule with the Internal ALB security group.
![(screenshot) ](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Internal%20ALB%20Security%20Group.PNG)

#### Webserver Security Group
```hcl
resource "aws_security_group" "webserver-sg" {
  name   = "webserver-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    var.tags,
    {
      Name = "webserver-sg"
    },
  )
}

# Webserver Security Group Rules
resource "aws_security_group_rule" "inbound-web-https" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.int-alb-sg.id
  security_group_id        = aws_security_group.webserver-sg.id
}

resource "aws_security_group_rule" "inbound-web-ssh" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion_sg.id
  security_group_id        = aws_security_group.webserver-sg.id
}
```
This section explains the Terraform code used to create a **Security Group** for a Web Server and define its inbound rules.

#### **1. Create Web Server Security Group**

1. **`resource "aws_security_group" "webserver-sg" {`**  
   - Declares a Terraform resource of type `aws_security_group` named `webserver-sg`. This will create a security group in AWS.

2. **`name = "webserver-sg"`**  
   - Specifies the name of the security group.

3. **`vpc_id = aws_vpc.main.id`**  
   - Associates the security group with the VPC.

4. **`egress { ... }`**  
   - Defines outbound rules for the security group.  
   - **`from_port = 0`**: Allows all outbound traffic.  
   - **`to_port = 0`**: Allows all outbound traffic.  
   - **`protocol = "-1"`**: Applies to all protocols.  
   - **`cidr_blocks = ["0.0.0.0/0"]`**: Allows traffic to any IP address.

5. **`tags = merge(var.tags, { Name = "webserver-sg" })`**  
   - Adds tags to the security group.  
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).  
   - **`Name = "webserver-sg"`**: Assigns a name tag to the security group.

#### **2. Web Server Security Group Rules**

1. **`resource "aws_security_group_rule" "inbound-web-https" {`**  
   - Declares a Terraform resource of type `aws_security_group_rule` named `inbound-web-https`. This will create an inbound rule for HTTPS traffic.

2. **`type = "ingress"`**  
   - Specifies that this is an inbound rule.

3. **`from_port = 443`**  
   - Allows traffic on port 443 (HTTPS).

4. **`to_port = 443`**  
   - Allows traffic on port 443 (HTTPS).

5. **`protocol = "tcp"`**  
   - Specifies the TCP protocol.

6. **`source_security_group_id = aws_security_group.int-alb-sg.id`**  
   - Allows traffic from the Internal ALB security group.

7. **`security_group_id = aws_security_group.webserver-sg.id`**  
   - Associates this rule with the Web Server security group.

#### **3. Bastion Host SSH Access Rule**

1. **`resource "aws_security_group_rule" "inbound-web-ssh" {`**  
   - Declares a Terraform resource of type `aws_security_group_rule` named `inbound-web-ssh`. This will create an inbound rule for SSH access.

2. **`type = "ingress"`**  
   - Specifies that this is an inbound rule.

3. **`from_port = 22`**  
   - Allows traffic on port 22 (SSH).

4. **`to_port = 22`**  
   - Allows traffic on port 22 (SSH).

5. **`protocol = "tcp"`**  
   - Specifies the TCP protocol.

6. **`source_security_group_id = aws_security_group.bastion_sg.id`**  
   - Allows traffic from the Bastion Host security group.

7. **`security_group_id = aws_security_group.webserver-sg.id`**  
   - Associates this rule with the Web Server security group.
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Webserver%20Security%20Group.PNG)

#### Data Layer Security Group
```hcl
resource "aws_security_group" "datalayer-sg" {
  name   = "datalayer-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    var.tags,
    {
      Name = "datalayer-sg"
    },
  )
}

# Data Layer Security Group Rules
resource "aws_security_group_rule" "inbound-nfs-port" {
  type                     = "ingress"
  from_port                = 2049
  to_port                  = 2049
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.webserver-sg.id
  security_group_id        = aws_security_group.datalayer-sg.id
}

resource "aws_security_group_rule" "inbound-mysql-bastion" {
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion_sg.id
  security_group_id        = aws_security_group.datalayer-sg.id
}

resource "aws_security_group_rule" "inbound-mysql-webserver" {
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.webserver-sg.id
  security_group_id        = aws_security_group.datalayer-sg.id
}
```
This section explains the Terraform code used to create a **Security Group** for the Data Layer and define its inbound rules.

### **1. Create Data Layer Security Group**

1. **`resource "aws_security_group" "datalayer-sg" {`**  
   - Declares a Terraform resource of type `aws_security_group` named `datalayer-sg`. This will create a security group in AWS.

2. **`name = "datalayer-sg"`**  
   - Specifies the name of the security group.

3. **`vpc_id = aws_vpc.main.id`**  
   - Associates the security group with the VPC.

4. **`egress { ... }`**  
   - Defines outbound rules for the security group.  
   - **`from_port = 0`**: Allows all outbound traffic.  
   - **`to_port = 0`**: Allows all outbound traffic.  
   - **`protocol = "-1"`**: Applies to all protocols.  
   - **`cidr_blocks = ["0.0.0.0/0"]`**: Allows traffic to any IP address.

5. **`tags = merge(var.tags, { Name = "datalayer-sg" })`**  
   - Adds tags to the security group.  
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).  
   - **`Name = "datalayer-sg"`**: Assigns a name tag to the security group.

### **2. Data Layer Security Group Rules**

1. **`resource "aws_security_group_rule" "inbound-nfs-port" {`**  
   - Declares a Terraform resource of type `aws_security_group_rule` named `inbound-nfs-port`. This will create an inbound rule for NFS traffic.

2. **`type = "ingress"`**  
   - Specifies that this is an inbound rule.

3. **`from_port = 2049`**  
   - Allows traffic on port 2049 (NFS).

4. **`to_port = 2049`**  
   - Allows traffic on port 2049 (NFS).

5. **`protocol = "tcp"`**  
   - Specifies the TCP protocol.

6. **`source_security_group_id = aws_security_group.webserver-sg.id`**  
   - Allows traffic from the Web Server security group.

7. **`security_group_id = aws_security_group.datalayer-sg.id`**  
   - Associates this rule with the Data Layer security group.

### **3. Bastion Host MySQL Access Rule**

1. **`resource "aws_security_group_rule" "inbound-mysql-bastion" {`**  
   - Declares a Terraform resource of type `aws_security_group_rule` named `inbound-mysql-bastion`. This will create an inbound rule for MySQL traffic from the Bastion Host.

2. **`type = "ingress"`**  
   - Specifies that this is an inbound rule.

3. **`from_port = 3306`**  
   - Allows traffic on port 3306 (MySQL).

4. **`to_port = 3306`**  
   - Allows traffic on port 3306 (MySQL).

5. **`protocol = "tcp"`**  
   - Specifies the TCP protocol.

6. **`source_security_group_id = aws_security_group.bastion_sg.id`**  
   - Allows traffic from the Bastion Host security group.

7. **`security_group_id = aws_security_group.datalayer-sg.id`**  
   - Associates this rule with the Data Layer security group.

### **4. Web Server MySQL Access Rule**

1. **`resource "aws_security_group_rule" "inbound-mysql-webserver" {`**  
   - Declares a Terraform resource of type `aws_security_group_rule` named `inbound-mysql-webserver`. This will create an inbound rule for MySQL traffic from the Web Server.

2. **`type = "ingress"`**  
   - Specifies that this is an inbound rule.

3. **`from_port = 3306`**  
   - Allows traffic on port 3306 (MySQL).

4. **`to_port = 3306`**  
   - Allows traffic on port 3306 (MySQL).

5. **`protocol = "tcp"`**  
   - Specifies the TCP protocol.

6. **`source_security_group_id = aws_security_group.webserver-sg.id`**  
   - Allows traffic from the Web Server security group.

7. **`security_group_id = aws_security_group.datalayer-sg.id`**  
   - Associates this rule with the Data Layer security group.
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Data%20Layer%20Security%20Group.PNG)

#### Security Group Relationships

```mermaid
graph TD;
    A[External ALB SG] -->|HTTPS| B[Nginx SG]
    C[Bastion SG] -->|SSH| B
    B -->|HTTPS| D[Internal ALB SG]
    D -->|HTTPS| E[Webserver SG]
    C -->|SSH| E
    E -->|NFS| F[Data Layer SG]
    E -->|MySQL| F
    C -->|MySQL| F
```

![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Security%20Groups%20in%20the%20Code%20Phase%20after%20tf-plan%20and%20tf-apply.PNG)
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Security%20Groups%20in%20the%20Code%20Phase%20after%20tf-plan%20and%20tf-apply1.PNG)
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Security%20Groups%20in%20AWS%20Console.PNG)


## Target Groups and Load Balancer Configuration

### Target Groups Setup
Target Groups are used to route requests to registered targets (EC2 instances). We will now create three target groups for our infrastructure:

Nginx Target Group
WordPress Target Group
Tooling Target Group
Create a new file target-groups.tf:

```hcl
# Nginx Target Group
resource "aws_lb_target_group" "nginx-tgt" {
  name        = "nginx-tgt"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "instance"
  
  health_check {
    interval            = 10
    path               = "/healthz"
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
  }

  tags = merge(
    var.tags,
    {
      Name = "nginx-tgt"
    },
  )
}

# WordPress Target Group
resource "aws_lb_target_group" "wordpress-tgt" {
  name        = "wordpress-tgt"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "instance"
  
  health_check {
    interval            = 10
    path               = "/healthz"
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
  }

  tags = merge(
    var.tags,
    {
      Name = "wordpress-tgt"
    },
  )
}

# Tooling Target Group
resource "aws_lb_target_group" "tooling-tgt" {
  name        = "tooling-tgt"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "instance"
  
  health_check {
    interval            = 10
    path               = "/healthz"
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
  }

  tags = merge(
    var.tags,
    {
      Name = "tooling-tgt"
    },
  )
}
```
Explaining the Terraform code used to create **Target Groups** for Nginx, WordPress, and Tooling.

#### **1. Nginx Target Group**

1. **`resource "aws_lb_target_group" "nginx-tgt" {`**  
   - Declares a Terraform resource of type `aws_lb_target_group` named `nginx-tgt`. This will create a target group for Nginx.

2. **`name = "nginx-tgt"`**  
   - Specifies the name of the target group.

3. **`port = 80`**  
   - Listens on port 80 (HTTP).

4. **`protocol = "HTTP"`**  
   - Uses the HTTP protocol.

5. **`vpc_id = aws_vpc.main.id`**  
   - Associates the target group with the VPC.

6. **`target_type = "instance"`**  
   - Routes traffic to EC2 instances.

7. **`health_check { ... }`**  
   - Configures health checks for the target group.  
   - **`interval = 10`**: Checks every 10 seconds.  
   - **`path = "/healthz"`**: Uses `/healthz` for health checks.  
   - **`healthy_threshold = 2`**: Requires 2 successful checks to mark as healthy.  
   - **`unhealthy_threshold = 2`**: Requires 2 failed checks to mark as unhealthy.  
   - **`timeout = 5`**: Sets the health check timeout to 5 seconds.

8. **`tags = merge(var.tags, { Name = "nginx-tgt" })`**  
   - Adds tags to the target group.  
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).  
   - **`Name = "nginx-tgt"`**: Assigns a name tag to the target group.

#### **2. WordPress Target Group**

1. **`resource "aws_lb_target_group" "wordpress-tgt" {`**  
   - Declares a Terraform resource of type `aws_lb_target_group` named `wordpress-tgt`. This will create a target group for WordPress.

2. **`name = "wordpress-tgt"`**  
   - Specifies the name of the target group.

3. **`port = 80`**  
   - Listens on port 80 (HTTP).

4. **`protocol = "HTTP"`**  
   - Uses the HTTP protocol.

5. **`vpc_id = aws_vpc.main.id`**  
   - Associates the target group with the VPC.

6. **`target_type = "instance"`**  
   - Routes traffic to EC2 instances.

7. **`health_check { ... }`**  
   - Configures health checks for the target group.  
   - Uses the same health check settings as the Nginx target group.

8. **`tags = merge(var.tags, { Name = "wordpress-tgt" })`**  
   - Adds tags to the target group.  
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).  
   - **`Name = "wordpress-tgt"`**: Assigns a name tag to the target group.

#### **3. Tooling Target Group**

1. **`resource "aws_lb_target_group" "tooling-tgt" {`**  
   - Declares a Terraform resource of type `aws_lb_target_group` named `tooling-tgt`. This will create a target group for Tooling.

2. **`name = "tooling-tgt"`**  
   - Specifies the name of the target group.

3. **`port = 80`**  
   - Listens on port 80 (HTTP).

4. **`protocol = "HTTP"`**  
   - Uses the HTTP protocol.

5. **`vpc_id = aws_vpc.main.id`**  
   - Associates the target group with the VPC.

6. **`target_type = "instance"`**  
   - Routes traffic to EC2 instances.

7. **`health_check { ... }`**  
   - Configures health checks for the target group.  
   - Uses the same health check settings as the Nginx target group.

8. **`tags = merge(var.tags, { Name = "tooling-tgt" })`**  
   - Adds tags to the target group.  
   - **`merge()`**: Combines the default tags (`var.tags`) with a resource-specific tag (`Name`).  
   - **`Name = "tooling-tgt"`**: Assigns a name tag to the target group.
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/content%20of%20target-groups%20tf.PNG)
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/target%20group%20after%20terraform%20plan%20and%20terraform%20apply.PNG)
![(screenshot)](https://github.com/Prince-Tee/IAC_AWSinfrastructureUsingTerraform_Part2/blob/main/Screenshot%20from%20my%20local%20environmrnt/Target%20Groups%20Created%20in%20AWS%20Console.PNG)

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