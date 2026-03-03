# CloudFormation Deployment Guide

**Simple guide to deploy EC2 + RDS + S3 infrastructure using CloudFormation.**

This creates:
- **EC2 instance** (Ubuntu 22.04) with static IP
- **RDS MySQL** database (managed by AWS)
- **S3 bucket** for media uploads
- **VPC** with public and private subnets
- **Security Groups** (firewall rules)

---

## Prerequisites

### Step 1: Install AWS CLI

**On macOS:**
```bash
brew install awscli

# Verify installation
aws --version
```

Expected output: `aws-cli/2.x.x Python/3.x.x ...`

**On Linux:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

**On Windows:**
Download and run the MSI installer from: https://aws.amazon.com/cli/

---

### Step 2: Configure AWS Credentials

**Why:** AWS CLI needs your access keys to authenticate and create resources.

**Get your AWS credentials:**
1. Go to AWS Console → IAM → Users
2. Click your username (or create a new user)
3. Go to "Security credentials" tab
4. Click "Create access key"
5. Choose "Command Line Interface (CLI)"
6. Copy the **Access Key ID** and **Secret Access Key** (shown only once!)

**Configure AWS CLI:**
```bash
aws configure
```

Enter these when prompted:

```
AWS Access Key ID: [paste your Access Key ID]
AWS Secret Access Key: [paste your Secret Access Key]
Default region name: ap-southeast-2
Default output format: json
```

**Verify configuration:**
```bash
aws configure list
```

You should see:
```
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                <not set>             None    None
access_key     ****************ABCD shared-credentials-file    ~/.aws/credentials
secret_key     ****************WXYZ shared-credentials-file    ~/.aws/credentials
    region                ap-southeast-2      config-file    ~/.aws/config
```

**Test your credentials:**
```bash
aws sts get-caller-identity
```

You should see your AWS account ID and user ARN.

---

### Step 3: Create EC2 Key Pair (One-Time Setup)

**Why:** You need an SSH key to connect to your EC2 server.

```bash
aws ec2 create-key-pair \
  --key-name krahft-deploy-ssh-key \
  --region ap-southeast-2 \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/krahft-deploy-ssh-key.pem

# Set correct permissions (SSH requires this)
chmod 400 ~/.ssh/krahft-deplo-ssh-key.pem
```

**Verify the key pair was created:**
```bash
aws ec2 describe-key-pairs \
  --key-names krahft-deploy-ssh-key \
  --region ap-southeast-2 \
  --query 'KeyPairs[].KeyName' \
  --output table
```

You should see `krahft-deploy-ssh-key` listed.

> **Important:** Save `~/.ssh/nest-deploy-key.pem` safely — AWS stores only the public key. The private key can NEVER be downloaded again. Back it up!

---

## Step 1: Deploy Infrastructure

### Basic Command Template

```bash
aws cloudformation create-stack \
  --stack-name YOUR-STACK-NAME \
  --template-body file://cloudformation.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=KeyPairName,ParameterValue=nest-deploy-key \
    ParameterKey=ProjectName,ParameterValue=YOUR-PROJECT-NAME \
    ParameterKey=Environment,ParameterValue=YOUR-ENV-NAME \
    ParameterKey=DBMasterPassword,ParameterValue=YOUR-DB-PASSWORD \
  --region ap-southeast-2
```

### How to Edit the Command

**Replace these values:**

| Parameter | What to Replace | Example |
|-----------|----------------|---------|
| `YOUR-STACK-NAME` | Your stack name (unique in AWS) | `myapp-test-stack` |
| `YOUR-PROJECT-NAME` | Project prefix for resources | `myapp-test` |
| `YOUR-ENV-NAME` | Environment: `test` or `beta` | `test` |
| `YOUR-DB-PASSWORD` | RDS MySQL password (min 8 chars) | `MySecurePass123!` |

---

### Example: Deploy Test Environment

```bash
aws cloudformation create-stack \
  --stack-name Krahft-Test \
  --template-body file://cloudformation.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=KeyPairName,ParameterValue=krahft-deploy-ssh-key \
    ParameterKey=ProjectName,ParameterValue=krahft-test \
    ParameterKey=Environment,ParameterValue=test \
    ParameterKey=InstanceType,ParameterValue=t3.small \
    ParameterKey=DBInstanceClass,ParameterValue=db.t3.micro \
    ParameterKey=DBName,ParameterValue=mysqldb \
    ParameterKey=DBMasterPassword,ParameterValue=Krahft_12345! \
  --region ap-southeast-2
```

**Output:** You'll get a `StackId` — that confirms CloudFormation accepted the template.

---

### Example: Deploy Beta Environment

```bash
aws cloudformation create-stack \
  --stack-name Krahft-Beta \
  --template-body file://cloudformation.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=KeyPairName,ParameterValue=krahft-deploy-ssh-key \
    ParameterKey=ProjectName,ParameterValue=krahft-beta \
    ParameterKey=Environment,ParameterValue=beta \
    ParameterKey=InstanceType,ParameterValue=t3.small \
    ParameterKey=DBInstanceClass,ParameterValue=db.t3.micro \
    ParameterKey=DBName,ParameterValue=mysqldb \
    ParameterKey=DBMasterPassword,ParameterValue='Krahft_123456!' \
  --region ap-southeast-2
```

**Note:** You can deploy both Test and Beta at the same time — they are completely separate.

---

## Step 2: Wait for Stack Creation (10-15 minutes)

**Why:** CloudFormation creates resources one by one. RDS takes the longest (5-10 minutes).

### Check Test Environment Status

```bash
aws cloudformation describe-stacks \
  --stack-name Krahft-Test \
  --query 'Stacks[0].StackStatus' \
  --region ap-southeast-2
```

### Check Beta Environment Status

```bash
aws cloudformation describe-stacks \
  --stack-name Krahft-Beta \
  --query 'Stacks[0].StackStatus' \
  --region ap-southeast-2
```

**Keep checking every 30 seconds until you see:** `"CREATE_COMPLETE"`

You might see `"CREATE_IN_PROGRESS"` — that's normal, just wait.

---

## Step 3: Get Static IP and Other Outputs

### Get Test Environment Static IP

**Method 1: Get only the IP (simplest)**
```bash
aws cloudformation describe-stacks \
  --stack-name Krahft-Test \
  --query 'Stacks[0].Outputs[?OutputKey==`EC2PublicIP`].OutputValue' \
  --output text \
  --region ap-southeast-2
```

**Output:** `13.235.45.67` (your static IP)

**Method 2: Get all outputs in table format**
```bash
aws cloudformation describe-stacks \
  --stack-name Krahft-Test \
  --query 'Stacks[0].Outputs' \
  --region ap-southeast-2 \
  --output table
```

**Example output:**
```
--------------------------------------------------------------------------------------------------
|                                    DescribeStacks                                              |
+------------------+-----------------------------------------------------------------------------+
|  EC2PublicIP     |  13.235.45.67                    ← YOUR STATIC IP (use this)                |
|  RDSEndpoint     |  myapp-test-db.xxxxx.ap-southeast-2.rds.amazonaws.com                          |
|  RDSPort         |  3306                                                                       |
|  DBName          |  nestdb                                                                      |
|  DBMasterUsername|  admin                                                                       |
|  S3BucketName    |  myapp-test-media-830858426686                                               |
|  AppUrl          |  http://13.235.45.67                                                         |
|  ApiUrl          |  http://13.235.45.67/api                                                     |
+------------------+-----------------------------------------------------------------------------+
```

---

### Get Beta Environment Static IP

**Method 1: Get only the IP (simplest)**
```bash
aws cloudformation describe-stacks \
  --stack-name myapp-beta-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`EC2PublicIP`].OutputValue' \
  --output text \
  --region ap-southeast-2
```

**Output:** `13.235.45.68` (your Beta static IP - different from Test)

**Method 2: Get all outputs in table format**
```bash
aws cloudformation describe-stacks \
  --stack-name myapp-beta-stack \
  --query 'Stacks[0].Outputs' \
  --region ap-southeast-2 \
  --output table
```

---

### Quick Commands to Get Both IPs at Once

**Get Test IP:**
```bash
echo "Test IP: $(aws cloudformation describe-stacks --stack-name myapp-test-stack --query 'Stacks[0].Outputs[?OutputKey==`EC2PublicIP`].OutputValue' --output text --region ap-southeast-2)"
```

**Get Beta IP:**
```bash
echo "Beta IP: $(aws cloudformation describe-stacks --stack-name myapp-beta-stack --query 'Stacks[0].Outputs[?OutputKey==`EC2PublicIP`].OutputValue' --output text --region ap-southeast-2)"
```

**Get Both IPs:**
```bash
echo "Test IP: $(aws cloudformation describe-stacks --stack-name myapp-test-stack --query 'Stacks[0].Outputs[?OutputKey==`EC2PublicIP`].OutputValue' --output text --region ap-southeast-2)"
echo "Beta IP: $(aws cloudformation describe-stacks --stack-name myapp-beta-stack --query 'Stacks[0].Outputs[?OutputKey==`EC2PublicIP`].OutputValue' --output text --region ap-southeast-2)"
```

---

### Get Specific Outputs (JSON Format)

**Test Environment:**
```bash
aws cloudformation describe-stacks \
  --stack-name myapp-test-stack \
  --query 'Stacks[0].Outputs' \
  --region ap-southeast-2 \
  --output json
```

**Beta Environment:**
```bash
aws cloudformation describe-stacks \
  --stack-name myapp-beta-stack \
  --query 'Stacks[0].Outputs' \
  --region ap-southeast-2 \
  --output json
```

---

## Quick Reference: What Each Output Means

| Output | Description | Use For |
|--------|-------------|---------|
| `EC2PublicIP` | **Static IP address** (never changes) | SSH, deploy app, give to client |
| `RDSEndpoint` | Database hostname | Set as `DB_HOST` in your `.env` file |
| `RDSPort` | Database port (usually 3306) | Set as `DB_PORT` in your `.env` file |
| `DBName` | Database name | Set as `DB_NAME` in your `.env` file |
| `DBMasterUsername` | Database username | Set as `DB_USERNAME` in your `.env` file |
| `S3BucketName` | S3 bucket name | Set as `S3_BUCKET_NAME` in your `.env` file |

---

## Common Commands

### Check Stack Status

**Test:**
```bash
aws cloudformation describe-stacks \
  --stack-name myapp-test-stack \
  --query 'Stacks[0].StackStatus' \
  --region ap-southeast-2
```

**Beta:**
```bash
aws cloudformation describe-stacks \
  --stack-name myapp-beta-stack \
  --query 'Stacks[0].StackStatus' \
  --region ap-southeast-2
```

### List All Stacks

```bash
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE \
  --region ap-southeast-2 \
  --query 'StackSummaries[].StackName' \
  --output table
```

### Delete Stack (⚠️ Destroys Everything)

**Test:**
```bash
aws cloudformation delete-stack \
  --stack-name myapp-test-stack \
  --region ap-southeast-2
```

**Beta:**
```bash
aws cloudformation delete-stack \
  --stack-name myapp-beta-stack \
  --region ap-southeast-2
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `aws: command not found` | Install AWS CLI (see Step 1 above) |
| `Unable to locate credentials` | Run `aws configure` and enter your credentials |
| `Stack already exists` | Use a different `--stack-name` or delete the existing stack first |
| `CAPABILITY_NAMED_IAM required` | Make sure you include `--capabilities CAPABILITY_NAMED_IAM` |
| `Key pair not found` | Create the key pair first (see Prerequisites Step 3) |
| `Password too weak` | RDS password must be at least 8 characters |
| Stack stuck in `CREATE_IN_PROGRESS` | Wait 15-20 minutes. RDS takes the longest. Check AWS Console for errors. |
| `Access Denied` | Check your IAM user has permissions to create EC2, RDS, S3, CloudFormation |

---

## Summary: Complete Workflow

### First Time Setup

```bash
# 1. Install AWS CLI
brew install awscli  # macOS
# or download from aws.amazon.com/cli for Windows/Linux

# 2. Configure credentials
aws configure
# Enter: Access Key ID, Secret Key, region: ap-southeast-2, format: json

# 3. Create key pair
aws ec2 create-key-pair --key-name nest-deploy-key --region ap-southeast-2 \
  --query 'KeyMaterial' --output text > ~/.ssh/nest-deploy-key.pem
chmod 400 ~/.ssh/nest-deploy-key.pem

# 4. Deploy Test environment
aws cloudformation create-stack \
  --stack-name myapp-test-stack \
  --template-body file://cloudformation.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=KeyPairName,ParameterValue=nest-deploy-key \
    ParameterKey=ProjectName,ParameterValue=myapp-test \
    ParameterKey=Environment,ParameterValue=test \
    ParameterKey=DBMasterPassword,ParameterValue=MySecurePass123! \
  --region ap-southeast-2

# 5. Wait for completion (check status every 30 sec)
aws cloudformation describe-stacks \
  --stack-name myapp-test-stack \
  --query 'Stacks[0].StackStatus' \
  --region ap-southeast-2

# 6. Get Test static IP
aws cloudformation describe-stacks \
  --stack-name myapp-test-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`EC2PublicIP`].OutputValue' \
  --output text \
  --region ap-southeast-2
```

### Deploy Beta Environment

```bash
# 1. Deploy Beta (same command, different stack name and params)
aws cloudformation create-stack \
  --stack-name myapp-beta-stack \
  --template-body file://cloudformation.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=KeyPairName,ParameterValue=nest-deploy-key \
    ParameterKey=ProjectName,ParameterValue=myapp-beta \
    ParameterKey=Environment,ParameterValue=beta \
    ParameterKey=DBMasterPassword,ParameterValue=MySecurePass123! \
  --region ap-southeast-2

# 2. Wait for completion
aws cloudformation describe-stacks \
  --stack-name myapp-beta-stack \
  --query 'Stacks[0].StackStatus' \
  --region ap-southeast-2

# 3. Get Beta static IP
aws cloudformation describe-stacks \
  --stack-name myapp-beta-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`EC2PublicIP`].OutputValue' \
  --output text \
  --region ap-southeast-2
```

---

## Next Steps

After getting your static IPs:

1. **SSH to EC2:**
   - Test: `ssh -i ~/.ssh/nest-deploy-key.pem ubuntu@TEST-STATIC-IP`
   - Beta: `ssh -i ~/.ssh/nest-deploy-key.pem ubuntu@BETA-STATIC-IP`

2. **Deploy your app** using the static IPs

3. **Configure your app** to use:
   - `DB_HOST` = RDS endpoint from outputs
   - `S3_BUCKET_NAME` = S3 bucket name from outputs

---

**That's it!** You now have EC2 + RDS + S3 infrastructure ready to use for both Test and Beta environments.

---

## Upgrade EC2 and DB Instance Type (Later)

Use these commands when you want to upgrade the EC2 instance type and RDS DB instance class after initial deployment.
// Get instance ID

aws cloudformation describe-stack-resources \
  --stack-name Krahft-Test \
  --region ap-southeast-2 \
  --query "StackResources[?ResourceType=='AWS::EC2::Instance'].PhysicalResourceId" \
  --output text

aws cloudformation describe-stack-resources \
  --stack-name Krahft-Beta \
  --region ap-southeast-2 \
  --query "StackResources[?ResourceType=='AWS::EC2::Instance'].PhysicalResourceId" \
  --output text

```bash
# 1) Stop EC2 (optional but safe before instance type change)
aws ec2 stop-instances \
  --instance-ids <EC2_INSTANCE_ID> \
  --region ap-southeast-2

# //Status check
aws ec2 describe-instances \
  --instance-ids i-09ee21dd2625e13f6 \
  --region ap-southeast-2 \
  --query "Reservations[0].Instances[0].State.Name" \
  --output text


# 2) Stop RDS (optional; only works for supported RDS setups)
aws rds stop-db-instance \
  --db-instance-identifier krahft-beta-db \
  --region ap-southeast-2

aws rds wait db-instance-stopped \
  --db-instance-identifier krahft-beta-db \
  --region ap-southeast-2


# 3) Upgrade via CloudFormation stack update
#    (change EC2 and DB type; keep other parameters unchanged)
aws cloudformation update-stack \
  --stack-name Krahft-Beta \
  --use-previous-template \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=KeyPairName,UsePreviousValue=true \
    ParameterKey=ProjectName,UsePreviousValue=true \
    ParameterKey=Environment,UsePreviousValue=true \
    ParameterKey=InstanceType,ParameterValue=t3.medium \
    ParameterKey=DBInstanceClass,ParameterValue=db.t3.small \
    ParameterKey=DBName,UsePreviousValue=true \
    ParameterKey=DBMasterPassword,UsePreviousValue=true \
  --region ap-southeast-2


# 4) Wait for update to complete
aws cloudformation wait stack-update-complete \
  --stack-name Krahft-Test \
  --region ap-southeast-2


# 5) Start EC2 and RDS again (if still stopped)
aws ec2 start-instances \
  --instance-ids <EC2_INSTANCE_ID> \
  --region ap-southeast-2

aws rds start-db-instance \
  --db-instance-identifier krahft-test-db \
  --region ap-southeast-2
```
