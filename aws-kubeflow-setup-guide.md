# AWS Kubeflow Setup Guide

## 1. Create IAM User

1. Log in to AWS Console and navigate to IAM
2. Click "Users" > "Add user"
3. Set user details and select "Programmatic access"
4. Attach policies or create a custom policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:*",
                "eks:*",
                "iam:CreateServiceLinkedRole",
                "iam:AttachRolePolicy",
                "iam:PutRolePolicy",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:ListRoles",
                "iam:GetRole",
                "iam:PassRole",
                "elasticloadbalancing:*",
                "cloudformation:*"
            ],
            "Resource": "*"
        }
    ]
}
```

5. Review and create user
6. Save the Access Key ID and Secret Access Key

## 2. Configure AWS CLI

```bash
aws configure
# Enter your Access Key ID, Secret Access Key, region (e.g., us-west-2), and output format
```

## 3. Create EC2 Instance

1. Create a security group:

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)
SECURITY_GROUP_ID=$(aws ec2 create-security-group --group-name KubeflowSG --description "Security group for Kubeflow" --vpc-id $VPC_ID --output text)
aws ec2 authorize-security-group-ingress --group-id $SECURITY_GROUP_ID --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SECURITY_GROUP_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
```

2. Create a key pair:

```bash
KEY_NAME="kubeflow-key-$(date +%Y%m%d)"
aws ec2 create-key-pair --key-name $KEY_NAME --query 'KeyMaterial' --output text > ${KEY_NAME}.pem
chmod 400 ${KEY_NAME}.pem
```

3. Launch instance:

```bash
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[0].SubnetId" --output text)
AMI_ID=$(aws ec2 describe-images --owners 099720109477 --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*" "Name=state,Values=available" --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text)

INSTANCE_ID=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --count 1 \
    --instance-type t2.xlarge \
    --key-name $KEY_NAME \
    --security-group-ids $SECURITY_GROUP_ID \
    --subnet-id $SUBNET_ID \
    --output text \
    --query 'Instances[0].InstanceId')

aws ec2 wait instance-running --instance-ids $INSTANCE_ID
PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
```

## 4. SSH into Instance

```bash
ssh -i ${KEY_NAME}.pem ubuntu@$PUBLIC_IP
```

## 5. Install Kubeflow

Once connected to the instance:

```bash
# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install AWS CLI
sudo apt-get install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS CLI
aws configure

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Create EKS cluster
eksctl create cluster --name=kubeflow-cluster --nodes=2 --node-type=t3.xlarge --region=us-west-2

# Install kfctl
wget https://github.com/kubeflow/kfctl/releases/download/v1.2.0/kfctl_v1.2.0-0-gbc038f9_linux.tar.gz
tar -xvf kfctl_v1.2.0-0-gbc038f9_linux.tar.gz
sudo mv kfctl /usr/local/bin

# Set up environment variables
export PATH=$PATH:/usr/local/bin
export AWS_CLUSTER_NAME=kubeflow-cluster
export KFAPP=${AWS_CLUSTER_NAME}
export CONFIG_URI="https://raw.githubusercontent.com/kubeflow/manifests/v1.2-branch/kfdef/kfctl_aws.v1.2.0.yaml"

# Download and customize config file
wget -O kfctl_aws.yaml $CONFIG_URI

# Initialize and generate Kubeflow deployment
kfctl init ${KFAPP} --config=kfctl_aws.yaml -V
cd ${KFAPP}
kfctl generate all -V

# Apply the configuration
kfctl apply all -V

# Verify the installation
kubectl get pods -n kubeflow

# Get the Kubeflow dashboard URL
kubectl get ingress -n istio-system
```

## 6. Instance Management

Start instance:
```bash
aws ec2 start-instances --instance-ids $INSTANCE_ID
```

Stop instance:
```bash
aws ec2 stop-instances --instance-ids $INSTANCE_ID
```

Terminate instance:
```bash
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
```

Remember to clean up resources when done to avoid unnecessary charges:
```bash
eksctl delete cluster --name=kubeflow-cluster
```

