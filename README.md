# Create EKS-Cluster & Nodes In Terraform
## 1. EC2 (Elastic Compute Cloud)

. Launch instance

. Instance type -- EX. (c7i-flex.large)

. Security groups -- EX. (launch-wizard-1)

. Storage -- EX. (20GB)

. Launch instance

---


## 2. Commands =

### 1. Switch to root user
```sh
sudo -i
```


### 2. Update the instance
```sh
apt update
```
### 3. Install AWS CLI on Ubuntu
```sh
snap install aws-cli --classic
```

### 4. Configure AWS CLI
```sh
aws configure
```

#### Access key ID
#### Secret access key

### 5. Terraform Instalation
```sh
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```
### 6. Write EKS Cluster file
```sh
nano main.tf
```
```sh
provider "aws" {
  region = "ap-south-1"
}

resource "aws_iam_role" "cbz_eks_role" {
  name = "cbz-eks-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
      Effect = "Allow"
      Principal = { Service = "eks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "cluster_AmazonEKSClusterPolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.cbz_eks_role.name
}

data "aws_vpc" "my_vpc" {
  default = true
}

data "aws_subnets" "subnet_ids" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.my_vpc.id]
  }
}

resource "aws_eks_cluster" "my_cluster" {
  name     = "my-cluster"
  role_arn = aws_iam_role.cbz_eks_role.arn

  vpc_config {
    subnet_ids = data.aws_subnets.subnet_ids.ids
  }

  depends_on = [
    aws_iam_role_policy_attachment.cluster_AmazonEKSClusterPolicy,
  ]
}
```
```sh
####################
# Node role for EKS managed node group
####################
resource "aws_iam_role" "eks_node_role" {
  name = "my-cluster-node-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "node_AmazonEKSWorkerNodePolicy" {
  role       = aws_iam_role.eks_node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "node_AmazonEKS_CNI_Policy" {
  role       = aws_iam_role.eks_node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_role_policy_attachment" "node_ECRReadOnly" {
  role       = aws_iam_role.eks_node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}

# Optional but useful: allow SSM access for debugging (Session Manager)
resource "aws_iam_role_policy_attachment" "node_SSM" {
  role       = aws_iam_role.eks_node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

####################
# EKS Managed Node Group
####################
variable "node_instance_type" {
  type    = string
  default = "t3.micro"
}

variable "node_min_size" {
  default = 1
}

variable "node_desired_size" {
  default = 1
}

variable "node_max_size" {
  default = 2
}

resource "aws_eks_node_group" "my_node_group" {
  cluster_name    = aws_eks_cluster.my_cluster.name
  node_group_name = "my-cluster-ng-1"
  node_role_arn   = aws_iam_role.eks_node_role.arn

  # Use the same subnets you already fetched for the cluster
  subnet_ids = data.aws_subnets.subnet_ids.ids

  scaling_config {
    desired_size = var.node_desired_size
    max_size     = var.node_max_size
    min_size     = var.node_min_size
  }

  instance_types = [var.node_instance_type]

  # Optional: tags for the nodes
  tags = {
    Name = "my-cluster-node"
    env  = "dev"
  }

  # Ensure node group waits for cluster to exist before creating
  depends_on = [aws_eks_cluster.my_cluster]
}
~~~
