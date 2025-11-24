#  Production-Grade AWS VPC Implementation  
A Complete Step-by-Step Guide (Public + Private Subnets, NAT, ALB, ASG, Bastion Host)

This repository documents the implementation of a **production-grade, multi-AZ AWS VPC architecture** built using the **VPC and More** creation option.  
It covers VPC design, subnet layout, NAT gateways, load balancing, auto scaling, secure access via Bastion, and deployment of a sample application.  
The goal is to help beginners move beyond the default VPC and learn how real-world cloud infrastructure is designed.  

---

## 📘 1. Why This Architecture?

Modern production environments need:

- **High Availability** across multiple AZs  
- **Security** through private subnets  
- **Controlled outbound access** for servers  
- **Scalability** via Auto Scaling  
- **Load Balancing** for consistent traffic handling  


This architecture achieves all these using:

- VPC  
- Public & Private Subnets  
- NAT Gateways  
- Application Load Balancer (ALB)  
- Auto Scaling Group (ASG)  
- Bastion Host
- S3 Gateway is not used in current implementation

  <img width="827" height="691" alt="image" src="https://github.com/user-attachments/assets/00e3d92f-0a73-4e83-aef9-b2842231b137" />


### 🔐 Public vs Private Subnets  
- **Public Subnets:** NAT Gateways, Load Balancers, Bastion Hosts  
- **Private Subnets:** Your application servers (hidden from internet)  

### 🌍 High Availability  
Resources are replicated across **Two Availability Zones**, ensuring fault tolerance.  

---

## 📦 2. Core Components Explained

### **VPC (Virtual Private Cloud)**  
Your logically isolated network.  

### **Auto Scaling Group (ASG)**  
Automatically scales EC2 instances based on load.  

### **Application Load Balancer (ALB)**  
Distributes traffic across private EC2 instances.  
It lives in the *public* subnet.  

### **Bastion Host**  
A jump-box for SSH access into private EC2 instances.  

---

## 🧩 3. Architecture Diagram  
The architecture follows this structure:

- Multi-AZ  
- Public + Private Subnets  
- ALB in public  
- EC2 (via ASG) in private  
- NAT Gateways for outbound internet  


---

## 🔄 4. How the Architecture Works

1. A VPC is created across 2 AZs  
2. Each AZ contains **1 Public** + **1 Private Subnet**  
3. NAT Gateways are deployed in public subnets  
4. ALB sits in public subnets to receive traffic  
5. ASG launches EC2s inside private subnets only  
6. Traffic flow:  
   **User → ALB → Private EC2s**  
7. Private EC2s use NAT Gateway for software updates  

---

# 🛠️ 5. Step-by-Step Implementation

## ✔️ Step 1 — VPC & Networking Setup  

### 🔧 Actions
- Go to **VPC Dashboard → Create VPC**
- Choose **VPC and More** option
- Select:
  - **2 Availability Zones**
  - **2 Public Subnets**
  - **2 Private Subnets**
  - **NAT Gateway = 1 per AZ**

 <img width="1757" height="960" alt="image" src="https://github.com/user-attachments/assets/4c9ccde7-1bee-4c90-8b23-3c5cd4d84575" />

### 💡 Why NAT Gateways?  
Private servers need outward internet access (e.g., `apt-get update`) **without exposing themselves publicly**.

### ⚠️ Cost Note  
NAT Gateways incur hourly + data processing charges.  
Remember to clean up after labs.  

---

## ✔️ Step 2 — Auto Scaling Group (ASG)  

### 1. Network Settings  
Leave defaults as provided.

### 2. Create a Security Group  
- Name: `aws-prod-example`  
- Inbound Rules:  
  - SSH → 0.0.0.0/0  
  - TCP 8000 → 0.0.0.0/0  

### 3. ASG Configuration  
- Select Launch Template  
- Choose the created VPC  
- Select **Private Subnets only**  
- Desired Capacity: **2 EC2 instances**

### 4. Load Balancer  
Skip here — will create separately.

<img width="1504" height="569" alt="image" src="https://github.com/user-attachments/assets/c2311329-4197-4664-a0de-9d1df0d410a7" />

### 5. Verification  
Check EC2 console for two running EC2 instances. 

<img width="1695" height="500" alt="image" src="https://github.com/user-attachments/assets/b2b06d65-fd86-4194-983d-76d1ab0e9d0c" />

---

## ✔️ Step 3 — Bastion Host (Jump Box)  

Private EC2s have no public IP—so how do we SSH into them?

### Steps:
1. Launch a small EC2 in **public subnet**

<img width="1705" height="969" alt="image" src="https://github.com/user-attachments/assets/febd20fb-3fc7-449e-9081-5a4e8c532723" />
<img width="1743" height="495" alt="image" src="https://github.com/user-attachments/assets/f32ca9e7-f14d-432c-9701-f1b0bbbb65ce" />



3. SCP the key pair to the bastion:

`scp -i key.pem key.pem ubuntu@<bastion-ip>:/home/ubuntu/`

<img width="1887" height="292" alt="image" src="https://github.com/user-attachments/assets/295cf948-715e-47ca-8ed9-3f83662ac9ad" />


4. SSH into the bastion:  


`ssh -i key.pem ubuntu@<bastion-ip>`

4. From bastion → private EC2:  


`ssh -i key.pem ubuntu@<private-ip>`

---

## ✔️ Step 4 — Deploy the Test Application  

SSH into each private instance via the bastion, then:

### **On EC2 Instance A**


`echo "First project to demonstrate application in private subnet" > index.html`

`python3 -m http.server 8000`


### **On EC2 Instance B**


`echo "Second project to demonstrate application in private subnet" > index.html`

`python3 -m http.server 8000`


Now you have two servers responding differently.


Now you have two servers responding differently.

---

## ✔️ Step 5 — Application Load Balancer  

### Steps:
1. Create **Application Load Balancer**
   - Type: Internet Facing  
   - Subnets: Public subnets of both AZs
    
2. Create Target Group
   - Register the two private EC2 instances  

3. Access using ALB DNS URL

<img width="1807" height="848" alt="image" src="https://github.com/user-attachments/assets/ae8cedd3-053e-408a-b99b-ded2bd21e903" />

### 💡 Load Balancing Test  
Refresh the page multiple times →  
You should see alternating responses from Server A ↔ Server B.

### ⚠️ If ALB Fails  
Add inbound rule to ALB SG:
  
Allow HTTP (80) from 0.0.0.0/0 


---

# 🧹 6. Cleanup Checklist  
To avoid accidental AWS charges, delete resources in this order:  

1. Auto Scaling Group  
2. Target Group  
3. Load Balancer  
4. NAT Gateways  
5. Bastion Host  
6. EC2 Instances  
7. VPC  

---

# 🎉 Final Summary  
By completing this project, you have built:

- Multi-AZ production-ready VPC  
- Public + Private Subnet Architecture  
- Private EC2 Instances  
- NAT Gateway for secure outbound access  
- Application Load Balancer for traffic distribution  
- Auto Scaling Group for high availability  
- Bastion Host for secure SSH access  

This forms the foundation of most real-world AWS enterprise deployments.

---

# 🙌 Author  
**Narasimha Potturi**  
DevOps & Cloud Engineer  
