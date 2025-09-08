# Automating K3s Cluster and Full-Stack Application Deployment

## Step 01: Configure AWS CLI
The AWS CLI is a command-line tool that allows you to interact with AWS services programmatically. It simplifies provisioning resources, such as EC2 instances and load balancers, which are required to host our database cluster and application server. Let's configure the AWS CLI:

```bash
aws configure
```

- **AWS Access Key ID**: Your access key to authenticate AWS API requests.  
- **AWS Secret Access Key**: A secret key associated with your access key.  
- **Default region**: The AWS region in which you want to provision your resources (e.g., `ap-southeast-1`).  
- **Default output format**: Choose JSON, text, or table.  

---

## Step 02: Provisioning Compute Resources

### 1. Create a Directory for Your Infrastructure
```bash
mkdir k3s-cluster-infra
cd k3s-cluster-infra
```

### 2. Install Python venv
Set up a Python virtual environment (venv) to manage dependencies for Pulumi or other Python-based tools:

```bash
sudo apt update
sudo apt install python3.8-venv -y
```

### 3. Initialize a New Pulumi Project
Login to Pulumi:
```bash
pulumi login
```

Initialize the project:
```bash
pulumi new aws-python
```

Go with the default options. Change the AWS region to `ap-southeast-1`.

---

### 4. Update the `__main__.py` File

Below is the Pulumi program that provisions AWS resources and sets up a K3s cluster:

```python
# (code omitted for brevity – same as provided above)
```

---

### 5. Create an AWS Key Pair
```bash
cd ~/.ssh/
aws ec2 create-key-pair --key-name k3s-cluster --output text --query 'KeyMaterial' > k3s-cluster.id_rsa
chmod 400 k3s-cluster.id_rsa
```

---

### 6. Provision the Infrastructure
```bash
pulumi up --yes
```

---

### 7. SSH into the EC2 Instances
```bash
ssh master
ssh worker-1
ssh worker-2
```

---

## K3S Cluster Verification
On the master node:
```bash
kubectl get nodes
```

---

## Full Stack Application Deployment

### Step 1: Cloning the Repository
```bash
git clone https://github.com/poridhioss/Full-Stack-Application.git
```

### Project Structure
```plaintext
Full-Stack-Application/
├── frontend/                    # React TypeScript app
├── backend/                     # Node.js Express API
├── DB/                          # PostgreSQL schema + seed
├── k8s/                         # Kubernetes manifests
```

---

### Step 2: Backend Setup
```bash
cd backend
npm install
```

Containerize and push with Docker + Makefile (`USERNAME` must be updated to your DockerHub username):
```bash
docker login
make build
make push
```

---

### Step 3: Frontend Setup
```bash
cd frontend
npm install
```

Same process as backend:  
```bash
docker login
make build
make push
```

---

### Step 4: Database Setup
Schema in `DB/init/01_init.sql`. Example users table, indexes, and trigger.

---

### Step 5: Deploy with Kubernetes
Copy manifests:
```bash
scp -r k8s/ master:~/
```

Apply in order:
```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/postgres-init.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
kubectl apply -f k8s/ingress.yaml
```

---

### Step 6: Verification & Monitoring
```bash
kubectl get pods -n user-management
kubectl get all -n user-management
kubectl logs -f deployment/backend -n user-management
```

---

### Access the App
```plaintext
http://MASTER_PUBLIC_IP
```

Login, Register, Dashboard available.

---

## Conclusion
This guide walks through provisioning a K3s cluster with Pulumi and deploying a full-stack user management app with Kubernetes. It highlights containerization, orchestration, and modern IaC practices.
