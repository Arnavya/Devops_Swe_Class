==========================================================================

Lecture Title : Terraform in action

Lecture Date : 12th Jan 2026 

==========================================================================

# Q1. Provision a VPC with Public Subnet, Internet Gateway, and Route Table Using Terraform


File Structure:

```
WORKSPACE
├── main.tf
├── outputs.tf
├── vpc.tf
└── generate_plan_json.sh
```

### main.tf

```hcl
provider "aws" {
  region = "us-west-2"
}
```

### vpc.tf

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "public_subnet" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-west-2a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "public-rt"
  }
}

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}
```

### outputs.tf

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_id" {
  value = aws_subnet.public_subnet.id
}

output "igw_id" {
  value = aws_internet_gateway.igw.id
}
```

### Run Terraform Commands

```bash
terraform init
terraform plan -out=tfplan.binary
sudo bash generate_plan_json.sh         # Password: user@123!
terraform apply tfplan.binary
```


# Q2. Create an EBS-Backed EC2 Instance with Volume Attachment


File Structure:

```
WORKSPACE
├── provider.tf # Configures AWS provider and version
├── ec2.tf # EC2 instance definition
├── ebs.tf # EBS volume definition (8 GB, gp2)
├── attachment.tf # Volume attachment logic using aws_volume_attachment
└── generate_plan_json.sh
```

### provider.tf

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.44.0"
    }
  }

  required_version = ">= 1.3.0"
}

provider "aws" {
  region = "us-west-2"
}
```

### ec2.tf

```hcl
resource "aws_instance" "lab_instance" {
  ami           = "ami-05ee755be0cd7555c"
  instance_type = "t2.micro"

  tags = {
    Name = "lab-instance"
    lab  = "ec2-ebs-lab"
  }
}
```

### ebs.tf

```hcl
resource "aws_ebs_volume" "lab_volume" {
  availability_zone = aws_instance.lab_instance.availability_zone
  size              = 8
  type              = "gp2"

  tags = {
    Name = "lab-volume"
    lab  = "ec2-ebs-lab"
  }
}
```

### attachment.tf

```hcl
resource "aws_volume_attachment" "lab_attachment" {
  device_name  = "/dev/sdf"
  volume_id    = aws_ebs_volume.lab_volume.id
  instance_id  = aws_instance.lab_instance.id
  force_detach = true
}
```

### Run Terraform Commands

```bash
terraform init
terraform plan -out=tfplan.binary
sudo bash generate_plan_json.sh         # Password: user@123!
terraform apply tfplan.binary
```