# Task 1. Web Application Deployment with EC2 Auto Scaling and Load Balancer

## Overview

This project demonstrates the deployment of a Dockerized web application using AWS EC2 Auto Scaling Group and an Application Load Balancer (ALB). It includes automatic scaling based on traffic load and verification through load testing using Locust.

The application used is [`nodejs-demoapp`](https://github.com/benc-uk/nodejs-demoapp), which is available as a pre-built Docker image: `bencuk/nodejs-demoapp`.

---

## Architecture

- **EC2 Auto Scaling Group (ASG)**: Automatically adjusts the number of EC2 instances based on traffic demand.
- **Launch Template**: Used to install Docker and run the application container on EC2 instance startup.
- **Application Load Balancer (ALB)**: Routes incoming HTTP traffic to healthy EC2 instances.
- **CloudWatch**: Monitors CPU utilization to trigger scale in/out events.
- **Locust**: Load testing tool used to simulate traffic and observe scaling behavior.

---

## 1. EC2 Auto Scaling Setup

### ✅ Prerequisites
- AWS Account with admin privileges
- EC2 Key Pair (optional for SSH access)
- Default VPC and public subnets in a selected region (e.g., `eu-north-1`)

### 🚀 Launch Template 

Created a launch template used by the Auto Scaling Group.

- **AMI**: Ubuntu 24.04
- **Instance Type**: t3.micro 
- **Network Settings**: Used default VPC/subnet settings  
- **User Data Script**:

```bash
#!/bin/bash
set -e

# Update system and install required packages
apt-get update
apt-get install -y ca-certificates curl

# Create keyring directory for Docker
install -m 0755 -d /etc/apt/keyrings

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker apt repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable and start Docker
systemctl enable docker
systemctl start docker

# Run the demoapp container (port 80 on host mapped to 3000 in container)
docker run -d -p 80:3000 --name demoapp bencuk/nodejs-demoapp
```
---



## 2. 🧱 Auto Scaling Group Settings
- **Minimum Capacity**: 1  
- **Maximum Capacity**: 3
- **Desired Capacity**: 1  
- **VPC**: Used default with public subnets  
- **Availability Zones**: Selected at least two for high availability  
- **Public IP**: Enabled (for accessing the instance via ALB)  
- **Health Checks**: EC2 + ELB  
- **Health Check Grace Period**: 150 seconds  
- **Instance Warmup**: 100 seconds  
- **Scaling Policy**:  
  - **Type**: Target Tracking Scaling  
  - **Metric Type**: Average CPU Utilization  
  - **Target Value**: 60%  
  - **Cooldown Period**: 300 seconds  
- **Instance Type**: `t3.micro`  
- **Launch Template**: Contains the User Data script for Docker installation and app deployment  
- **Monitoring**: Basic (CloudWatch group metrics optional)  
- **Instance Protection from Scale-In**: Disabled (allow automatic scale-in)  
- **Termination Policy**: Default (oldest launch configuration first)

---

## 3. Application Load Balancer Setup

- **Type**: Application Load Balancer (ALB)  
- **Scheme**: Internet-facing  
- **IP address type**: IPv4  
- **Listeners**: HTTP on port 80  
- **Availability Zones**: Selected the same as in the Auto Scaling Group  
- **Target Group**:  
  - **Name**: Week2Task1ALBTG  
  - **Target type**: Instance  
  - **Protocol**: HTTP  
  - **Port**: 80  
  - **Health checks**:  
    - **Protocol**: HTTP  
    - **Path**: `/`  
    - **Healthy threshold**: 2  
    - **Unhealthy threshold**: 2  
    - **Timeout**: 5 seconds  
    - **Interval**: 30 seconds  
- **Register targets**: Leaved empty — the Auto Scaling Group will automatically register instances with this target group.

---
## 4. Load Testing with Locust

To simulate high traffic and observe the Auto Scaling Group’s behavior, used **Locust** — a modern load testing tool that allows you to define user behavior in Python.

My settings were:

![image](https://github.com/user-attachments/assets/d2e334d1-6fcf-4e33-a016-b262884d8f46)

---

## 5. Report

In the screenshots below, you can see my configuration and the tests I performed.

Demo app is accessible ✅:

![image](https://github.com/user-attachments/assets/968b28ec-b1e3-4a79-a3d1-b09b1c9404ab)


When the CPU utilization reached 60%, the dynamic scaling policy was triggered.

![image](https://github.com/user-attachments/assets/3a482ac2-ffde-45e9-aec6-9f3e6f5d3a2f)

![image](https://github.com/user-attachments/assets/fe7cb8cd-2127-4ae5-b0f0-fba6b927a3f6)

![image](https://github.com/user-attachments/assets/07ce1b96-f5dd-4c86-bc0f-afdcfdfef02a)

![image](https://github.com/user-attachments/assets/0d72a5ba-9475-4a6a-93eb-9c3cafd1988d)

---

![image](https://github.com/user-attachments/assets/3cfa5714-2b9f-40a5-b415-e31c4699df11)

![image](https://github.com/user-attachments/assets/e0dc451b-c1b6-400d-890b-8f4802767637)

![image](https://github.com/user-attachments/assets/1024ab23-8626-47fa-8ab6-43c40a825b7b)


## 📝 Task 4: Deployment Method Comparison Report

In this section, we summarize and compare the two deployment methods in this project — **EC2 Auto Scaling Group** and **ECS Fargate** — in terms of their **ease of setup**, **cost**, and **scalability**. This comparison helps determine the best option for running Dockerized applications with dynamic traffic requirements.

---

### ⚙️ 1. Ease of Setup

| Criteria            | EC2 Auto Scaling Group                         | ECS Fargate                                         |
|---------------------|------------------------------------------------|-----------------------------------------------------|
| Docker Setup        | Requires manual installation via user-data    | No manual Docker setup required                    |
| Instance Management | Needs AMI or Launch Template with User Data   | Fully managed compute — no instance management     |
| Complexity          | Higher — setup involves EC2, ALB, ASG         | Lower — abstracted infrastructure via ECS console  |
| Learning Curve      | Steeper for beginners                         | Easier for container-first workflows               |

---

### 💸 2. Cost

| Criteria               | EC2 Auto Scaling Group                    | ECS Fargate                                       |
|------------------------|-------------------------------------------|---------------------------------------------------|
| Pay Model              | Pay per instance hour                     | Pay per vCPU and memory usage per second         |
| Idle Cost              | Instances remain running unless scaled    | No idle cost when services are idle              |
| Cost Optimization      | Requires fine-tuned scaling policies      | Automatically optimized at task level            |
| Minimum Cost Footprint | Higher                                    | Lower — especially at low usage                  |

---

### 📈 3. Scalability

| Criteria                  | EC2 Auto Scaling Group                          | ECS Fargate                                            |
|---------------------------|--------------------------------------------------|--------------------------------------------------------|
| Horizontal Scaling        | Configurable with CloudWatch metrics            | Configurable via ECS Service Auto Scaling              |
| Scale-out Speed           | Slower (due to EC2 provisioning time)           | Faster — new tasks launch in seconds                  |
| Load Balancer Integration | Elastic Load Balancer required                  | Integrated with ECS — ALB support out of the box      |
| Cold Start Time           | Can take up to several minutes                  | Generally <1 minute for Fargate tasks                 |

---

### ✅ Conclusion

| Category     | Winner        | Reason                                                                 |
|--------------|----------------|------------------------------------------------------------------------|
| Ease of Use  | **ECS Fargate** | Simplified deployment, no infrastructure to manage                    |
| Cost         | **ECS Fargate** | Lower cost at low-to-medium workloads, pay-per-use model              |
| Scalability  | **ECS Fargate** | Faster cold starts, integrated scaling, ideal for burst traffic       |
| Flexibility  | **EC2 ASG**     | More customizable environment, full control over base OS and runtime |

**Recommendation**:  
For fast-growing startups and microservices that prioritize **speed of deployment**, **cost-effectiveness**, and **automatic scaling**, **ECS Fargate** is the preferred option.  
However, **EC2 Auto Scaling Groups** are better suited for scenarios requiring **more control**, **custom dependencies**, or **persistent compute environments**.

---

