---
layout: post
title: Creating a VPC in AWS using terraform
subtitle: A basic example
cover-img: /assets/img/abstract-geometrical-shapes.jpg
thumbnail-img: /assets/img/cloud-icons.jpg
gh-repo: dtlight/example-vpc
gh-badge: [star, fork, follow]
tags: [aws, terraform, vpc]
comments: false
author: David Light
---

{: .box-warning}
To follow this tutorial you will need the [AWS](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) CLIs installed. Personally I recommend [tfenv](https://github.com/tfutils/tfenv) to install and seamlessly switch between terraform versions. Lastly you will need [AWS credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html).<br/><br/>The terraform code can be accessed on [GitHub](https://github.com/dtlight/example-vpc){:target="_blank"}.

###  Why Terraform?
I've chosen Terraform (TF) an infrastructure as code (IaaC) tool, it has a large number of providers and is cloud-agnostic. Whichever IaaC tool you use, it's helpful to use one, here's why:


**Automation:** Enables efficient and consistent provisioning of infrastructure.

**Version Control:** Facilitates history tracking, rollback, and collaborative development.

**Scalability:** Supports easy replication and dynamic scaling of infrastructure.

**Documentation:** infra configuration is self-documented in code.

**Reusability:** the creation of reusable templates and modules for consistency.

## Architecture overview

The architecture of an Amazon Virtual Private Cloud (VPC) in AWS consists of several key components that collectively provide a virtual network environment for AWS resources. The VPC is an isolated virtual network in the AWS cloud that closely resembles a traditional network that would operate in a data center. It allows us to define our IP address range, create subnets, and configure routing tables. Here's a diagram of the components I'm going to use:



![aws-vpc](https://dtlight.github.io/assets/img/aws-vpc.png){: .mx-auto.d-block :}

Each component is explained as we progress.

## Initialising Terraform

First, initialise the TF directory where we keep our configuration.

```bash
mkdir tf-config
cd tf-config
```

Next to define our aws provider in TF by creating `main.tf` with the following content:

```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.8"
    }
  }

  required_version = ">= 1.6.5"
}
```
and then to initialise the directory with `terraform init` which should give an output message saying terraform has been successfully initialised.

![tf-init](https://dtlight.github.io/assets/img/tf-init.png){: .mx-auto.d-block :}

## Defining input variables
Input variables are parameters that allow you to customize the behavior of your Terraform configurations. These variables act as placeholders and can be used to make configurations more flexible and reusable.

Create a new file called vars.tf and enter the following (you can modify region and environment variables)

```terraform
variable "aws_region" {
  default = "eu-west-2"
}

#ev, staging, production etc
variable "environment" {
  default = "my-env"
}

variable "vpc_cidr" {
  default     = "10.0.0.0/16"
  description = "CIDR block of the vpc"
}

variable "public_subnets_cidr" {
  type        = list(any)
  default     = ["10.0.0.0/20", "10.0.128.0/20"]
  description = "CIDR block for Public Subnet"
}

variable "private_subnets_cidr" {
  type        = list(any)
  default     = ["10.0.16.0/20", "10.0.144.0/20"]
  description = "CIDR block for Private Subnet"
}
```

Now using the aws_region variable we just defined, we'll add an aws region to the provider we created earlier in main.tf

```terraform
provider "aws" {
  region = var.aws_region
}
```

## Locals

These are similar to variables (vars), both are used to enhance the flexibility and maintainability of configurations, but they serve slightly different purposes..

**Brief comparison:** variables are primarily for input and customization, while locals are for creating reusable expressions and improving code readability within a specific scope. 

**Detailed comparison:** 

Purpose: 

* Variables (vars): Used for taking input values from users or setting default values. They provide a way to parameterize your configurations and make them more dynamic.
* Locals: Used for creating reusable expressions or intermediate values within a module or resource block. They are helpful for simplifying complex expressions and improving readability.

Scope:

* Variables (vars): Have a wider scope and can be defined at different levels (e.g., root module, child module, or even at the command line when executing Terraform).
* Locals: Have a narrower scope and are limited to the module or resource block where they are defined. They are not accessible outside of this scope.

Input vs. Intermediate Values:

* Variables (vars): Are typically used for input values that can be set by users or by default values within the configuration.
* Locals: Are used for intermediate values or expressions calculated based on variables or other resources within the same module.

Mutability:

* Variables (vars): Can be changed or overridden at different levels, providing a way to customize behavior at runtime.
* Locals: Are immutable within a module; once defined, their values cannot be changed.

### Availability Zones
Each availability zone in a region is assigned a unique letter identifier, such as "a," "b," "c," and so on. So, in order to provision our VPC across two availability zones (eg eu-west-2a and eu-west-2b) we can add the following local value to our `main.tf` file:

```terraform
locals {
  availability_zones = ["${var.aws_region}a", "${var.aws_region}b"]
}
```

## Defining the VPC

We can begin defining a VPC with the following:

```terraform
resource "aws_vpc" "vpc" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}
```

and applying it with `terraform apply` which will display a plan and ask us if we want to accept or not,

![tf-plan](https://dtlight.github.io/assets/img/tf-plan.png){: .mx-auto.d-block :}

to which only `yes` is accepted and if successful the following success message should be displayed.

![tf-apply](https://dtlight.github.io/assets/img/tf-apply.png){: .mx-auto.d-block :}

We can also check in the AWS console:

![aws-console-vpc](https://dtlight.github.io/assets/img/aws-console-vpc.png){: .mx-auto.d-block :}

{: .box-warning}
When using the AWS console to view your resources, ensure you select the correct region

### Private and Public VPC Subnets

Within a VPC, we can create subnets, which are logical segments of the IP address range. Subnets are associated with a specific Availability Zone (AZ), and instances within a subnet can communicate with each other.

```terraform
# Public subnet
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.vpc.id
  count                   = length(var.public_subnets_cidr)
  cidr_block              = element(var.public_subnets_cidr, count.index)
  availability_zone       = element(local.availability_zones, count.index)
  map_public_ip_on_launch = true

  tags = {
    Name        = "${var.environment}-${element(local.availability_zones, count.index)}-public-subnet"
    Environment = "${var.environment}"
  }
}

# Private Subnet
resource "aws_subnet" "private_subnet" {
  vpc_id                  = aws_vpc.vpc.id
  count                   = length(var.private_subnets_cidr)
  cidr_block              = element(var.private_subnets_cidr, count.index)
  availability_zone       = element(local.availability_zones, count.index)
  map_public_ip_on_launch = false

  tags = {
    Name        = "${var.environment}-${element(local.availability_zones, count.index)}-private-subnet"
    Environment = "${var.environment}"
  }
}
```

### Internet Gateway and NAT Gateway

**An Internet Gateway (IGW)** allows resources within the VPC to communicate with the internet. It enables instances in the VPC to have public IP addresses and facilitates outbound and inbound internet traffic.


**Network Address Translation:** For private subnets that need internet access, a Network Address Translation (NAT) gateway or NAT instance is used to allow outbound internet traffic while keeping instances in the private subnet hidden from the internet as they are unidirectional for outbound traffic only.

{: .box-warning}
A well architected VPC would have a NAT gateway in more than one availability zone. 

For the purpose of this example a single NAT gateway with a single elastic IP is created.

```terraform
#Internet gateway
resource "aws_internet_gateway" "ig" {
  vpc_id = aws_vpc.vpc.id
  tags = {
    "Name"        = "${var.environment}-igw"
    "Environment" = var.environment
  }
}

# Elastic-IP (eip) for NAT
resource "aws_eip" "nat_eip" {
  vpc        = true
  depends_on = [aws_internet_gateway.ig]
}

# NAT Gateway
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = element(aws_subnet.public_subnet.*.id, 0)
  tags = {
    Name        = "nat-gateway-${var.environment}"
    Environment = "${var.environment}"
  }
}
```

Let's run a `terraform plan`` to see where we're at, I've copied the result to a [gist](https://gist.github.com/dtlight/c6bc42cc099dcee8f1c4a131797f952e){:target="_blank"}.

### Route Tables for private and public subnets

Create route table for the private subnet:

```terraform
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.vpc.id
  tags = {
    Name        = "${var.environment}-private-route-table"
    Environment = "${var.environment}"
  }
}
```

Create route table for the public subnet:

```terraform
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name        = "${var.environment}-public-route-table"
    Environment = "${var.environment}"
  }
}
```

Configure public route to use internet gateway for internet access:

```terraform
resource "aws_route" "public_internet_gateway" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.ig.id
}
```

Configure private route to use a NAT gateway for internet access:

```terraform
resource "aws_route" "private_internet_gateway" {
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_nat_gateway.nat.id
}
```

Now we have subnets, route tables, and route table access to the internet, we can associate these route tables with the correct subnets.

To associate the route table with the private subnets:

```terraform
resource "aws_route_table_association" "private" {
  count          = length(var.private_subnets_cidr)
  subnet_id      = element(aws_subnet.private_subnet.*.id, count.index)
  route_table_id = aws_route_table.private.id
}
```

And with the public subnets:

```terraform
resource "aws_route_table_association" "public" {
  count          = length(var.public_subnets_cidr)
  subnet_id      = element(aws_subnet.public_subnet.*.id, count.index)
  route_table_id = aws_route_table.public.id
}
```

The end result of the VPC looks like this in the AWS console:

![vpc-with-unused-route-table](https://dtlight.github.io/assets/img/vpc-rt-unused.png){: .mx-auto.d-block :}

Notice the unused route table (highlighted in orange) which by default is the main route table, whilst it may be tempting to delete this (in which case you need to choose another route table to be your main), you may find troubles when destroying your VPC through Terraform. In this case you can delete via the AWS console.


### Next Steps

{: .box-error}
Remember to delete your resources by applying a `terraform destroy` command

Some ideas to build on this example:
* NAT gateways in multiple AZs
* Network Access control list modification to deny certain traffic
* VPC Peering
* Running some Lambda functions, RDS, ELB or EC2 instances within the VPC
