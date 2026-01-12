# Project-4-Private-Service-Exposure-using-VPC-Endpoint-PrivateLink-with-Network-Load-Balancer
This project demonstrate the use of VPC endpoint exposing private service with NLB without having public IP, Internet, NAT gatewat. All traffic stays inside AWS network.
# 🔐 Private Service Exposure Using VPC Endpoint (PrivateLink) with Network Load Balancer

This project demonstrates how to expose a private service running in one VPC to another VPC using **AWS PrivateLink** and a **Network Load Balancer (NLB)** without using the public internet, Internet Gateway, or NAT Gateway at runtime.

The entire communication happens over the AWS private backbone, providing a highly secure, scalable, and enterprise-grade architecture.

---

## 🏗 Architecture Overview

```
Consumer VPC                          Provider VPC
┌───────────────┐                    ┌────────────────┐
│  EC2 Client   │                    │   EC2 Backend  │
│ Private Subnet│                    │(Web App)Pvt Sub│
│      |        │                    │        |       │
│      |        │                    │   Target Group │
│      ▼        │                    │        |       │
│ Interface VPC Endpoint ─────────▶  │Network LB     │
│  (PrivateLink)                     │ (Internal)     │
└───────────────┘                    └────────────────┘
```
Flow:
Consumer EC2 → Interface VPC Endpoint → AWS PrivateLink → Internal NLB → Backend EC2
The architecture consists of two VPCs: a Provider VPC that hosts a private application behind an internal Network Load Balancer, and a Consumer VPC that accesses this application through an Interface VPC Endpoint using AWS PrivateLink.
All communication happens over the AWS private backbone, with no dependency on Internet Gateway or NAT Gateway during runtime.
A NAT Gateway is used only during the provisioning phase to allow the backend EC2 instance to install required software packages.

No Internet Gateway  
No NAT Gateway (for runtime)  
No Public IPs  

---

## 🧩 Components Used

### Provider Side (Service Provider)
- VPC (e.g., `10.0.0.0/24`)
- Private Subnet (e.g., `10.0.0.128/25`)
- Public Subnet (e.g., `10.0.0.0/25`), Internet Gateway, Route Table, NAT Gateway (Temporary)
- Backend EC2 instance (Apache Web Server)
- Target Group (TCP:80)
- Internal Network Load Balancer (NLB)
- VPC Endpoint Service (PrivateLink Service)
- IAM Role for SSM:  
  - `AmazonSSMManagedInstanceCore`

### Consumer Side (Service Consumer)
- VPC (e.g., `192.168.0.0/24`)
- Private Subnet (e.g., `192.168.0.0/25`)
- EC2 Client instance
- Interface VPC Endpoint pointing to Provider Endpoint Service
- IAM Role for SSM:
  - `AmazonSSMManagedInstanceCore`

### Systems Manager (SSM)
Used to connect to private EC2 instances without SSH or public IP.

Interface endpoints required:
```
com.amazonaws.<region>.ssm
com.amazonaws.<region>.ec2messages
com.amazonaws.<region>.ssmmessages
```
### 🌐 Temporary Use of NAT Gateway (Bootstrapping Only)

During the initial setup of the backend EC2 instance in the Provider VPC, a **NAT Gateway** was used temporarily to allow outbound internet access for installing required packages (like Apache HTTP Server) via user data.

This was required because:
- The backend EC2 runs in a private subnet
- Private subnets have no direct internet access
- `yum install httpd` requires outbound connectivity to Amazon Linux repositories

Architecture during bootstrap phase:
```
Backend EC2 (Private Subnet)
|
v
NAT Gateway (Public Subnet)
|
v
Internet (for package installation only)
```

---

### ⚙ Backend EC2 User Data Script

```
#!/bin/bash
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "Hello from Provider VPC Backend EC2" > /var/www/html/index.html
```
---
Once the backend server was configured and the web service was running:
- The NAT Gateway was no longer required for application traffic
- All service communication used AWS PrivateLink
- The architecture returned to a fully private design

Runtime traffic flow:
```
Consumer EC2
→ Interface Endpoint
→ PrivateLink
→ NLB
→ Backend EC2
```

## 🔐Security Group Design
### 1. Backend EC2 Security Group (Provider)
Since NLB preserves the source IP, backend must allow traffic from the NLB subnet CIDR:
```
Inbound:
TCP 80 → <NLB Subnet CIDR>   (example: 10.0.0.0/24 or 10.0.0.128/25)
Outbound:
All traffic
```
### 2. NLB Security Group (Provider)
```
Inbound:
TCP 80 → Consumer VPC CIDR  (or tightened further using prefix lists)
Outbound:
All traffic
```
### 3. Interface Endpoint Security Group (Consumer)
```
Inbound:
TCP 80 → consumer-ec2-sg
Outbound:
All traffic
```

### 4. Consumer EC2 Security Group
```
Outbound:
All traffic
Inbound:
Not required (SSM used for access)
```

## 🧪 Validation Steps
### Verify backend locally:
```
curl localhost
```

### Verify NLB from another EC2 in Provider VPC:
```
curl http://<NLB-DNS-NAME>
```

### Verify PrivateLink from Consumer EC2:
```
curl http://<INTERFACE-ENDPOINT-DNS>
```
Expected Result :
```
Hello from Provider VPC Backend EC2
```

---
## ✅ Result
- Successfully accessed a private service across VPCs.
- No public IPs were used.
- No Internet Gateway or NAT Gateway required for runtime.
- All traffic stayed inside AWS private backbone.
- Fully functional PrivateLink architecture implemented.

## 🔒 Security Considerations
- No internet exposure.
- No SSH access; SSM used instead.
- Service is accessible only via Interface Endpoint.
- Backend EC2 only accepts traffic from NLB subnet CIDR.
- Endpoint access limited to authorized consumer EC2 instances.
- Zero-trust, least-privilege networking model.

## 🎯 Why This Project Is Important
This architecture is used by:
- SaaS providers
- Banking and fintech platforms
- Microservice communication across VPCs
- Enterprise service sharing models
It demonstrates real-world AWS networking design and security best practices.

## 📚 Learning Outcomes
- Understanding of AWS PrivateLink
- Difference between ALB and NLB use cases
- Why NLB preserves source IP
- How VPC Interface Endpoints work
- Why NAT/Internet is not required for PrivateLink
- How SSM replaces SSH securely
- How to design zero-internet architectures

## 🚀 Conclusion
This project proves that fully private, secure, and scalable service-to-service communication is possible in AWS using:
- Network Load Balancer
- VPC Endpoint Service
- Interface VPC Endpoints
- Systems Manager
It reflects enterprise-grade cloud architecture and strong understanding of AWS networking fundamentals
