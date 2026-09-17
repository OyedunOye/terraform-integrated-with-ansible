# Terraform AWS EC2 Deployment Integrated with Ansible Playbook for Server Configuration

Terraform configuration that provisions a VPC and a single EC2 instance on AWS, and integrates automatic server configuration by ansible playbook run by local-exec provisioner in a null_resource block.

## Architecture

Defined in `main.tf`:

- **VPC & networking**
  - `aws_vpc.my-app-vpc` - a VPC with a configurable CIDR block
  - `aws_subnet.my-app-subnet-1` — a single subnet in a configurable availability zone
  - `aws_internet_gateway.my-app-igw` - internet gateway attached to the VPC
  - `aws_default_route_table.main-rtb` - the VPC's default route table, updated with a `0.0.0.0/0` route via the internet gateway
  - `aws_default_security_group.default-sg` - the VPC's default security group, allowing:
    - inbound SSH (port 22) from a configurable IP range
    - inbound TCP 8080 from anywhere (app traffic)
    - all outbound traffic

- **Compute**
  - `data.aws_ami.latest-amazon-linux-image` - looks up the latest Amazon Linux 2023 (x86_64, HVM) AMI
  - `aws_key_pair.ssh-key` - registers an existing local SSH public key for instance access
  - `aws_instance.my-app-server` - the EC2 instance, launched with a public IP address

- **Provider** (`providers.tf`) - pins the AWS provider to `~> 6.0`. AWS credentials/region are read from `~/.aws/credentials` (not configured in code).

- **Server Configuration by Ansible Playbook** - Uses local-exec provisioner to run an ansible playbook on the local machine to configure the terraform provisioned server.

## Outputs

| Output | Description |
|---|---|
| `aws_ami_id` | ID of the AMI used for the instance |
| `ec2-public_ip` | Public IP address of the EC2 instance |

## Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/downloads) compatible with the AWS provider `~> 6.0`
- An AWS account with credentials configured in `~/.aws/credentials` (or environment variables) and permissions to manage VPC, EC2, and related networking resources
- An existing local SSH key pair to use for instance access

## File structure

| File | Purpose |
|---|---|
| `main.tf` | VPC, networking, security group, AMI lookup, key pair, and EC2 instance resources |
| `providers.tf` | Terraform and AWS provider version requirements |
| `terraform.tfvars` | Variable values (gitignored - not committed, contains environment-specific config) |

## Variables

Declared in `main.tf` and supplied via `terraform.tfvars`:

| Name | Description |
|---|---|
| `vpc_cidr_block` | CIDR block for the VPC |
| `subnet_cidr_block` | CIDR block for the subnet |
| `avail_zone` | Availability zone for the subnet and instance |
| `env_prefix` | Prefix used to name/tag resources (e.g. `dev`) |
| `my_ip_range` | CIDR range allowed to SSH into the instance (your IP) |
| `instance_type` | EC2 instance type |
| `current_playbook` | Name of ansible playbook |
| `public_ssh_key_location` | Path to your local SSH public key file |

`terraform.tfvars` and `*.tfstate*` files are excluded from version control (see `.gitignore`) since they may contain sensitive or environment-specific data. Create your own `terraform.tfvars` before running Terraform, e.g.:

```hcl
vpc_cidr_block           = "10.0.0.0/16"
subnet_cidr_block        = "10.0.1.0/24"
avail_zone               = "us-east-1a"
env_prefix               = "dev"
my_ip_range              = "<your-ip>/32"
instance_type            = "t2.micro"
public_ssh_key_location  = "~/.ssh/id_rsa.pub"
```

## Usage

```bash
# Initialize Terraform and download providers
terraform init

# Review the planned changes
terraform plan

# Apply the configuration
terraform apply

# Tear down the infrastructure when 

terraform destroy

# ssh into the new ec2 instance

ssh ec2-user@<ec2-public_ip>
```
