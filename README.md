# AWS EC2 Deployment Using Docker & FastAPI — Implementation Guide

A complete guide to deploy a FastAPI + Docker application on **AWS EC2**, with two options:

- **Option A** — Clone repo directly on EC2 and build there (simplest)
- **Option B** — Push image to AWS ECR, then pull and run on EC2 (like Azure ACR workflow)

Your app runs on **port 8000** via `uvicorn app:app --host 0.0.0.0 --port 8000`.

---

## Option A: Direct Deploy on EC2 (Simplest — No ECR Needed)

### Step 1 — Launch an EC2 Instance

1. Go to **AWS Console → EC2 → Launch Instance**
2. Choose:
   - **AMI**: Ubuntu Server 22.04 LTS (Free Tier eligible)
   - **Instance type**: `t2.micro` (Free Tier) or `t3.micro` for better performance
   - **Key pair**: Create or select an existing `.pem` key pair (you'll need this to SSH in)
3. **Security Group** — add these inbound rules:

   | Type       | Port  | Source     | Purpose                |
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

### Step 4 — Clone Your Repo & Build

```bash
git clone https://github.com/ajeetkumarAI/medical-insurance-price_pred-azure-deployment-using-fastapi-docker.git
cd medical-insurance-price_pred-azure-deployment-using-fastapi-docker

sudo docker build -t fastapi-insurance:latest .
```

### Step 5 — Run the Container

```bash
sudo docker run -d -p 8000:8000 --name fastapi-app --restart unless-stopped fastapi-insurance:latest
```

### Step 6 — Access Your App

Open in browser:

```
http://<EC2-PUBLIC-IP>:8000
```

> **Note:** Use `http://` not `https://`. Your app does not have SSL configured, so `https://` will give an `ERR_SSL_PROTOCOL_ERROR`.

---

## Option B: Push to AWS ECR First, Then Pull on EC2

This is the AWS equivalent of your Azure workflow:
`az acr login` → `docker push` → deploy becomes `aws ecr get-login-password` → `docker push` → deploy.

---

### Part 1 — Install AWS CLI (On Your Local Machine)

#### Windows

Download and run the AWS CLI MSI installer:

```
https://awscli.amazonaws.com/AWSCLIV2.msi
```

Or install using `msiexec` from Command Prompt:

```cmd
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
```

Verify installation:

```cmd
aws --version
```

Example output:

```
aws-cli/2.27.41 Python/3.11.6 Windows/10 exe/AMD64 prompt/off
```

If Windows cannot find the command, close and reopen the terminal to refresh the PATH.

#### Linux

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verify installation:

```bash
aws --version
```

#### Configure AWS Credentials (Both Platforms)

```bash
aws configure
```

Enter:

```
AWS Access Key ID:      <your-access-key>
AWS Secret Access Key:  <your-secret-key>
Default region name:    us-east-1
Default output format:  json
```

---

### Part 2 — Create an ECR Repository

```bash
aws ecr create-repository --repository-name myfirstcontainerregistry001 --region us-east-1
```

This returns a Repository URI like:

```
417780655856.dkr.ecr.us-east-1.amazonaws.com/myfirstcontainerregistry001
```

---

### Part 3 — Authenticate Docker to ECR (Locally)

**On Windows CMD:**

```cmd
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 417780655856.dkr.ecr.us-east-1.amazonaws.com
```

**On Linux/macOS:**

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 417780655856.dkr.ecr.us-east-1.amazonaws.com
```

> **Important:** Do NOT use the PowerShell command `(Get-ECRLoginCommand).Password | ...` in CMD — it will fail. Use the `aws ecr get-login-password` command shown above.

---

### Part 4 — Build, Tag & Push the Docker Image

```bash
# Build the image
docker build -t myfirstcontainerregistry001 .

# Tag for ECR
docker tag myfirstcontainerregistry001:latest 417780655856.dkr.ecr.us-east-1.amazonaws.com/myfirstcontainerregistry001:latest

# Push to ECR
docker push 417780655856.dkr.ecr.us-east-1.amazonaws.com/myfirstcontainerregistry001:latest
```

---

### Part 5 — On EC2: Install Prerequisites

SSH into your EC2 instance and install Docker + AWS CLI:

```bash
# Install Docker
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker

# Install AWS CLI v2
sudo apt-get install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version

# Configure AWS credentials on EC2
aws configure
```

---

### Part 6 — On EC2: Login to ECR, Pull & Run

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | sudo docker login --username AWS --password-stdin 417780655856.dkr.ecr.us-east-1.amazonaws.com

# Pull the image from ECR
sudo docker pull 417780655856.dkr.ecr.us-east-1.amazonaws.com/myfirstcontainerregistry001:latest

# Run the container
sudo docker run -d -p 8000:8000 --name fastapi-app --restart unless-stopped 417780655856.dkr.ecr.us-east-1.amazonaws.com/myfirstcontainerregistry001:latest
```

### Part 7 — Access Your App

```
http://<EC2-PUBLIC-IP>:8000
```

---

## (Optional) Add Nginx as a Reverse Proxy (Serve on Port 80)

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

> Make sure your Security Group allows inbound traffic on port 80.

---

## Azure vs AWS — Side-by-Side Comparison

| Step                | Azure (ACR + Web App)                    | AWS (ECR + EC2)                                        |
|---------------------|------------------------------------------|--------------------------------------------------------|
| Container Registry  | Azure Container Registry (ACR)           | Amazon ECR                                             |
| Login to registry   | `az acr login --name mycr1211`           | `aws ecr get-login-password ... \| docker login`       |
| Tag image           | `docker tag ... mycr1211.azurecr.io/...` | `docker tag ... 417780655856.dkr.ecr.us-east-1.amazonaws.com/...` |
| Push image          | `docker push mycr1211.azurecr.io/...`    | `docker push 417780655856.dkr.ecr.us-east-1.amazonaws.com/...`    |
| Deploy              | Azure Web App (PaaS — managed)           | EC2 instance (IaaS — you manage the server)            |
| Access app          | `https://yourapp.azurewebsites.net`      | `http://<EC2-PUBLIC-IP>:8000`                          |

---

## Useful Docker Commands

```bash
# Check running containers
sudo docker ps

# View container logs
sudo docker logs fastapi-app

# Stop the app
sudo docker stop fastapi-app

# Restart the app
sudo docker start fastapi-app

# Rebuild after code changes
sudo docker stop fastapi-app && sudo docker rm fastapi-app
sudo docker build -t fastapi-insurance:latest .
sudo docker run -d -p 8000:8000 --name fastapi-app --restart unless-stopped fastapi-insurance:latest
```

---

## Important Notes

- **Use `http://` not `https://`** — Your app has no SSL certificate. `https://` will show `ERR_SSL_PROTOCOL_ERROR`. Use `http://<EC2-PUBLIC-IP>:8000`.
- **Elastic IP** — By default, your EC2 public IP changes on reboot. Go to **EC2 → Elastic IPs → Allocate** and associate it with your instance to get a fixed IP.
- **Security** — For production, add HTTPS using Let's Encrypt + Nginx or put an AWS ALB in front.
- **Cost** — `t2.micro` / `t3.micro` are Free Tier eligible for 12 months (750 hrs/month).
- **Your Dockerfile and code need zero changes** — the same image works on both Azure and AWS.

---

## API Endpoints

| Endpoint  | Method | Description                    |
|-----------|--------|--------------------------------|
| `/`       | GET    | Home page with prediction form |
| `/predict`| POST   | Predict insurance charges      |
| `/health` | GET    | Health check                   |
