# AWS VPC Peering Project Report
### EWB Courses — Cloud Networking Lab

---

**Student Name:** Gayathri  
**AWS Account ID:** 812704795287  
**Region:** Asia Pacific (Mumbai) — `ap-south-1`  
**Date:** November 1, 2026  

---

## Table of Contents

1. [Project Title and Objective](#1-project-title-and-objective)
2. [Introduction to VPC Peering](#2-introduction-to-vpc-peering)
3. [Architecture Diagram](#3-architecture-diagram)
4. [AWS Services Used](#4-aws-services-used)
5. [VPC and Subnet Configuration](#5-vpc-and-subnet-configuration)
6. [Route Table Configuration](#6-route-table-configuration)
7. [Security Group Configuration](#7-security-group-configuration)
8. [EC2 Instance Configuration](#8-ec2-instance-configuration)
9. [Implementation Procedure](#9-implementation-procedure)
10. [Terminal Testing Screenshots](#10-terminal-testing-screenshots)
11. [Results and Observations](#11-results-and-observations)
12. [Conclusion](#12-conclusion)

---

## 1. Project Title and Objective

### Title
**Implementing AWS VPC Peering for Private Cross-VPC Communication**

### Objective
The objective of this project is to design, configure, and validate a **VPC Peering** setup on Amazon Web Services (AWS) that enables private, low-latency communication between two isolated Virtual Private Clouds (VPCs) — `ewb-vpc-a` and `ewb-vpc-b` — without routing traffic through the public internet.

**Specific Goals:**
- Create two custom VPCs with non-overlapping CIDR blocks
- Configure public and private subnets within each VPC
- Establish a VPC Peering connection between the two VPCs
- Configure route tables to direct peered traffic through the peering connection
- Define security groups to control SSH and ICMP traffic
- Launch EC2 instances (App A, App B, Bastion) and verify cross-VPC connectivity via ping
- Demonstrate ICMP blocking and restoration through security group rule management

---

## 2. Introduction to VPC Peering

### What is VPC Peering?

Amazon Virtual Private Cloud (VPC) Peering is a networking connection between two VPCs that enables routing of traffic between them using private IPv4 or IPv6 addresses. Instances in either VPC can communicate with each other as if they are within the same network.

### Key Characteristics

| Property | Detail |
|---|---|
| Traffic routing | Private IP addresses only (no internet) |
| Supports | Intra-region and inter-region peering |
| Overlapping CIDRs | **Not allowed** — CIDRs must be unique |
| Transitive peering | **Not supported** — direct peering required |
| Bandwidth | No bandwidth bottleneck (uses AWS backbone) |
| Cost | Data transfer charges apply for cross-AZ/region |

### Why Use VPC Peering?

- **Security:** Traffic never traverses the public internet
- **Performance:** Uses AWS's high-bandwidth, low-latency internal network
- **Cost Efficiency:** Eliminates the need for VPN gateways or NAT for cross-VPC communication
- **Simplicity:** No complex routing devices; managed natively by AWS
- **Use Cases:** Microservices architectures, multi-account setups, shared services VPCs

### How VPC Peering Works

1. **Requester VPC** initiates the peering connection request
2. **Accepter VPC** accepts the request (can be same or different account/region)
3. **Route tables** in both VPCs are updated to route traffic for the peer CIDR through the peering connection
4. **Security groups** are configured to allow the desired traffic types
5. Instances in both VPCs can now communicate privately

---

## 3. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AWS Region: ap-south-1 (Mumbai)                 │
│                                                                         │
│  ┌────────────────────────────────┐   ┌────────────────────────────────┐│
│  │       ewb-vpc-a                │   │       ewb-vpc-b                ││
│  │       10.10.0.0/16             │   │       10.20.0.0/16             ││
│  │                                │   │                                ││
│  │  ┌──────────────────────────┐  │   │  ┌──────────────────────────┐ ││
│  │  │  ewb-a-public subnet     │  │   │  │  ewb-b-private subnet    │ ││
│  │  │  (ewb-rt-a-public)       │  │   │  │  (ewb-rt-b-private)      │ ││
│  │  │                          │  │   │  │                          │ ││
│  │  │  ┌──────────────┐        │  │   │  │  ┌──────────────┐        │ ││
│  │  │  │ ewb-bastion  │        │  │   │  │  │  ewb-app-b   │        │ ││
│  │  │  │ (t3.micro)   │        │  │   │  │  │  (t3.micro)  │        │ ││
│  │  │  │ 13.232.78.159│        │  │   │  │  │ 10.20.1.10   │        │ ││
│  │  │  └──────────────┘        │  │   │  └──────────────────────────┘ ││
│  │  └──────────────────────────┘  │   │                                ││
│  │                                │   └────────────────────────────────┘│
│  │  ┌──────────────────────────┐  │             │  ▲                    │
│  │  │  ewb-a-private subnet    │  │             │  │  VPC Peering       │
│  │  │  (ewb-rt-a-private)      │  │             │  │  (pcx-01dcf9a...)  │
│  │  │                          │  │    ◄────────┘  │                    │
│  │  │  ┌──────────────┐        │  │    ────────────►                    │
│  │  │  │  ewb-app-a   │        │  │                                     │
│  │  │  │  (t3.micro)  │        │  │                                     │
│  │  │  │  10.10.2.10  │        │  │                                     │
│  │  │  └──────────────┘        │  │                                     │
│  │  └──────────────────────────┘  │                                     │
│  └────────────────────────────────┘                                     │
│                │                                                         │
│         ┌──────▼──────┐                                                  │
│         │  ewb-igw-a  │ (Internet Gateway for Bastion SSH access)        │
│         │ igw-05cae.. │                                                  │
│         └─────────────┘                                                  │
│                │                                                         │
│           [Internet]                                                     │
└─────────────────────────────────────────────────────────────────────────┘

Traffic flow (private):
  Bastion (10.10.1.x) ──SSH──► App-A (10.10.2.10) ──ICMP──► App-B (10.20.1.10)
  App-B (10.20.1.10) ──ICMP──► App-A (10.10.2.10)
  [All cross-VPC traffic via VPC Peering — no internet routing]
```

---

## 4. AWS Services Used

| Service | Resource | Purpose |
|---|---|---|
| **Amazon VPC** | ewb-vpc-a, ewb-vpc-b | Isolated network environments |
| **VPC Subnets** | ewb-a-public, ewb-a-private, ewb-b-private | Network segmentation |
| **Internet Gateway** | ewb-igw-a | Public internet access for Bastion host |
| **VPC Peering** | ewb-a-b-peer | Private cross-VPC communication |
| **Route Tables** | ewb-rt-a-public, ewb-rt-a-private, ewb-rt-b-private | Traffic routing rules |
| **Security Groups** | ewb-sg-app-a, ewb-sg-app-b, ewb-sg-bastion | Firewall rule management |
| **Amazon EC2** | ewb-app-a, ewb-app-b, ewb-bastion | Application & jump-box instances |
| **EC2 Key Pair** | ewb-lab.pem | SSH authentication |

---

## 5. VPC and Subnet Configuration

### 5.1 VPC Configuration

Two custom VPCs were created with **non-overlapping CIDR blocks** to satisfy the VPC Peering prerequisite.

| VPC Name | VPC ID | CIDR Block | State |
|---|---|---|---|
| ewb-vpc-a | vpc-02aba32da5ecb496d | 10.10.0.0/16 | ✅ Available |
| ewb-vpc-b | vpc-08814cc300cd266aa | 10.20.0.0/16 | ✅ Available |

**Screenshot — VPC A (ewb-vpc-a):**
![VPC A Configuration](01_VPC_Configuration/VPC_A_ewb-vpc-a_10.10.0.0-16.png)

**Screenshot — VPC B (ewb-vpc-b):**
![VPC B Configuration](01_VPC_Configuration/VPC_B_ewb-vpc-b_10.20.0.0-16.png)

---

### 5.2 Subnet Configuration

Three custom subnets were created across the two VPCs:

| Subnet Name | Subnet ID | VPC | CIDR Block | Type |
|---|---|---|---|---|
| ewb-a-public | subnet-02e2abcbb7304797b | ewb-vpc-a | 10.10.1.0/24 | Public |
| ewb-a-private | subnet-0ad6c6bb2f36d9272 | ewb-vpc-a | 10.10.2.0/24 | Private |
| ewb-b-private | subnet-070ebcf0fa60e1054 | ewb-vpc-b | 10.20.1.0/24 | Private |

- **ewb-a-public**: Hosts the Bastion EC2 instance; has a route to the Internet Gateway
- **ewb-a-private**: Hosts ewb-app-a; routes cross-VPC traffic through the peering connection
- **ewb-b-private**: Hosts ewb-app-b in VPC-B; routes cross-VPC traffic through the peering connection

**Screenshot — Subnets Overview:**
![Subnets Overview](02_Subnet_Configuration/Subnets_Overview.png)

### 5.3 Internet Gateway

An Internet Gateway (`ewb-igw-a`) was attached to `ewb-vpc-a` exclusively, providing internet access for the Bastion host to allow SSH access from external machines.

| IGW Name | IGW ID | State | Attached VPC |
|---|---|---|---|
| ewb-igw-a | igw-05cae3991000f5a63 | ✅ Attached | vpc-02aba32da5ecb496d / ewb-vpc-a |

**Screenshot — Internet Gateway:**
![Internet Gateway](05_Security_Groups/Internet_Gateways.png)

---

## 6. Route Table Configuration

Three custom route tables were created to direct traffic appropriately.

### 6.1 Route Tables Overview

| Route Table Name | Route Table ID | Associated Subnet |
|---|---|---|
| ewb-rt-a-public | rtb-00db1b25f0c53098f | ewb-a-public |
| ewb-rt-a-private | rtb-04f21e717c07e2f5a | ewb-a-private |
| ewb-rt-b-private | rtb-0961530be8a4fc616 | ewb-b-private |

**Screenshot — Route Tables Overview:**
![Route Tables Overview](03_Route_Tables/Route_Tables_Overview.png)

---

### 6.2 ewb-rt-a-public (Public Route Table)

This route table is associated with the **ewb-a-public** subnet and contains 3 routes:

| Destination | Target | Purpose |
|---|---|---|
| 10.10.0.0/16 | local | Local VPC-A traffic |
| 10.20.0.0/16 | pcx-01dcf9a5beed58893 | Route to VPC-B via Peering |
| 0.0.0.0/0 | igw-05cae3991000f5a63 | Internet access for Bastion |

**Screenshot — ewb-rt-a-public Routes:**
![RT A Public Routes](03_Route_Tables/ewb-rt-a-public_Routes.png)

---

### 6.3 ewb-rt-a-private (VPC-A Private Route Table)

This route table is associated with the **ewb-a-private** subnet and contains 2 routes:

| Destination | Target | Purpose |
|---|---|---|
| 10.10.0.0/16 | local | Local VPC-A traffic |
| 10.20.0.0/16 | pcx-01dcf9a5beed58893 | Route to VPC-B via Peering |

**Screenshot — ewb-rt-a-private Routes:**
![RT A Private Routes](03_Route_Tables/ewb-rt-a-private_Routes.png)

---

### 6.4 ewb-rt-b-private (VPC-B Private Route Table)

This route table is associated with the **ewb-b-private** subnet and contains 2 routes:

| Destination | Target | Purpose |
|---|---|---|
| 10.10.0.0/16 | pcx-01dcf9a5beed58893 | Route to VPC-A via Peering |
| 10.20.0.0/16 | local | Local VPC-B traffic |

**Screenshot — ewb-rt-b-private Routes:**
![RT B Private Routes](03_Route_Tables/ewb-rt-b-private_Routes.png)

> **Key Insight:** Both private route tables have routes pointing to the VPC Peering connection (`pcx-01dcf9a5beed58893`) for the peer VPC's CIDR. This is what enables bidirectional cross-VPC communication.

---

## 7. Security Group Configuration

Three security groups were created to control inbound traffic to each EC2 instance.

### 7.1 Security Group Overview

| SG Name | SG ID | VPC | Purpose |
|---|---|---|---|
| ewb-sg-app-a | sg-0b522eb9c142e0851 | ewb-vpc-a | App-A instance firewall |
| ewb-sg-app-b | sg-0b4919420d7c83026 | ewb-vpc-b | App-B instance firewall |
| ewb-sg-bastion | sg-06993848db01da7a5 | ewb-vpc-a | Bastion host firewall |

---

### 7.2 ewb-sg-app-a (Security Group for App-A)

| Rule ID | IP Version | Type | Protocol | Port |
|---|---|---|---|---|
| sgr-002462db8f16b4cc8 | IPv4 | SSH | TCP | 22 |
| sgr-028bf7e408dd10a09 | IPv4 | Custom ICMP - IPv4 (Echo Request) | ICMP | N/A |

- Allows SSH from the Bastion host for management
- Allows ICMP Echo Request (ping) from VPC-B for cross-VPC connectivity testing

**Screenshot — ewb-sg-app-a Inbound Rules:**
![SG App A](05_Security_Groups/SG_ewb-sg-app-a_Inbound.png)

---

### 7.3 ewb-sg-app-b (Security Group for App-B)

| Rule ID | IP Version | Type | Protocol | Port |
|---|---|---|---|---|
| sgr-05321082728f9a8b4 | IPv4 | All ICMP - IPv4 | ICMP | All |
| sgr-04e1b47e315a20d3a | IPv4 | SSH | TCP | 22 |

- Allows All ICMP (ping) from any source (for cross-VPC testing from App-A)
- Allows SSH on port 22 for management

**Screenshot — ewb-sg-app-b Inbound Rules:**
![SG App B](05_Security_Groups/SG_ewb-sg-app-b_Inbound.png)

---

### 7.4 ewb-sg-bastion (Security Group for Bastion)

| Rule ID | IP Version | Type | Protocol | Port |
|---|---|---|---|---|
| sgr-0e548c82a7a7822a7 | IPv4 | SSH | TCP | 22 |

- Only allows SSH (port 22) access from the internet for the jump-box host

**Screenshot — ewb-sg-bastion Inbound Rules:**
![SG Bastion](05_Security_Groups/SG_ewb-sg-bastion_Inbound.png)

---

## 8. EC2 Instance Configuration

Three EC2 instances were launched — all running **Amazon Linux 2023** on **t3.micro** in **ap-south-1a**.

### 8.1 Instance Summary

| Instance Name | Instance ID | State | Type | Subnet | Private IP |
|---|---|---|---|---|---|
| ewb-bastion | i-0dd3d7716b3f9e9e4 | ✅ Running | t3.micro | ewb-a-public | 10.10.1.x |
| ewb-app-a | i-0fc64d7ed2e4fb0e3 | ✅ Running | t3.micro | ewb-a-private | 10.10.2.10 |
| ewb-app-b | i-046f4e5ea72c38ca4 | ✅ Running | t3.micro | ewb-b-private | 10.20.1.10 |

**Key Configuration Decisions:**
- **ewb-bastion**: Placed in the public subnet with a public IP (`13.232.78.159`) to allow direct SSH access from developer machines. Acts as a jump-box.
- **ewb-app-a**: Placed in the private subnet of VPC-A (10.10.2.0/24). Only accessible via Bastion.
- **ewb-app-b**: Placed in the private subnet of VPC-B (10.20.1.0/24). Accessible via cross-VPC SSH from App-A through peering.
- All instances use the same key pair `ewb-lab.pem` for SSH authentication.

**Screenshot — EC2 Instances Running:**
![EC2 Instances](06_EC2_Instances/EC2_Instances_Running.png)

---

## 9. Implementation Procedure

The project was implemented in the following sequential steps:

### Step 1: Create VPCs
1. Navigate to **VPC Console → Your VPCs → Create VPC**
2. Create `ewb-vpc-a` with CIDR `10.10.0.0/16`
3. Create `ewb-vpc-b` with CIDR `10.20.0.0/16`
4. Verify both VPCs show **Available** status

### Step 2: Create Subnets
1. Navigate to **VPC Console → Subnets → Create Subnet**
2. Create `ewb-a-public` (10.10.1.0/24) in `ewb-vpc-a`, Availability Zone: `ap-south-1a`
3. Create `ewb-a-private` (10.10.2.0/24) in `ewb-vpc-a`, Availability Zone: `ap-south-1a`
4. Create `ewb-b-private` (10.20.1.0/24) in `ewb-vpc-b`, Availability Zone: `ap-south-1a`

### Step 3: Create and Attach Internet Gateway
1. Navigate to **VPC Console → Internet Gateways → Create Internet Gateway**
2. Create `ewb-igw-a`
3. Attach it to `ewb-vpc-a` using **Actions → Attach to VPC**

### Step 4: Establish VPC Peering Connection
1. Navigate to **VPC Console → Peering Connections → Create Peering Connection**
2. Name: `ewb-a-b-peer`
3. Requester VPC: `ewb-vpc-a` (vpc-02aba32da5ecb496d)
4. Accepter VPC: `ewb-vpc-b` (vpc-08814cc300cd266aa) — same account, same region
5. Click **Create Peering Connection**
6. Navigate to the peering connection → **Actions → Accept Request**
7. Verify Status shows **Active**

### Step 5: Configure Route Tables
1. Create route table `ewb-rt-a-public`, associate with `ewb-a-public` subnet
   - Add route: `0.0.0.0/0` → `ewb-igw-a` (internet)
   - Add route: `10.20.0.0/16` → `pcx-01dcf9a5beed58893` (peering)
2. Create route table `ewb-rt-a-private`, associate with `ewb-a-private` subnet
   - Add route: `10.20.0.0/16` → `pcx-01dcf9a5beed58893` (peering)
3. Create route table `ewb-rt-b-private`, associate with `ewb-b-private` subnet
   - Add route: `10.10.0.0/16` → `pcx-01dcf9a5beed58893` (peering)

### Step 6: Create Security Groups
1. Create `ewb-sg-bastion` in `ewb-vpc-a`:
   - Inbound: SSH (TCP 22) from `0.0.0.0/0`
2. Create `ewb-sg-app-a` in `ewb-vpc-a`:
   - Inbound: SSH (TCP 22) from Bastion security group
   - Inbound: Custom ICMP (Echo Request) from `10.20.0.0/16`
3. Create `ewb-sg-app-b` in `ewb-vpc-b`:
   - Inbound: SSH (TCP 22) from `10.10.0.0/16`
   - Inbound: All ICMP from `10.10.0.0/16`

### Step 7: Launch EC2 Instances
1. Launch `ewb-bastion`:
   - AMI: Amazon Linux 2023
   - Type: t3.micro
   - VPC: ewb-vpc-a, Subnet: ewb-a-public
   - Enable public IP auto-assign
   - Security Group: ewb-sg-bastion
   - Key Pair: ewb-lab.pem
2. Launch `ewb-app-a`:
   - VPC: ewb-vpc-a, Subnet: ewb-a-private
   - Security Group: ewb-sg-app-a
   - Key Pair: ewb-lab.pem
3. Launch `ewb-app-b`:
   - VPC: ewb-vpc-b, Subnet: ewb-b-private
   - Security Group: ewb-sg-app-b
   - Key Pair: ewb-lab.pem

### Step 8: Testing
1. Copy PEM key to Bastion: `scp -i ewb-lab.pem ewb-lab.pem ec2-user@<bastion-ip>:/home/ec2-user/`
2. SSH into Bastion: `ssh -i ewb-lab.pem ec2-user@13.232.78.159`
3. SSH from Bastion to App-A: `ssh -i ~/ewb-lab.pem ec2-user@10.10.2.10`
4. From App-A, ping App-B: `ping -c 4 10.20.1.10`
5. From App-A, SSH into App-B: `ssh -i ~/ewb-lab.pem ec2-user@10.20.1.10`
6. From App-B, ping App-A: `ping -c 4 10.10.2.10`
7. Test ICMP blocking by removing ICMP rule from App-A security group
8. Restore ICMP rule and confirm ping resumes

---

## 10. Terminal Testing Screenshots

### Test 1: SSH into Bastion → App-A (Jump-box Access)

The PEM key was first copied to the Bastion host using `scp`, then SSH was used to connect to Bastion (`13.232.78.159`), and from there to App-A (`10.10.2.10`) using the private key.

**Screenshot — SSH Bastion to App-A:**
![SSH Bastion to App A](07_Terminal_Testing/SSH_Bastion_to_App_A.png)

> **Observation:** Successful SSH chain: `Local Machine → Bastion (13.232.78.159) → App-A (10.10.2.10)`. The hostname `ip-10-10-10-1-10` confirms we are inside the private subnet instance.

---

### Test 2: App-A Pings App-B — ICMP Blocked (Initial Test)

From App-A (10.10.1.10), a ping to App-B (10.20.1.10) was attempted. The result was **100% packet loss** because the ICMP rule was not yet configured in the security group.

**Screenshot — App-A to App-B ICMP Blocked:**
![ICMP Block Then Ping](07_Terminal_Testing/App_A_to_App_B_ICMP_Block_then_Ping.png)

> **Observation:** `4 packets transmitted, 0 received, 100% packet loss` — confirmed that without the ICMP security group rule, cross-VPC pings are blocked even when peering is active.

---

### Test 3: App-A Pings App-B — After ICMP Rule Added (Restored)

After adding the **Custom ICMP Echo Request** rule to `ewb-sg-app-a`, the ping from App-A (10.10.2.10) to App-B (10.20.1.10) succeeded.

**Screenshot — App-A to App-B Ping Success + ICMP Restored:**
![Ping Success and ICMP Restored](07_Terminal_Testing/App_A_to_App_B_Ping_Success_and_ICMP_Restored.png)

> **Observation:** `4 packets transmitted, 4 received, 0% packet loss, time 3128ms`, RTT `min/avg/max/mdev = 0.170/0.182/0.192/0.009 ms` — confirms successful cross-VPC ping via peering connection.

---

### Test 4: App-B Pings App-A (Bidirectional Verification)

From App-B (10.20.1.10), pings to App-A (10.10.2.10) were sent to verify **bidirectional** VPC Peering communication.

**Screenshot — App-B to App-A Ping Success:**
![App B to App A Ping](07_Terminal_Testing/App_B_to_App_A_Ping_Success.png)

> **Observation:** Two consecutive ping tests from App-B to App-A both show `4 packets transmitted, 4 received, 0% packet loss`. RTT values of ~0.17-0.19 ms confirm low-latency private network connectivity.

---

### Test 5: ICMP Block from App-A (Blocked Outbound — Security Group Verification)

From App-A (10.10.2.10), a ping to App-B (10.20.1.10) showed `100% packet loss`, demonstrating ICMP block from App-A's perspective when the ICMP rule is absent, followed by successful re-establishment after rule restoration.

**Screenshot — App-A Ping App-B ICMP Block:**
![App A ICMP Block](07_Terminal_Testing/App_A_Ping_App_B_ICMP_Block.png)

> **Observation:** Demonstrates security group enforcement — VPC Peering alone is insufficient; security group rules must explicitly permit ICMP traffic for cross-VPC pings to succeed.

---

## 11. Results and Observations

### Summary of Test Results

| Test Case | Source | Destination | Expected | Result |
|---|---|---|---|---|
| SSH via Bastion → App-A | Local (13.232.78.159) | App-A (10.10.2.10) | ✅ Connected | **PASS** |
| SSH Bastion → App-B via App-A | Bastion | App-B (10.20.1.10) | ✅ Connected | **PASS** |
| Ping App-A → App-B (No ICMP rule) | App-A (10.10.2.10) | App-B (10.20.1.10) | ❌ Blocked | **PASS (expected block)** |
| Ping App-A → App-B (ICMP rule added) | App-A (10.10.2.10) | App-B (10.20.1.10) | ✅ Ping success | **PASS** |
| Ping App-B → App-A (bidirectional) | App-B (10.20.1.10) | App-A (10.10.2.10) | ✅ Ping success | **PASS** |
| ICMP restore verification | App-A (10.10.2.10) | App-B (10.20.1.10) | ✅ Ping resumes | **PASS** |

### Network Performance Observations

| Metric | Value | Notes |
|---|---|---|
| Latency (App-A → App-B) | ~0.170–0.192 ms | Excellent — private AWS backbone |
| Latency (App-B → App-A) | ~0.171–0.186 ms | Consistent bidirectional |
| Packet Loss (after config) | 0% | Zero packet loss |
| Packet Loss (before ICMP rule) | 100% | Security group blocking as expected |

### Key Learnings

1. **VPC Peering requires route table updates on BOTH sides** — adding the peering connection alone is not sufficient. Each VPC's route table must explicitly route the peer's CIDR through the peering connection.

2. **Security Groups are stateful** — allowing inbound ICMP automatically allows the echo reply back. No explicit outbound rule was needed.

3. **Non-overlapping CIDRs are mandatory** — VPC Peering cannot be established if the VPCs have overlapping IP ranges.

4. **VPC Peering is NOT transitive** — if VPC-A peers with VPC-B, and VPC-B peers with VPC-C, VPC-A cannot reach VPC-C through VPC-B automatically.

5. **Bastion host pattern** — private instances are best accessed via a Bastion (jump-box) in the public subnet, maintaining security while allowing administrative access.

6. **ICMP rule specificity** — `ewb-sg-app-a` used "Custom ICMP - Echo Request" while `ewb-sg-app-b` used "All ICMP". Both approaches work; the former is more restrictive and a security best practice.

---

## 12. Conclusion

This project successfully demonstrated the implementation of **AWS VPC Peering** to enable private, secure cross-VPC communication between two isolated network environments (`ewb-vpc-a` with CIDR `10.10.0.0/16` and `ewb-vpc-b` with CIDR `10.20.0.0/16`).

### What Was Achieved

✅ Two isolated VPCs created with non-overlapping CIDRs  
✅ Public and private subnets configured for network segmentation  
✅ Internet Gateway attached to VPC-A for Bastion host internet access  
✅ VPC Peering connection (`ewb-a-b-peer`) established and **Active**  
✅ Route tables updated in both VPCs to route peer traffic through the peering connection  
✅ Security groups configured for SSH and ICMP control  
✅ Three EC2 instances launched and verified running (3/3 status checks passed)  
✅ Cross-VPC ping tests passed with **0% packet loss** and sub-millisecond RTT  
✅ ICMP blocking and restoration demonstrated through security group management  
✅ Bidirectional communication verified (App-A ↔ App-B)  

### Business Value

The VPC Peering architecture established in this project is a foundational pattern used in production environments for:
- **Multi-tier architectures** (web tier in one VPC, database tier in another)
- **Multi-account AWS Organizations** (shared services VPC peered to application VPCs)
- **Microservices communication** (isolated service VPCs communicating privately)
- **Disaster Recovery** setups where resources span multiple VPCs

The low latency (~0.18 ms RTT) and zero packet loss confirm that VPC Peering leverages AWS's high-performance backbone network, making it suitable for latency-sensitive, production-grade applications.

---

*Report prepared as part of the EWB Courses AWS Networking Lab — VPC Peering Student Project.*  
*AWS Region: ap-south-1 (Asia Pacific — Mumbai)*  
*Completed: November 1, 2026*

