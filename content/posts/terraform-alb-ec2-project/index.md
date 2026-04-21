---
title: "Provisioning AWS Infrastructure with Terraform: EC2, ALB, and Security Groups"
date: 2025-09-12
draft: false
description: "A hands-on walkthrough of using Terraform to provision a complete AWS web stack — EC2 instance, Application Load Balancer, Target Groups, and Security Groups — all as code."
tags: ["AWS", "Terraform", "IaC", "EC2", "ALB", "DevOps"]
categories: ["AWS Projects"]
---

## Overview

In this project, I used **Terraform** to automate the provisioning of a complete web server stack on AWS. The goal was to define infrastructure as code (IaC) and deploy it with a single command — no clicking around the AWS Console.

The final architecture includes an **EC2 instance** running Apache behind an **Application Load Balancer (ALB)**, with properly scoped **Security Groups** controlling traffic flow.

### Architecture

```text
┌─────────────┐         ┌──────────────────────────────────────────┐
│   Laptop    │         │                 AWS                      │
│             │         │                                          │
│  Browser  ──┼────────►│  ALB (web-alb)                           │
│             │         │    └── Listener :80                      │
│  Terraform  │         │         └── Target Group (web-tg)        │
│    CLI    ──┼────────►│              └── EC2 (WebServer-ubuntu)   │
│             │         │                                          │
└─────────────┘         │  Security Groups:                        │
                        │    alb-sg  → allows HTTP from anywhere   │
                        │    web-sg  → allows HTTP from ALB only   │
                        │             + SSH from anywhere           │
                        └──────────────────────────────────────────┘
```

## Prerequisites

Before running Terraform, I installed the necessary tools on an Ubuntu EC2 instance that served as my workstation.

**AWS CLI** — configured with `aws sts get-caller-identity` to verify credentials.

**Terraform CLI** — installed from HashiCorp's official APT repository:

```bash
# Add HashiCorp GPG key and repository
wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt-get install terraform

# Verify
terraform --version
# Terraform v1.13.2
```

## Terraform Configuration

I structured the project into standard Terraform files. The full source is available on GitHub:
👉 [ckanth15/terraform-resources-generator](https://github.com/ckanth15/terraform-resources-generator)

### Project Structure

```text
terraform-project/
├── main.tf              # All resource definitions
├── variables.tf         # Input variable declarations
├── terraform.tfvars     # Variable values (gitignored — contains secrets)
├── terraform.tfstate    # State file (auto-generated)
└── .gitignore
```

### Provider Configuration

```hcl
provider "aws" {
  region     = var.aws_region
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
}
```

### Networking — Default VPC and Subnets

Rather than creating a custom VPC, I referenced the default VPC and its subnets using **data sources**:

```hcl
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}
```

### Security Groups

Two security groups control traffic — one for the ALB and one for the EC2 instance. The key design decision is that the EC2 instance only accepts HTTP traffic from the ALB's security group, not from the public internet directly.

**ALB Security Group** — allows inbound HTTP (port 80) from anywhere:

```hcl
resource "aws_security_group" "alb_sg" {
  name        = "alb-sg"
  description = "Allow HTTP traffic to ALB"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    description = "Allow HTTP from anywhere"
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
}
```

**EC2 Security Group** — allows HTTP only from the ALB security group, plus SSH:

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Allow HTTP from ALB and SSH from anywhere"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    description     = "Allow HTTP from ALB"
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id]
  }

  ingress {
    description = "Allow SSH from anywhere"
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
}
```

### Application Load Balancer

The ALB, target group, and listener are wired together to forward HTTP traffic to the EC2 instance:

```hcl
resource "aws_lb" "alb" {
  name               = "web-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = data.aws_subnets.default.ids
}

resource "aws_lb_target_group" "tg" {
  name     = "web-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = data.aws_vpc.default.id

  health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
    matcher             = "200"
  }
}

resource "aws_lb_listener" "http_listener" {
  load_balancer_arn = aws_lb.alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg.arn
  }
}
```

### EC2 Instance with User Data

A nice touch in this config is the **conditional user data** — the bootstrap script changes based on whether you're deploying Amazon Linux or Ubuntu:

```hcl
locals {
  user_data_ubuntu = <<-EOF
    #!/bin/bash
    apt-get update -y
    apt-get install -y apache2
    systemctl start apache2
    systemctl enable apache2
    echo "<h1>Hello from Ubuntu + Terraform</h1>" > /var/www/html/index.html
  EOF

  user_data_amazon = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Hello from Amazon Linux + Terraform</h1>" > /var/www/html/index.html
  EOF
}

resource "aws_instance" "web" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = data.aws_subnets.default.ids[0]
  vpc_security_group_ids = [aws_security_group.web_sg.id]
  key_name               = var.key_name
  user_data              = var.os_type == "ubuntu" ? local.user_data_ubuntu : local.user_data_amazon

  tags = {
    Name = "WebServer-${var.os_type}"
  }
}

resource "aws_lb_target_group_attachment" "tg_attach" {
  target_group_arn = aws_lb_target_group.tg.arn
  target_id        = aws_instance.web.id
  port             = 80
  depends_on       = [aws_instance.web]
}
```

### Variables

All configurable values are declared in `variables.tf` and assigned in `terraform.tfvars`:

```hcl
# variables.tf
variable "aws_region"    { default = "us-east-1" }
variable "instance_type" { default = "t2.micro" }
variable "os_type"       { default = "amazon" }
variable "ami_id"        { type = string }
variable "key_name"      { default = null }
variable "aws_access_key" {
  type    = string
  default = ""
}
variable "aws_secret_key" {
  type      = string
  default   = ""
  sensitive = true
}
```

```hcl
# terraform.tfvars (gitignored — never commit secrets!)
aws_region     = "us-east-1"
os_type        = "ubuntu"
instance_type  = "t2.micro"
aws_access_key = "YOUR_ACCESS_KEY"
aws_secret_key = "YOUR_SECRET_KEY"
ami_id         = "ami-090c309e8ced8ecc2"
```

> ⚠️ **Important:** Never commit `terraform.tfvars` with real credentials. Add it to `.gitignore` and consider using environment variables or AWS profiles instead.

### Outputs

```hcl
output "alb_dns_name"      { value = aws_lb.alb.dns_name }
output "ec2_public_ip"     { value = aws_instance.web.public_ip }
output "web_instance_id"   { value = aws_instance.web.id }
output "security_group_alb" { value = aws_security_group.alb_sg.id }
output "security_group_web" { value = aws_security_group.web_sg.id }
output "target_group_arn"  { value = aws_lb_target_group.tg.arn }
output "alb_listener_arn"  { value = aws_lb_listener.http_listener.arn }
```

## Execution

With the configuration ready, the deployment followed the standard Terraform workflow:

```bash
# Format and validate
terraform fmt
terraform validate
# Success! The configuration is valid.

# Initialize providers
terraform init
# Terraform has been successfully initialized!

# Preview changes
terraform plan
# Plan: 7 to add, 0 to change, 0 to destroy.

# Deploy
terraform apply
# Apply complete! Resources: 7 added, 0 changed, 0 destroyed.
#
# Outputs:
# alb_dns_name = "web-alb-410006054.us-east-1.elb.amazonaws.com"
# ec2_public_ip = "75.101.180.247"
```

Terraform created all 7 resources: two security groups, the ALB, target group, listener, EC2 instance, and target group attachment.

## Validation

After the ALB finished provisioning (~5 minutes), browsing to the ALB DNS name confirmed everything was working:

```
http://web-alb-410006054.us-east-1.elb.amazonaws.com
→ "Hello from Ubuntu + Terraform"
```

The EC2 instance appeared as **Running** in the AWS Console with health checks passing on the target group.

## Cleanup

Tearing everything down is a single command:

```bash
terraform destroy --auto-approve
# Destroy complete! Resources: 7 destroyed.
```

The AWS Console confirmed the EC2 instance moved to **Terminated** status, and all associated resources (ALB, security groups, target group) were removed.

## Key Takeaways

**Security-first design** — The EC2 instance's security group only permits HTTP from the ALB's security group, not from `0.0.0.0/0`. This ensures all web traffic flows through the load balancer.

**Reusability through variables** — The `os_type` variable lets this same config deploy either Ubuntu or Amazon Linux instances without changing the core infrastructure code.

**Sensitive data handling** — Using `sensitive = true` on the secret key variable, and keeping `terraform.tfvars` out of version control, prevents credential leaks.

**Clean lifecycle management** — `terraform destroy` removes everything cleanly, making this ideal for lab and learning environments where you don't want lingering resources accumulating charges.

## Source Code

The complete Terraform configuration is available at:
[github.com/ckanth15/terraform-resources-generator](https://github.com/ckanth15/terraform-resources-generator)

---

*This project was completed as part of the PG Program in Cloud Computing (PGPCC) from UT Austin via Great Learning.*
