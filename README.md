# 🚀 Spring Boot CI/CD Pipeline — Complete Step-by-Step Guide

> **Stack:** Java 17 + Spring Boot + Maven + Docker + GitHub Actions + AWS ECR + AWS EC2
> **Level:** Beginner friendly — every small step included

---

## 📋 Table of Contents

1. [What You Will Build](#what-you-will-build)
2. [Prerequisites](#prerequisites)
3. [Phase 1 — Create Spring Boot App](#phase-1--create-spring-boot-app)
4. [Phase 2 — Dockerize the App](#phase-2--dockerize-the-app)
5. [Phase 3 — Push to GitHub](#phase-3--push-to-github)
6. [Phase 4 — Set Up AWS](#phase-4--set-up-aws)
7. [Phase 5 — Add GitHub Secrets](#phase-5--add-github-secrets)
8. [Phase 6 — Create GitHub Actions Workflow](#phase-6--create-github-actions-workflow)
9. [Phase 7 — Trigger and Verify](#phase-7--trigger-and-verify)
10. [Phase 8 — Test the Full Pipeline](#phase-8--test-the-full-pipeline)
11. [Troubleshooting](#troubleshooting)
12. [What You Learned](#what-you-learned)

---

## What You Will Build

Every time you push code to GitHub, this happens **automatically:**

```
You push code → GitHub Actions triggers → Maven builds jar
→ Tests run → Docker image built → Image pushed to AWS ECR
→ EC2 pulls new image → App restarts → Live at your EC2 IP 🎉
```

---

## Prerequisites

### On Your Windows Laptop
| Tool | Purpose | Download |
|------|---------|---------|
| Git Bash | Terminal to SSH into EC2 | git-scm.com |
| VS Code | Code editor | code.visualstudio.com |

### On Your AWS EC2 Instance (Ubuntu 24)
Verify each is installed by running these commands after SSHing in:

```bash
java -version        # Must show Java 17
mvn -version         # Must show Apache Maven
git --version        # Must show git version
docker --version     # Must show Docker version
aws --version        # Must show aws-cli/2.x.x
```

---

## Connect to EC2 from Git Bash

Open Git Bash on your Windows laptop:

```bash
ssh -i /c/Users/YourName/Downloads/your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

> ⚠️ If you get a permissions error, run:
> ```bash
> chmod 400 /c/Users/YourName/Downloads/your-key.pem
> ```

---

## Phase 1 — Create Spring Boot App

### Step 1.1 — Fix Java Version (if needed)

If `java -version` shows Java 11 or lower, run:

```bash
# Install Java 17
sudo apt update
sudo apt install -y openjdk-17-jdk

# Set Java 17 as default
sudo update-alternatives --config java
# Type the number for Java 17 and press Enter

# Set JAVA_HOME
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# Verify
java -version
# Should show: openjdk version "17.x.x"
```

### Step 1.2 — Generate Spring Boot Project

```bash
# Go to home directory
cd ~

# Create project folder
mkdir springboot-cicd && cd springboot-cicd

# Download Spring Boot starter
curl https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.2.5 \
  -d baseDir=demo \
  -d groupId=com.example \
  -d artifactId=demo \
  -d dependencies=web \
  -o demo.zip

# Unzip it
unzip demo.zip
cd demo
```

### Step 1.3 — Add a REST Endpoint

```bash
nano src/main/java/com/example/demo/DemoApplication.java
```

Delete everything and paste this:

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @GetMapping("/")
    public String hello() {
        return "Hello from my CI/CD Pipeline!";
    }
}
```

Save: `Ctrl+O` → Enter → `Ctrl+X`

### Step 1.4 — Build and Test Locally

```bash
# Build the jar
mvn clean package -DskipTests

# Run the app
java -jar target/demo-0.0.1-SNAPSHOT.jar
```

You should see:
```
Started DemoApplication in 3.4 seconds
```

Open a **second Git Bash window**, SSH in again, and test:

```bash
curl http://localhost:8080
# Output: Hello from my CI/CD Pipeline!
```

Stop the app with `Ctrl+C`

> 💡 **To test in browser:** Open port 8080 in EC2 Security Group (AWS Console → EC2 → Security Groups → Edit Inbound Rules → Add port 8080) then visit `http://YOUR_EC2_IP:8080`

---

## Phase 2 — Dockerize the App

### Step 2.1 — Create Dockerfile

```bash
# Make sure you're in the right folder
cd ~/springboot-cicd/demo

nano Dockerfile
```

Paste this:

```dockerfile
FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Save: `Ctrl+O` → Enter → `Ctrl+X`

### Step 2.2 — Build Docker Image

```bash
docker build -t my-spring-app .
```

You will see Docker downloading layers and building. Wait until you see `Successfully built`.

### Step 2.3 — Run Docker Container

```bash
docker run -d --name my-appy -p 3000:8080 my-spring-app
```

> Note: `-p 3000:8080` means EC2 port 3000 maps to container port 8080

### Step 2.4 — Verify Container is Running

```bash
docker ps
```

You should see:
```
CONTAINER ID   IMAGE           COMMAND               STATUS        PORTS
82ada617861f   my-spring-app   "java -jar app.jar"   Up 2 mins    0.0.0.0:3000->8080/tcp
```

Test it:
```bash
curl http://localhost:3000
# Output: Hello from my CI/CD Pipeline!
```

### Step 2.5 — Open Port 3000 in AWS Security Group

1. AWS Console → EC2 → your instance → **Security** tab
2. Click the Security Group link
3. **Edit inbound rules** → **Add rule**
   - Type: `Custom TCP`
   - Port: `3000`
   - Source: `Anywhere IPv4 (0.0.0.0/0)`
4. **Save rules**

Visit in browser: `http://YOUR_EC2_IP:3000` ✅

---

## Phase 3 — Push to GitHub

### Step 3.1 — Create GitHub Repository

1. Go to **github.com** → click **+** (top right) → **New repository**
2. Repository name: `springboot-cicd`
3. Visibility: **Public**
4. ❌ Do NOT add README or .gitignore
5. Click **Create repository**

### Step 3.2 — Create a Personal Access Token

GitHub no longer accepts plain passwords. You need a token:

1. GitHub → click your profile picture → **Settings**
2. Scroll to bottom → **Developer settings**
3. **Personal access tokens** → **Tokens (classic)**
4. **Generate new token (classic)**
5. Name: `ec2-token`
6. Expiry: `90 days`
7. Check these scopes:
   - ✅ `repo`
   - ✅ `workflow`
8. Click **Generate token**
9. **Copy the token immediately** — it starts with `ghp_...`
   > ⚠️ You will NEVER see this token again after leaving the page!

### Step 3.3 — Configure Git on EC2

```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

### Step 3.4 — Push Code to GitHub

```bash
cd ~/springboot-cicd/demo

# Initialize git
git init

# Add all files
git add .

# First commit
git commit -m "Initial Spring Boot app with Dockerfile"

# Add your GitHub repo as remote
git remote add origin https://github.com/YOUR_USERNAME/springboot-cicd.git

# Push
git branch -M main
git push -u origin main
```

When prompted:
- Username: your GitHub username
- Password: paste your `ghp_...` token (it won't show — just paste and press Enter)

---

## Phase 4 — Set Up AWS

### Step 4.1 — Create ECR Repository

1. AWS Console → search **ECR** → **Elastic Container Registry**
2. Click **Create repository**
3. Visibility: **Private**
4. Repository name: `my-spring-app`
5. Click **Create repository**
6. **Copy the repository URI** — looks like:
   ```
   123456789012.dkr.ecr.ap-south-2.amazonaws.com/my-spring-app
   ```

### Step 4.2 — Create IAM User for GitHub Actions

1. AWS Console → search **IAM** → **Users** → **Create user**
2. Username: `github-actions-user`
3. Click **Next** → **Attach policies directly**
4. Search and check these 2 policies:
   - ✅ `AmazonEC2ContainerRegistryFullAccess`
   - ✅ `AmazonEC2FullAccess`
5. Click **Next** → **Create user**

### Step 4.3 — Create Access Keys

1. Click on `github-actions-user` → **Security credentials** tab
2. Scroll down → **Create access key**
3. Select **Application running outside AWS** → **Next**
4. Click **Create access key**
5. **Copy both values immediately:**
   - Access key ID (starts with `AKIA...`)
   - Secret access key (long random string)
   > ⚠️ You will NEVER see the secret key again!

### Step 4.4 — Install AWS CLI on EC2

```bash
# Download and install
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version
# Should show: aws-cli/2.x.x
```

### Step 4.5 — Configure AWS CLI on EC2

```bash
aws configure
```

Enter when prompted:
```
AWS Access Key ID: YOUR_ACCESS_KEY_ID
AWS Secret Access Key: YOUR_SECRET_ACCESS_KEY
Default region name: ap-south-2
Default output format: json
```

---

## Phase 5 — Add GitHub Secrets

Go to your GitHub repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Add each secret one by one:

| Secret Name | Value | Where to get it |
|-------------|-------|----------------|
| `AWS_ACCESS_KEY_ID` | `AKIA...` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | long string | IAM user secret key |
| `AWS_REGION` | `ap-south-2` | Your AWS region |
| `ECR_REPOSITORY` | `my-spring-app` | ECR repo name |
| `EC2_HOST` | `18.x.x.x` | Your EC2 public IP |
| `EC2_SSH_KEY` | full .pem contents | See below |

### How to get EC2_SSH_KEY value

Open Git Bash on your Windows laptop (not EC2):

```bash
cat /c/Users/YourName/Downloads/your-key.pem
```

Copy the **entire output** including the header and footer:
```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA1234abcd...
...many lines...
-----END RSA PRIVATE KEY-----
```

Paste this as the `EC2_SSH_KEY` secret value.

> ✅ You should now have all 6 secrets listed in GitHub

---

## Phase 6 — Create GitHub Actions Workflow

### Step 6.1 — Create Workflow File

```bash
cd ~/springboot-cicd/demo

# Create the folders
mkdir -p .github/workflows

# Create the workflow file
nano .github/workflows/cicd.yml
```

Paste this entire content:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build with Maven
        run: mvn clean package -DskipTests

      - name: Run tests
        run: mvn test

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image to ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/${{ secrets.ECR_REPOSITORY }}:$IMAGE_TAG .
          docker push $ECR_REGISTRY/${{ secrets.ECR_REPOSITORY }}:$IMAGE_TAG
          echo "IMAGE=$ECR_REGISTRY/${{ secrets.ECR_REPOSITORY }}:$IMAGE_TAG" >> $GITHUB_ENV

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            aws ecr get-login-password --region ${{ secrets.AWS_REGION }} | \
              docker login --username AWS --password-stdin ${{ steps.login-ecr.outputs.registry }}
            docker pull ${{ env.IMAGE }}
            docker stop my-appy || true
            docker rm my-appy || true
            docker run -d --name my-appy -p 3000:8080 ${{ env.IMAGE }}
```

Save: `Ctrl+O` → Enter → `Ctrl+X`

### Step 6.2 — Push Workflow to GitHub

```bash
git add .
git commit -m "Add GitHub Actions CI/CD workflow"
git push origin main
```

---

## Phase 7 — Trigger and Verify

### Step 7.1 — Watch the Pipeline Run

1. Go to your GitHub repo
2. Click the **Actions** tab
3. You'll see a workflow run in progress
4. Click on it to see each step

Each step will show:
- 🟡 Yellow = running
- ✅ Green = passed
- ❌ Red = failed

### Step 7.2 — All steps should pass

```
✅ Checkout code
✅ Set up JDK 17
✅ Build with Maven
✅ Run tests
✅ Configure AWS credentials
✅ Login to Amazon ECR
✅ Build and push Docker image to ECR
✅ Deploy to EC2
```

### Step 7.3 — Verify App is Live

Open in your browser:
```
http://YOUR_EC2_PUBLIC_IP:3000
```

You should see: **Hello from my CI/CD Pipeline!** 🎉

---

## Phase 8 — Test the Full Pipeline

This is the most exciting part — make a change and watch it auto-deploy!

### Step 8.1 — Change the code

```bash
cd ~/springboot-cicd/demo
nano src/main/java/com/example/demo/DemoApplication.java
```

Change this line:
```java
return "Hello from my CI/CD Pipeline!";
```

To:
```java
return "Version 2 - Auto Deployed by CI/CD! 🚀";
```

Save: `Ctrl+O` → Enter → `Ctrl+X`

### Step 8.2 — Push the change

```bash
git add .
git commit -m "Update message to v2"
git push origin main
```

### Step 8.3 — Watch it auto-deploy

1. Go to GitHub → **Actions tab**
2. Watch the pipeline run automatically
3. Once it's green, refresh your browser at `http://YOUR_EC2_IP:3000`
4. You'll see: **Version 2 - Auto Deployed by CI/CD! 🚀**

> You did NOT touch the server manually — the pipeline did everything! That's CI/CD! 🏆

---

## Troubleshooting

### ❌ `release version 17 not supported`
```bash
sudo apt install -y openjdk-17-jdk
sudo update-alternatives --config java
# Select Java 17 from the list
```

### ❌ `docker: permission denied`
```bash
sudo usermod -aG docker ubuntu
# Log out and SSH back in
```

### ❌ `repository not found` on git push
```bash
git remote remove origin
git remote add origin https://github.com/YOUR_ACTUAL_USERNAME/springboot-cicd.git
git push -u origin main
```

### ❌ `refusing to allow... without workflow scope`
- Go to GitHub → Settings → Developer settings → Personal access tokens
- Edit your token → check ✅ `workflow` scope → Update → copy new token
- Use new token when pushing

### ❌ `aws: command not found` in GitHub Actions
```bash
# Run this on your EC2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws configure
```

### ❌ `Credentials could not be loaded`
- Double-check secret names in GitHub — they are case sensitive
- Must be exactly: `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
- Delete and re-create the secrets with fresh IAM access keys

### ❌ `invalid reference format` (ECR image tag)
- `ECR_REPOSITORY` secret is missing
- Go to GitHub Secrets → add `ECR_REPOSITORY` with value `my-spring-app`

### ❌ `missing server host`
- `EC2_HOST` secret is missing
- Go to GitHub Secrets → add `EC2_HOST` with your EC2 public IP

---

## What You Learned

| Concept | Tool Used |
|---------|-----------|
| Version control | Git + GitHub |
| Automated build | Apache Maven |
| Automated testing | JUnit (via `mvn test`) |
| Containerization | Docker + Dockerfile |
| Container registry | Amazon ECR |
| Cloud server | AWS EC2 (Free Tier) |
| CI/CD automation | GitHub Actions |
| Infrastructure | AWS Free Tier |

---

## Key Commands Cheat Sheet

```bash
# SSH into EC2
ssh -i your-key.pem ubuntu@YOUR_EC2_IP

# Build Spring Boot app
mvn clean package -DskipTests

# Run app locally
java -jar target/demo-0.0.1-SNAPSHOT.jar

# Build Docker image
docker build -t my-spring-app .

# Run Docker container
docker run -d --name my-appy -p 3000:8080 my-spring-app

# Check running containers
docker ps

# View container logs
docker logs my-appy

# Stop container
docker stop my-appy

# Push code to GitHub (triggers pipeline)
git add .
git commit -m "your message"
git push origin main
```

---

## The Pipeline Flow (Summary)

```
┌─────────────┐     git push      ┌──────────────────┐
│  Your Code  │ ───────────────►  │   GitHub Actions  │
│  on EC2     │                   │   (triggered)     │
└─────────────┘                   └────────┬─────────┘
                                           │
                              ┌────────────▼────────────┐
                              │  1. Maven build & test   │
                              │  2. Docker image build   │
                              │  3. Push to AWS ECR      │
                              │  4. SSH into EC2         │
                              │  5. Pull & run new image │
                              └────────────┬────────────┘
                                           │
                              ┌────────────▼────────────┐
                              │   App live on EC2! 🎉   │
                              │  http://EC2_IP:3000      │
                              └─────────────────────────┘
```

---

*Guide created during hands-on practice session — June 2026*
