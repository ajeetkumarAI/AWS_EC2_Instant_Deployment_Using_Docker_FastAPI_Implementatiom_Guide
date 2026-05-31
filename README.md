# AWS_EC2_Instant_Deployment_Using_Docker_FastAPI_Implementatiom_Guide

# Deploy FastAPI + Docker on AWS EC2 using one of AI/ML project

Your app runs on **port 8000** via `uvicorn app:app --host 0.0.0.0 --port 8000`.

---

## Option A: Direct Deploy on EC2 (Simplest — no ECR needed)

### Step 1 — Launch an EC2 Instance

1. Go to **AWS Console → EC2 → Launch Instance**
2. Choose:
   - **AMI**: Ubuntu Server 22.04 LTS (Free Tier eligible)
   - **Instance type**: `t2.micro` (Free Tier) or `t2.small` for better performance
   - **Key pair**: Create or select an existing `.pem` key pair (you'll need this to SSH in)
3. **Security Group** — add these inbound rules:
   | Type       | Port  | Source    | Purpose               |
   |------------|-------|-----------|------------------------|
   | SSH        | 22    | Your IP   | SSH access             |
   | Custom TCP | 8000  | 0.0.0.0/0 | FastAPI app access     |
   | HTTP       | 80    | 0.0.0.0/0 | (Optional) Nginx proxy |
4. Click **Launch Instance**

### Step 2 — SSH into the Instance

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>
```

### Step 3 — Install Docker on EC2

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
# Log out and back in for group change to take effect
exit
```

Then SSH back in.



Step 4: Create and Activate a Virtual Environment

# Create a Python virtual environment named 'venv'
```bash
python3 -m venv venv
 ```

# Activate the virtual environment
# All pip installs will now go into venv/ instead of system Python
```bash
source venv/bin/activate
```

### Step 5 — Clone Your Repo & Build

```bash
git clone https://github.com/ajeetkumarAI/medical-insurance-price_pred-azure-deployment-using-fastapi-docker.git
cd medical-insurance-price_pred-azure-deployment-using-fastapi-docker

docker build -t fastapi-insurance:latest .
```

### Step 6 — Run the Container

```bash
docker run -d -p 8000:8000 --name fastapi-app --restart unless-stopped fastapi-insurance:latest
```

### Step 7 — Access Your App

Open in browser:
```
http://<EC2-PUBLIC-IP>:8000
```

That's it! Your app is live. ✅

---

## Option B: Push to AWS ECR first, then pull on EC2 (like you did with Azure ACR)

This is the AWS equivalent of your Azure workflow (`az acr login` → `docker push` → deploy).

### Step 1 — Install AWS CLI (on your local machine)

Here is the updated README content with both Windows and Linux AWS CLI installation methods included and formatted cleanly:

# Step 1 — Install AWS CLI

Before running the project, install and configure AWS CLI on your local machine.

## Windows Installation

### Requirements

* Microsoft-supported 64-bit Windows version
* Administrator rights to install software

### Install AWS CLI

Download and run the AWS CLI MSI installer:

```bash
https://awscli.amazonaws.com/AWSCLIV2.msi
```

Or install using `msiexec` from Command Prompt:

```cmd
C:\> msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
```

For silent installation:

```cmd
C:\> msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi /qn
```

### Verify Installation

Open Command Prompt and run:

```cmd
C:\> aws --version
```

Example output:

```bash
aws-cli/2.27.41 Python/3.11.6 Windows/10 exe/AMD64 prompt/off
```

If Windows cannot find the command, close and reopen the terminal to refresh the PATH.

### Configure AWS Credentials

```cmd
aws configure
```

Enter:

```text
AWS Access Key ID
AWS Secret Access Key
Default region name (example: ap-south-1)
Default output format (json)
```

---

## Linux Installation

### Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

unzip awscliv2.zip

sudo ./aws/install
```

### Verify Installation

```bash
aws --version
```

Example output:

```bash
aws-cli/2.x.x Python/3.x Linux/x86_64
```

### Configure AWS Credentials

```bash
aws configure
```

Enter:

```text
AWS Access Key ID
AWS Secret Access Key
Default region name (example: ap-south-1)
Default output format (json)
```

### Step 2 — Create an ECR Repository

```bash
aws ecr create-repository --repository-name fastapi-insurance --region ap-south-1
```

This will return a URI like: `123456789012.dkr.ecr.ap-south-1.amazonaws.com/fastapi-insurance`

### Step 3 — Authenticate Docker to ECR

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 417780655856.dkr.ecr.us-east-1.amazonaws.com
#aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

### Step 4 — Tag & Push the Image

```bash
# Build (if not already built)
docker build -t fastapi-insurance:latest .

# Tag for ECR
docker tag fastapi-insurance:latest 123456789012.dkr.ecr.ap-south-1.amazonaws.com/fastapi-insurance:latest

# Push
docker push 123456789012.dkr.ecr.ap-south-1.amazonaws.com/fastapi-insurance:latest
```

### Step 5 — Pull & Run on EC2

SSH into your EC2 instance, install Docker (Step 3 from Option A), then:

```bash
# Install AWS CLI on EC2
sudo apt-get install -y awscli

# Configure AWS credentials on EC2
aws configure

# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 417780655856.dkr.ecr.us-east-1.amazonaws.com
#aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com

# Pull the image
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/fastapi-insurance:latest
#docker pull 123456789012.dkr.ecr.ap-south-1.amazonaws.com/fastapi-insurance:latest

# Run
docker run -d -p 8000:8000 --name fastapi-app --restart unless-stopped 123456789012.dkr.ecr.ap-south-1.amazonaws.com/fastapi-insurance:latest
```

---

## (Optional) Add Nginx as a Reverse Proxy (serve on port 80)

This lets users access your app at `http://<EC2-PUBLIC-IP>` without specifying `:8000`.

```bash
sudo apt-get install -y nginx
```

Create config:

```bash
sudo tee /etc/nginx/sites-available/fastapi <<EOF
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF

sudo ln -sf /etc/nginx/sites-available/fastapi /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

Now your app is accessible at `http://<EC2-PUBLIC-IP>` (port 80).

---

## Azure vs AWS — Side-by-Side Comparison

| Step                     | Azure (what you did)                     | AWS (what to do now)                              |
|--------------------------|------------------------------------------|---------------------------------------------------|
| Container Registry       | Azure Container Registry (ACR)           | Amazon ECR                                        |
| Login to registry        | `az acr login --name mycr1211`           | `aws ecr get-login-password ... \| docker login`  |
| Tag image                | `docker tag ... mycr1211.azurecr.io/...` | `docker tag ... 123456.dkr.ecr.region.amazonaws.com/...` |
| Push image               | `docker push mycr1211.azurecr.io/...`    | `docker push 123456.dkr.ecr.region.amazonaws.com/...`    |
| Deploy                   | Azure Web App (PaaS)                     | EC2 instance (IaaS) — you manage the server       |
| Access app               | `https://yourapp.azurewebsites.net`      | `http://<EC2-PUBLIC-IP>:8000`                     |

---

## Useful Commands

```bash
# Check running containers
docker ps

# View logs
docker logs fastapi-app

# Stop the app
docker stop fastapi-app

# Restart the app
docker start fastapi-app

# Rebuild after code changes
docker stop fastapi-app && docker rm fastapi-app
docker build -t fastapi-insurance:latest .
docker run -d -p 8000:8000 --name fastapi-app --restart unless-stopped fastapi-insurance:latest
```

---

## Important Notes

- **Elastic IP**: By default, your EC2 public IP changes on reboot. Go to **EC2 → Elastic IPs → Allocate** and associate it with your instance to get a fixed IP.
- **Security**: For production, add HTTPS using Let's Encrypt + Nginx or put an AWS ALB in front.
- **Cost**: `t2.micro` is Free Tier eligible for 12 months (750 hrs/month).
- **Your Dockerfile and code need zero changes** — the same image works on both Azure and AWS.
