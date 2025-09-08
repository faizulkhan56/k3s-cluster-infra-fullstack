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
import os
import pulumi
import pulumi_aws as aws

# Configuration
config = pulumi.Config()
vpc_cidr = config.get("vpc_cidr") or "10.0.0.0/16"
public_subnet_cidr = config.get("public_subnet_cidr") or "10.0.1.0/24"
availability_zone = config.get("availability_zone") or "ap-southeast-1a"
ubuntu_ami_id = config.get("ami_id") or "ami-060e277c0d4cce553"  # Ubuntu 24.04 LTS
k3s_token = config.get("k3s_token") or "super-secret-token" # Update the token
access_key = config.get("k3sCluster") or "k3s-cluster"

# Create a VPC
vpc = aws.ec2.Vpc("my-vpc",
    cidr_block=vpc_cidr,
    tags={
        "Name": "my-vpc"
    }
)

# Create a public subnet
public_subnet = aws.ec2.Subnet("public-subnet",
    vpc_id=vpc.id,
    cidr_block=public_subnet_cidr,
    availability_zone=availability_zone,
    map_public_ip_on_launch=True,
    tags={
        "Name": "public-subnet"
    }
)

# Create an Internet Gateway
igw = aws.ec2.InternetGateway("internet-gateway",
    vpc_id=vpc.id,
    tags={
        "Name": "igw"
    }
)

# Create a route table
public_route_table = aws.ec2.RouteTable("public-route-table",
    vpc_id=vpc.id,
    tags={
        "Name": "rt-public"
    }
)

# Create a route in the route table for the Internet Gateway
route = aws.ec2.Route("igw-route",
    route_table_id=public_route_table.id,
    destination_cidr_block="0.0.0.0/0",
    gateway_id=igw.id
)

# Associate the route table with the public subnet
route_table_association = aws.ec2.RouteTableAssociation("public-route-table-association",
    subnet_id=public_subnet.id,
    route_table_id=public_route_table.id
)

# Security Group for K3s Cluster Nodes
k3s_security_group = aws.ec2.SecurityGroup("k3s-secgrp",
    vpc_id=vpc.id,
    description="Security group for K3s cluster nodes",
    ingress=[
        # K3s API server
        {
            "protocol": "tcp",
            "from_port": 6443,
            "to_port": 6443,
            "cidr_blocks": [public_subnet.cidr_block]
        },
        # Flannel VXLAN
        {
            "protocol": "udp",
            "from_port": 8472,
            "to_port": 8472,
            "cidr_blocks": [public_subnet.cidr_block]
        },
        # SSH for administration
        {
            "protocol": "tcp",
            "from_port": 22,
            "to_port": 22,
            "cidr_blocks": ["0.0.0.0/0"]
        },
        # HTTP/HTTPS for applications
        {
            "protocol": "tcp",
            "from_port": 80,
            "to_port": 80,
            "cidr_blocks": ["0.0.0.0/0"]
        },
        {
            "protocol": "tcp",
            "from_port": 443,
            "to_port": 443,
            "cidr_blocks": [public_subnet.cidr_block]
        },
        # Allow all internal cluster communication
        {
            "protocol": "-1",
            "from_port": 0,
            "to_port": 0,
            "cidr_blocks": [public_subnet.cidr_block]
        }
    ],
    egress=[
        # Allow all outbound traffic
        {
            "protocol": "-1",
            "from_port": 0,
            "to_port": 0,
            "cidr_blocks": ["0.0.0.0/0"]
        }
    ],
    tags={
        "Name": "k3s-cluster-sg"
    }
)

# Create K3s master instance
master_instance = aws.ec2.Instance("master-instance",
    instance_type="t3.small",
    vpc_security_group_ids=[k3s_security_group.id],
    ami=ubuntu_ami_id,
    subnet_id=public_subnet.id,
    key_name=access_key,
    associate_public_ip_address=True,
    user_data=f"""#!/bin/bash
    sudo apt update
    sudo hostnamectl set-hostname k3s-master
    # Install K3s server
    curl -sfL https://get.k3s.io | K3S_TOKEN="{k3s_token}" sh -s - server \\
    --cluster-init

    # Wait for K3s to be ready
    export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
    until /usr/local/bin/kubectl get nodes 2>/dev/null; do
        echo "Waiting for K3s to start..."
        sleep 5
    done

    chown ubuntu:ubuntu /etc/rancher/k3s/k3s.yaml
    chmod 600 /etc/rancher/k3s/k3s.yaml
    """,
    tags={
        "Name": "k3s-master"
    }
)

# Number of workers you want
num_workers = 2

# User data template (filled per worker)
def make_worker_userdata(ip, index):
    return f"""#!/bin/bash
    sudo apt update
    sudo hostnamectl set-hostname k3s-worker{index}
    # Install K3s agent
    curl -sfL https://get.k3s.io | K3S_TOKEN="{k3s_token}" K3S_URL=https://{ip}:6443 sh -s -
    """

# Create multiple worker instances
workers = []
for i in range(1, num_workers + 1):
    worker_user_data = master_instance.private_ip.apply(
        lambda ip, idx=i: make_worker_userdata(ip, idx)
    )

    worker = aws.ec2.Instance(f"worker{i}-instance",
        instance_type="t3.small",
        vpc_security_group_ids=[k3s_security_group.id],
        ami=ubuntu_ami_id,
        subnet_id=public_subnet.id,
        key_name=access_key,
        associate_public_ip_address=True,
        user_data=worker_user_data,
        tags={
            "Name": f"k3s-worker{i}"
        },
        opts=pulumi.ResourceOptions(
            depends_on=[master_instance]
        )
    )

    workers.append(worker)

# Export outputs
pulumi.export("vpc_id", vpc.id)
pulumi.export("public_subnet_id", public_subnet.id)
pulumi.export("igw_id", igw.id)
pulumi.export("public_route_table_id", public_route_table.id)

pulumi.export("master_instance_id", master_instance.id)
pulumi.export("master_instance_public_ip", master_instance.public_ip)
pulumi.export("master_private_ip", master_instance.private_ip)


for i, worker in enumerate(workers, start=1):
    pulumi.export(f"worker{i}_instance_id", worker.id)
    pulumi.export(f"worker{i}_instance_public_ip", worker.public_ip)
    pulumi.export(f"worker{i}_private_ip", worker.private_ip)


# Create config file
def create_config_file(all_ips):
    hosts = ["master"] + [f"worker-{i+1}" for i in range(len(workers))]
    config_lines = []

    for idx, host in enumerate(hosts):
        config_lines.append(f"Host {host}")
        config_lines.append(f"  HostName {all_ips[idx]}")
        config_lines.append(f"  User ubuntu")
        config_lines.append(f"  IdentityFile ~/.ssh/k3s-cluster.id_rsa\n")
        config_lines.append("")  # Blank line between hosts

    config_content = "\n".join(config_lines)
    
    config_path = os.path.expanduser("~/.ssh/config") # For ubuntu file system
    with open(config_path, "w") as config_file:
        config_file.write(config_content)

# Collect the IPs for all nodes
all_ips = [master_instance.public_ip] + [worker.public_ip for i, worker in enumerate(workers)]

# Create the config file with the IPs once the instances are ready
pulumi.Output.all(*all_ips).apply(create_config_file)

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
git clone https://github.com/faizulkhan56/Full-Stack-Application

```

### Project Structure
```plaintext
Full-Stack-Application/
├── frontend/                    # React TypeScript application
│   ├── public/                  # Static assets
│   ├── src/
│   │   ├── components/          # React components
│   │   │   ├── Login.tsx        # User login component
│   │   │   ├── Register.tsx     # User registration component
│   │   │   └── Dashboard.tsx    # Main dashboard with user management
│   │   ├── services/            # API services
│   │   │   ├── api.ts           # Axios configuration with interceptors
│   │   │   └── userService.ts   # User-related API calls
│   │   ├── App.tsx              # Main application component
│   │   └── index.tsx            # Application entry point
│   ├── Dockerfile               # Multi-stage Docker build
│   ├── nginx.conf               # Nginx configuration for production
│
├── backend/                     # Node.js Express API
│   ├── src/
│   │   ├── config/
│   │   │   └── database.js      # PostgreSQL connection configuration
│   │   ├── controllers/
│   │   │   └── userController.js # User business logic
│   │   ├── middleware/
│   │   │   └── authMiddleware.js # JWT authentication middleware
│   │   ├── models/
│   │   │   └── userModel.js     # User data access layer
│   │   ├── routes/
│   │   │   └── userRoutes.js    # API route definitions
│   │   └── server.js            # Express server setup
│   ├── Dockerfile               # Production-ready container image
│   ├── endpoint.http            # API testing endpoints
│
├── DB/                          # Database configuration
│   ├── init/
│   │   └── 01_init.sql          # Database schema and seed data
│
├── k8s/                         # Kubernetes manifests
│   ├── namespace.yaml           # Namespace and RBAC
│   ├── postgres.yaml            # PostgreSQL deployment
│   ├── postgres-init.yaml       # Database initialization ConfigMap
│   ├── backend.yaml             # Backend API deployment
│   ├── frontend.yaml            # Frontend deployment with NodePort
│   ├── ingress.yaml             # Traefik ingress configuration

```

---

### Step 2: Backend Setup
The backend is a Node.js/Express.js application that interacts with a PostgreSQL database. To prepare it for deployment, install the required dependencies by navigating to the backend/ directory and running:
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
Similar to the backend, the frontend includes a Dockerfile for containerization, optimized with a multi-stage build and Nginx for serving static assets. Follow the same steps as the backend:
```bash
cd frontend
npm install
```

Same process as backend:
The repository likely includes a Makefile or configuration file that references a Docker Hub username (e.g., USERNAME ?= your docker username). Before building the image, update this username to your own Docker Hub username. If using a Makefile, locate the line:
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
