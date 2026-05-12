# 🚀 Complete Deployment Guide
## Next.js + PostgreSQL + AWS S3 + Docker + EC2 + GitHub Actions CI/CD

> **Who is this for?** A fresher who has built a Next.js app (with API routes only, no separate backend) and needs to host it professionally using Docker, AWS EC2, AWS S3, PostgreSQL, and GitHub Actions CI/CD.

---

## 📐 Architecture Overview (Read This First!)

```
Your Machine (Code)
      |
      | git push
      ↓
GitHub Repository
      |
      | triggers automatically
      ↓
GitHub Actions (CI/CD Pipeline)
      |
      ├── Build Docker Image of your Next.js App
      ├── Push Image → Docker Hub (free registry)
      └── SSH into EC2 → Pull new image → Restart containers
                              |
                        AWS EC2 Server
                        ┌─────────────────────────────────┐
                        │  Docker Compose running:         │
                        │  ┌─────────────────────┐         │
                        │  │  Next.js App Container│         │
                        │  │  (port 3000)         │         │
                        │  └──────────┬──────────┘         │
                        │             │                     │
                        │  ┌──────────▼──────────┐         │
                        │  │ PostgreSQL Container  │         │
                        │  │  (port 5432)         │         │
                        │  └─────────────────────┘         │
                        └─────────────────────────────────┘
                                     │
                              AWS S3 Bucket
                         (stores uploaded files/images)
```

**What you need accounts for:**
- GitHub (free)
- Docker Hub (free) → https://hub.docker.com
- AWS Account (EC2 + S3)

---

## 📋 Phase 0: Accounts & Tools to Set Up First

### 0.1 — On Your Local Machine (Windows/Mac/Linux)

Install these if not already installed:

| Tool | What it's for | Download Link |
|------|---------------|---------------|
| Git | Push code to GitHub | https://git-scm.com |
| Node.js (v18+) | Run Next.js locally | https://nodejs.org |
| Docker Desktop | Test Docker locally | https://www.docker.com/products/docker-desktop |
| VS Code | Code editor | https://code.visualstudio.com |

### 0.2 — Create Docker Hub Account

1. Go to → https://hub.docker.com
2. Click **Sign Up** → use your email
3. Create a username (example: `yourname`) — **remember this username!**
4. Create a **repository**:
   - Click **Repositories** → **Create Repository**
   - Name it: `my-nextjs-app`
   - Set to **Public** (free)
   - Click **Create**

---

## 📦 Phase 1: Prepare Your Next.js Project

### 1.1 — Your Project Folder Structure

Your project should look like this (approximately):

```
my-app/
├── app/               ← or pages/ (Next.js 13+ uses app/)
│   ├── api/           ← your API routes
│   └── page.tsx
├── public/
├── .env.local         ← local environment variables (NEVER commit this)
├── .env.example       ← example env file (safe to commit)
├── next.config.js
├── package.json
└── tsconfig.json      ← only if using TypeScript
```

### 1.2 — Update `next.config.js` ⚠️ CRITICAL STEP

This tells Next.js to build in a way that works inside Docker:

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',   // ← THIS IS THE MOST IMPORTANT LINE
  // Add your other existing config below this
};

module.exports = nextConfig;
```

> ⚠️ **Why?** Without `output: 'standalone'`, the Docker image will be huge (1GB+) and may not run correctly. With it, it creates a minimal self-contained build.

### 1.3 — Create `.env.example` File

This file is committed to GitHub so your teammates know what variables are needed (but never put real values here):

```env
# .env.example

# Database
DATABASE_URL=postgresql://username:password@localhost:5432/mydb

# AWS S3
AWS_ACCESS_KEY_ID=your_access_key_here
AWS_SECRET_ACCESS_KEY=your_secret_key_here
AWS_REGION=ap-south-1
AWS_S3_BUCKET_NAME=your-bucket-name

# Next.js
NEXTAUTH_SECRET=some_random_secret_here
NEXTAUTH_URL=http://localhost:3000

# App
NODE_ENV=development
```

### 1.4 — Create `.gitignore` File

Make sure these are in your `.gitignore` (the `.gitignore` that `create-next-app` makes is usually fine, but double-check):

```
# .gitignore
node_modules/
.next/
.env
.env.local
.env.production
*.log
```

---

## 🐳 Phase 2: Dockerize Your Next.js App

### 2.1 — Create `Dockerfile` in Root of Project

```dockerfile
# Dockerfile

# ── Stage 1: Install dependencies ──────────────────────────────
FROM node:18-alpine AS deps

# Install libc compatibility for Alpine
RUN apk add --no-cache libc6-compat

WORKDIR /app

# Copy package files first (for Docker layer caching)
COPY package.json package-lock.json* ./

# Install only production dependencies
RUN npm ci

# ── Stage 2: Build the application ─────────────────────────────
FROM node:18-alpine AS builder

WORKDIR /app

# Copy node_modules from deps stage
COPY --from=deps /app/node_modules ./node_modules

# Copy all source code
COPY . .

# Build arguments (passed during docker build)
ARG DATABASE_URL
ARG AWS_ACCESS_KEY_ID
ARG AWS_SECRET_ACCESS_KEY
ARG AWS_REGION
ARG AWS_S3_BUCKET_NAME
ARG NEXTAUTH_SECRET
ARG NEXTAUTH_URL

# Set env vars for build time
ENV DATABASE_URL=$DATABASE_URL
ENV AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
ENV AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
ENV AWS_REGION=$AWS_REGION
ENV AWS_S3_BUCKET_NAME=$AWS_S3_BUCKET_NAME
ENV NEXTAUTH_SECRET=$NEXTAUTH_SECRET
ENV NEXTAUTH_URL=$NEXTAUTH_URL
ENV NEXT_TELEMETRY_DISABLED=1

# Build Next.js
RUN npm run build

# ── Stage 3: Run the application ────────────────────────────────
FROM node:18-alpine AS runner

WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

# Create a non-root user for security
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy built output from builder stage
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

# Switch to non-root user
USER nextjs

# Expose port 3000
EXPOSE 3000

ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

# Start the app
CMD ["node", "server.js"]
```

### 2.2 — Create `.dockerignore` in Root of Project

This prevents sending unnecessary files to Docker (makes builds much faster):

```
# .dockerignore
node_modules
.next
.git
.gitignore
README.md
.env
.env.local
.env.production
*.log
Dockerfile
docker-compose*.yml
.dockerignore
```

### 2.3 — Create `docker-compose.yml` for LOCAL Development

This lets you test the full setup on your laptop (Next.js + PostgreSQL together):

```yaml
# docker-compose.yml  ← use this ONLY on your local machine for testing

version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: local_postgres
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: nextjs_app
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://myuser:mypassword@postgres:5432/mydb
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
      AWS_REGION: ${AWS_REGION}
      AWS_S3_BUCKET_NAME: ${AWS_S3_BUCKET_NAME}
      NODE_ENV: production
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres_data:
```

### 2.4 — Create `docker-compose.prod.yml` for EC2 Production

This is what will run on your EC2 server:

```yaml
# docker-compose.prod.yml  ← this goes on EC2 server

version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: prod_postgres
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    # No ports exposed to outside - only accessible within Docker network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    image: YOUR_DOCKERHUB_USERNAME/my-nextjs-app:latest  # ← REPLACE with your Docker Hub username
    container_name: nextjs_app
    restart: always
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
      AWS_REGION: ${AWS_REGION}
      AWS_S3_BUCKET_NAME: ${AWS_S3_BUCKET_NAME}
      NODE_ENV: production
      NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
      NEXTAUTH_URL: ${NEXTAUTH_URL}
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres_data:
```

### 2.5 — Test Docker Locally

Open terminal in your project folder and run:

```bash
# Build and start locally
docker compose up --build

# Open browser → http://localhost:3000
# You should see your app running!

# To stop:
docker compose down
```

> ✅ **If it works locally, it will work on EC2!** Never skip this step.

---

## ☁️ Phase 3: AWS Setup

### 3.1 — Create an IAM User (For GitHub Actions to access AWS)

1. Log in to → https://console.aws.amazon.com
2. Search for **IAM** in the top search bar → click it
3. Click **Users** (left sidebar) → **Create user**
4. Username: `github-actions-user` → click **Next**
5. Select **Attach policies directly**
6. Search and add these two policies:
   - `AmazonS3FullAccess`
   - `AmazonEC2FullAccess` ← optional, needed only if you automate EC2 setup
7. Click **Next** → **Create user**
8. Click on the user you just created → **Security credentials** tab
9. Click **Create access key**
10. Select **Application running outside AWS** → **Next** → **Create**
11. ⚠️ **COPY BOTH KEYS NOW — you will not see the secret key again!**
    - `Access Key ID` → looks like `AKIAIOSFODNN7EXAMPLE`
    - `Secret Access Key` → looks like `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`

### 3.2 — Create an S3 Bucket

1. Search for **S3** in AWS → click it
2. Click **Create bucket**
3. **Bucket name**: `my-app-uploads-yourname` (must be globally unique)
4. **Region**: Choose `ap-south-1` (Mumbai) if you're in India — keep this consistent!
5. **Block Public Access settings**: 
   - If your app serves files directly from S3 to users → uncheck "Block all public access"
   - If your app uses pre-signed URLs (recommended) → keep it blocked
6. Click **Create bucket**
7. **Set CORS** (needed for file uploads from browser):
   - Click your bucket → **Permissions** tab → scroll to **CORS** → **Edit**
   - Paste this:
   ```json
   [
     {
       "AllowedHeaders": ["*"],
       "AllowedMethods": ["GET", "PUT", "POST", "DELETE"],
       "AllowedOrigins": ["http://localhost:3000", "https://yourdomain.com"],
       "ExposeHeaders": []
     }
   ]
   ```
   - Click **Save changes**

### 3.3 — Launch an EC2 Instance

1. Search for **EC2** in AWS → click it
2. Click **Launch instance**
3. Fill in:
   - **Name**: `my-nextjs-server`
   - **AMI (OS)**: Ubuntu Server 22.04 LTS *(pick the one that says "Free tier eligible")*
   - **Instance type**: `t2.micro` (free tier) or `t3.small` for better performance
   - **Key pair**: Click **Create new key pair**
     - Name: `my-server-key`
     - Type: **RSA**
     - Format: **.pem** (for Mac/Linux) or **.ppk** (for Windows with PuTTY)
     - Click **Create key pair** → file downloads automatically
     - ⚠️ **Save this `.pem` file safely — if you lose it, you cannot access your server!**
   - **Network settings**: Check these boxes:
     - ✅ Allow SSH traffic from: **My IP** (more secure)
     - ✅ Allow HTTPS traffic
     - ✅ Allow HTTP traffic
4. Click **Launch instance**
5. Wait ~1 minute → Click **View all instances**
6. Wait until **Instance State** shows **Running** and **Status check** shows **2/2 checks passed**
7. **Copy the Public IPv4 address** — example: `65.0.123.45` — save this!

### 3.4 — Add Custom Port to EC2 Security Group

Your Next.js app runs on port 3000. You need to open that port:

1. Click on your instance → scroll down → click **Security** tab
2. Click the **Security group** link (looks like `sg-0abc123...`)
3. Click **Edit inbound rules** → **Add rule**:
   - Type: **Custom TCP**
   - Port range: **3000**
   - Source: **Anywhere-IPv4** (0.0.0.0/0)
4. Click **Save rules**

### 3.5 — Connect to EC2 and Install Docker

Open your terminal and navigate to where your `.pem` file was downloaded:

```bash
# First, fix permissions on the key file (Mac/Linux only)
chmod 400 ~/Downloads/my-server-key.pem

# SSH into your server (replace YOUR_EC2_IP with your actual IP)
ssh -i ~/Downloads/my-server-key.pem ubuntu@YOUR_EC2_IP
```

> 💡 **Windows users**: Use Windows Terminal or PowerShell. The `chmod` command is not needed on Windows. Or use PuTTY with the .ppk file.

Once connected, you'll see a terminal showing `ubuntu@ip-...:~$`. Now run these commands **one by one**:

```bash
# 1. Update system packages
sudo apt-get update && sudo apt-get upgrade -y

# 2. Install required tools
sudo apt-get install -y ca-certificates curl gnupg

# 3. Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 4. Set up Docker repository
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. Install Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 6. Allow ubuntu user to run Docker without sudo
sudo usermod -aG docker ubuntu

# 7. Start Docker service
sudo systemctl start docker
sudo systemctl enable docker

# 8. Verify Docker is installed
docker --version
# You should see: Docker version 24.x.x, build xxxxx

# 9. EXIT and reconnect (needed for group change to take effect)
exit
```

Reconnect:
```bash
ssh -i ~/Downloads/my-server-key.pem ubuntu@YOUR_EC2_IP

# Verify Docker works without sudo
docker ps
# Should show empty table (no error)
```

### 3.6 — Create the App Directory and .env File on EC2

Still in the EC2 terminal:

```bash
# Create a directory for your app
mkdir -p /home/ubuntu/app
cd /home/ubuntu/app

# Create production .env file (fill in your actual values)
nano .env
```

In the nano editor, type your actual values:

```env
# Database
POSTGRES_USER=myuser
POSTGRES_PASSWORD=YourStrongPassword123!
POSTGRES_DB=mydb

# AWS S3
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_REGION=ap-south-1
AWS_S3_BUCKET_NAME=my-app-uploads-yourname

# Next.js
NEXTAUTH_SECRET=generate_a_random_64_char_string_here
NEXTAUTH_URL=http://YOUR_EC2_IP:3000
NODE_ENV=production
```

> 💡 To generate a random secret: run `openssl rand -base64 64` in your terminal

Save with `Ctrl+X` → `Y` → `Enter`

```bash
# Protect the .env file (only owner can read it)
chmod 600 .env
```

Now copy your `docker-compose.prod.yml` file to EC2:
```bash
# Run this from your LOCAL machine (not EC2)
scp -i ~/Downloads/my-server-key.pem docker-compose.prod.yml ubuntu@YOUR_EC2_IP:/home/ubuntu/app/
```

---

## 🔑 Phase 4: GitHub Repository & Secrets

### 4.1 — Push Your Code to GitHub

```bash
# On your LOCAL machine, in your project folder:
git init
git add .
git commit -m "Initial commit"
git branch -M main

# Create a new repo on github.com first, then:
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git
git push -u origin main
```

### 4.2 — Add GitHub Secrets

These secrets are used by GitHub Actions to deploy — they are encrypted and never visible in logs.

1. Go to your GitHub repo
2. Click **Settings** tab (top menu)
3. Left sidebar: **Secrets and variables** → **Actions**
4. Click **New repository secret** for each one below:

| Secret Name | Value to Put |
|-------------|-------------|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token (see below) |
| `EC2_HOST` | Your EC2 public IP (e.g., `65.0.123.45`) |
| `EC2_USER` | `ubuntu` |
| `EC2_SSH_KEY` | Contents of your .pem file (see below) |
| `AWS_ACCESS_KEY_ID` | From IAM step |
| `AWS_SECRET_ACCESS_KEY` | From IAM step |
| `AWS_REGION` | `ap-south-1` |
| `AWS_S3_BUCKET_NAME` | Your bucket name |
| `POSTGRES_USER` | `myuser` |
| `POSTGRES_PASSWORD` | `YourStrongPassword123!` |
| `POSTGRES_DB` | `mydb` |
| `NEXTAUTH_SECRET` | Your random secret |
| `NEXTAUTH_URL` | `http://YOUR_EC2_IP:3000` |

**How to get Docker Hub access token:**
1. Log in to Docker Hub
2. Click your profile (top right) → **Account settings**
3. Click **Security** → **New Access Token**
4. Name: `github-actions` → **Generate**
5. Copy the token (you won't see it again!)

**How to get EC2_SSH_KEY:**
```bash
# Run this on your LOCAL machine:
cat ~/Downloads/my-server-key.pem
```
Copy ALL the output including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` and paste it as the secret value.

---

## ⚙️ Phase 5: GitHub Actions CI/CD Pipeline

### 5.1 — Create the Workflow File

In your project, create this exact folder structure:

```
my-app/
└── .github/
    └── workflows/
        └── deploy.yml    ← create this file
```

```yaml
# .github/workflows/deploy.yml

name: Build, Push & Deploy

# Trigger: runs when you push to the main branch
on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    name: Build Docker Image & Deploy to EC2
    runs-on: ubuntu-latest   # GitHub provides this free Linux machine

    steps:
      # ── Step 1: Get your code ─────────────────────────────────────
      - name: Checkout code
        uses: actions/checkout@v4

      # ── Step 2: Log in to Docker Hub ──────────────────────────────
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # ── Step 3: Set up Docker Buildx (needed for multi-platform) ──
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # ── Step 4: Build & push Docker image ─────────────────────────
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/my-nextjs-app:latest
          # Pass build args (for Next.js build time variables)
          build-args: |
            AWS_ACCESS_KEY_ID=${{ secrets.AWS_ACCESS_KEY_ID }}
            AWS_SECRET_ACCESS_KEY=${{ secrets.AWS_SECRET_ACCESS_KEY }}
            AWS_REGION=${{ secrets.AWS_REGION }}
            AWS_S3_BUCKET_NAME=${{ secrets.AWS_S3_BUCKET_NAME }}
            NEXTAUTH_SECRET=${{ secrets.NEXTAUTH_SECRET }}
            NEXTAUTH_URL=${{ secrets.NEXTAUTH_URL }}
          cache-from: type=gha    # Use GitHub Actions cache (faster rebuilds)
          cache-to: type=gha,mode=max

      # ── Step 5: SSH into EC2 and deploy ───────────────────────────
      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd /home/ubuntu/app

            # Write the .env file fresh on every deploy
            cat > .env << 'EOF'
            POSTGRES_USER=${{ secrets.POSTGRES_USER }}
            POSTGRES_PASSWORD=${{ secrets.POSTGRES_PASSWORD }}
            POSTGRES_DB=${{ secrets.POSTGRES_DB }}
            AWS_ACCESS_KEY_ID=${{ secrets.AWS_ACCESS_KEY_ID }}
            AWS_SECRET_ACCESS_KEY=${{ secrets.AWS_SECRET_ACCESS_KEY }}
            AWS_REGION=${{ secrets.AWS_REGION }}
            AWS_S3_BUCKET_NAME=${{ secrets.AWS_S3_BUCKET_NAME }}
            NEXTAUTH_SECRET=${{ secrets.NEXTAUTH_SECRET }}
            NEXTAUTH_URL=${{ secrets.NEXTAUTH_URL }}
            NODE_ENV=production
            EOF

            # Pull the newest image from Docker Hub
            docker pull ${{ secrets.DOCKERHUB_USERNAME }}/my-nextjs-app:latest

            # Stop and remove old containers (ignore error if not running)
            docker compose -f docker-compose.prod.yml down || true

            # Start fresh with new image
            docker compose -f docker-compose.prod.yml up -d

            # Remove old unused Docker images (saves disk space)
            docker image prune -f
```

### 5.2 — Update docker-compose.prod.yml Image Name

Open `docker-compose.prod.yml` and replace the image line:

```yaml
# REPLACE this line:
image: YOUR_DOCKERHUB_USERNAME/my-nextjs-app:latest

# WITH your actual Docker Hub username, example:
image: john123/my-nextjs-app:latest
```

### 5.3 — Commit and Push the Workflow

```bash
git add .
git commit -m "Add Docker + CI/CD pipeline"
git push origin main
```

---

## 🚀 Phase 6: Watch Your First Deployment

### 6.1 — Monitor GitHub Actions

1. Go to your GitHub repo
2. Click the **Actions** tab
3. You'll see a workflow running called **"Build, Push & Deploy"**
4. Click on it to see live logs
5. Wait ~5–8 minutes for first deployment (later deploys are faster due to cache)
6. All steps should show ✅ green checkmarks

### 6.2 — Test Your Deployed App

Open your browser:
```
http://YOUR_EC2_IP:3000
```

You should see your Next.js app running live! 🎉

### 6.3 — Verify Everything is Running on EC2

SSH into EC2 and check:

```bash
ssh -i ~/Downloads/my-server-key.pem ubuntu@YOUR_EC2_IP

# Check running containers
docker ps

# You should see TWO containers:
# - nextjs_app (port 3000)
# - prod_postgres

# Check app logs
docker logs nextjs_app

# Check database logs
docker logs prod_postgres
```

---

## 🔁 Phase 7: How CI/CD Works Going Forward

After setup, every time you update your code:

```bash
# Make your code changes in VS Code
# Then:
git add .
git commit -m "Added new feature"
git push origin main

# ← GitHub Actions AUTOMATICALLY:
#   1. Builds new Docker image
#   2. Pushes to Docker Hub
#   3. SSHes into EC2
#   4. Pulls new image
#   5. Restarts containers
# ← Your live app updates in ~5 minutes!
```

---

## 🗄️ Phase 8: Database Management

### 8.1 — Run Migrations on EC2

After your first deploy, you need to create your database tables:

```bash
# SSH into EC2
ssh -i ~/Downloads/my-server-key.pem ubuntu@YOUR_EC2_IP

# Run migration command inside the app container
# If using Prisma:
docker exec nextjs_app npx prisma migrate deploy

# If using plain SQL:
docker exec nextjs_app node scripts/migrate.js

# If using drizzle:
docker exec nextjs_app npx drizzle-kit push
```

### 8.2 — Connect pgAdmin to Production Database (Optional)

You can connect your local pgAdmin to the production database via SSH tunnel:

1. In pgAdmin → right-click **Servers** → **Register** → **Server**
2. **General tab**: Name: `EC2 Production`
3. **Connection tab**:
   - Host: `localhost`
   - Port: `5432`
   - Username: `myuser`
   - Password: `YourStrongPassword123!`
4. **SSH Tunnel tab**:
   - Toggle **Use SSH tunneling** ON
   - Host: `YOUR_EC2_IP`
   - Username: `ubuntu`
   - Authentication: **Identity file**
   - Identity file: browse to your `.pem` file
5. Click **Save**

---

## 🔒 Phase 9: Security Checklist Before Going Live

- [ ] `.env` file is in `.gitignore` and NOT committed to GitHub
- [ ] EC2 SSH is restricted to your IP in Security Group
- [ ] S3 bucket doesn't expose sensitive files publicly
- [ ] Docker containers don't expose PostgreSQL port (5432) to the internet
- [ ] Strong PostgreSQL password (mix of letters, numbers, symbols)
- [ ] `NEXTAUTH_SECRET` is a random 64+ character string

---

## 🛠️ Common Errors & Fixes

### Error: `Cannot connect to database`
```bash
# Check if postgres container is running
docker ps

# Check postgres logs
docker logs prod_postgres

# Make sure DATABASE_URL matches the postgres container name in compose
# It should be: postgresql://user:pass@postgres:5432/dbname
#                                          ^^^^^^^^ this must match service name in compose
```

### Error: `docker: Permission denied`
```bash
# Add ubuntu user to docker group
sudo usermod -aG docker ubuntu
# Then EXIT and SSH back in
```

### Error: `Port 3000 already in use`
```bash
# Stop all containers
docker compose -f docker-compose.prod.yml down
# Then start again
docker compose -f docker-compose.prod.yml up -d
```

### Error: `GitHub Actions - SSH failed`
- Check that `EC2_SSH_KEY` secret contains the FULL .pem file including headers
- Check that `EC2_HOST` is just the IP address (no `http://` or trailing slash)
- Check EC2 Security Group allows SSH (port 22) from `0.0.0.0/0` or GitHub's IP range

### Error: `image not found` on EC2
- Check Docker Hub username is correct in `docker-compose.prod.yml`
- Make sure Docker Hub repo is **Public**
- Re-check `DOCKERHUB_USERNAME` secret spelling

### Error: `Module not found` or app crashes on start
```bash
# Check app logs for exact error
docker logs nextjs_app --tail 100
```

---

## 📁 Final File Summary

Here's every file you need to create or modify:

```
my-app/
├── .github/
│   └── workflows/
│       └── deploy.yml          ← NEW: GitHub Actions pipeline
├── .dockerignore               ← NEW: Tells Docker what to ignore
├── .env.example                ← NEW: Template for env vars (safe to commit)
├── docker-compose.yml          ← NEW: For local testing only
├── docker-compose.prod.yml     ← NEW: Runs on EC2
├── Dockerfile                  ← NEW: How to build the Docker image
├── next.config.js              ← MODIFIED: Added output: 'standalone'
└── ... rest of your Next.js files
```

---

## 🎯 Quick Reference Commands

```bash
# LOCAL: Start local dev with Docker
docker compose up --build

# LOCAL: Stop local Docker
docker compose down

# EC2: Check running containers
docker ps

# EC2: View app logs (last 50 lines)
docker logs nextjs_app --tail 50

# EC2: Restart app only (not database)
docker compose -f docker-compose.prod.yml restart app

# EC2: Enter app container shell
docker exec -it nextjs_app sh

# EC2: Enter postgres container shell
docker exec -it prod_postgres psql -U myuser -d mydb
```

---

*Guide prepared for: Next.js (API routes only) + PostgreSQL + AWS S3 + Docker + AWS EC2 + GitHub Actions CI/CD*
*Difficulty level: Fresher-friendly, every step explained*
