# 🔗 AWS VPC Peering — Secure Private Network Communication

> **EWB Courses | AWS Cloud Networking Lab**  
> Implemented by: **Gayathri** | Region: `ap-south-1` (Asia Pacific — Mumbai)

---

## 📌 Project Overview

This project demonstrates how to set up **AWS VPC Peering** to enable private, secure, cross-VPC communication between two isolated Virtual Private Clouds — without routing traffic through the public internet.

Two EC2 application instances (`ewb-app-a` in VPC-A and `ewb-app-b` in VPC-B) communicate privately using AWS's internal backbone network via a VPC Peering connection. A **Bastion host** in the public subnet of VPC-A provides secure SSH jump-box access.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   AWS Region: ap-south-1 (Mumbai)               │
│                                                                  │
│   ┌─────────────────────────┐     ┌──────────────────────────┐  │
│   │      ewb-vpc-a          │     │      ewb-vpc-b            │  │
│   │      10.10.0.0/16       │     │      10.20.0.0/16         │  │
│   │                         │     │                           │  │
│   │  [ewb-a-public subnet]  │     │  [ewb-b-private subnet]   │  │
│   │   ┌──────────────┐      │     │   ┌──────────────┐        │  │
│   │   │ ewb-bastion  │      │     │   │  ewb-app-b   │        │  │
│   │   │ 13.232.78.159│      │     │   │  10.20.1.10  │        │  │
│   │   └──────┬───────┘      │     │   └──────────────┘        │  │
│   │          │ SSH           │     └──────────────────────────┘  │
│   │   [ewb-a-private subnet] │              ▲  │                  │
│   │   ┌──────▼───────┐      │◄─ Peering ───┘  │                  │
│   │   │  ewb-app-a   │      │── pcx-01dcf9a ──►│                  │
│   │   │  10.10.2.10  │      │                               │  │
│   │   └──────────────┘      │                               │  │
│   └─────────┬───────────────┘                               │  │
│             │                                               │  │
│       [ewb-igw-a]                                           │  │
│             │ (Internet — SSH access only)                      │
└─────────────────────────────────────────────────────────────────┘

Flow: Local → Bastion (public IP) → App-A (private) ←→ App-B (private VPC-B)
      All cross-VPC traffic uses VPC Peering — zero internet exposure
```

---

## 📁 Repository Structure

```
aws-vpc-peering-secure-network/
│
├── README.md                          ← You are here
├── Project_Report.md                  ← Full technical report (all 12 sections)
│
├── 01_VPC_Configuration/              ← VPC creation screenshots
│   ├── VPC_A_ewb-vpc-a_10.10.0.0-16.png
│   └── VPC_B_ewb-vpc-b_10.20.0.0-16.png
│
├── 02_Subnet_Configuration/           ← Subnet setup proof
│   └── Subnets_Overview.png
│
├── 03_Route_Tables/                   ← Route table configuration (peering routes)
│   ├── Route_Tables_Overview.png
│   ├── ewb-rt-a-public_Routes.png
│   ├── ewb-rt-a-private_Routes.png
│   └── ewb-rt-b-private_Routes.png
│
├── 04_VPC_Peering/                    ← Active peering connection proof
│   └── VPC_Peering_ewb-a-b-peer_Active.png
│
├── 05_Security_Groups/                ← Firewall rules for all instances
│   ├── Internet_Gateways.png
│   ├── SG_ewb-sg-app-a_Inbound.png
│   ├── SG_ewb-sg-app-b_Inbound.png
│   └── SG_ewb-sg-bastion_Inbound.png
│
├── 06_EC2_Instances/                  ← EC2 launch proof (all running)
│   └── EC2_Instances_Running.png
│
└── 07_Terminal_Testing/               ← SSH + ping test results
    ├── SSH_Bastion_to_App_A.png
    ├── App_A_to_App_B_ICMP_Block_then_Ping.png
    ├── App_A_to_App_B_Ping_Success_and_ICMP_Restored.png
    ├── App_B_to_App_A_Ping_Success.png
    └── App_A_Ping_App_B_ICMP_Block.png
```

---

## ⚙️ Infrastructure Summary

### VPCs

| Name | VPC ID | CIDR | State |
|---|---|---|---|
| ewb-vpc-a | vpc-02aba32da5ecb496d | 10.10.0.0/16 | ✅ Available |
| ewb-vpc-b | vpc-08814cc300cd266aa | 10.20.0.0/16 | ✅ Available |

### Subnets

| Name | CIDR | VPC | Type |
|---|---|---|---|
| ewb-a-public | 10.10.1.0/24 | ewb-vpc-a | Public (Bastion) |
| ewb-a-private | 10.10.2.0/24 | ewb-vpc-a | Private (App-A) |
| ewb-b-private | 10.20.1.0/24 | ewb-vpc-b | Private (App-B) |

### VPC Peering Connection

| Name | Connection ID | Requester | Accepter | Status |
|---|---|---|---|---|
| ewb-a-b-peer | pcx-01dcf9a5beed58893 | ewb-vpc-a (10.10.0.0/16) | ewb-vpc-b (10.20.0.0/16) | ✅ Active |

### Route Tables

| Route Table | Subnet | Key Routes |
|---|---|---|
| ewb-rt-a-public | ewb-a-public | `0.0.0.0/0` → IGW, `10.20.0.0/16` → Peering |
| ewb-rt-a-private | ewb-a-private | `10.20.0.0/16` → Peering |
| ewb-rt-b-private | ewb-b-private | `10.10.0.0/16` → Peering |

### Security Groups

| SG Name | VPC | Inbound Rules |
|---|---|---|
| ewb-sg-bastion | ewb-vpc-a | SSH (TCP 22) from `0.0.0.0/0` |
| ewb-sg-app-a | ewb-vpc-a | SSH (TCP 22) + ICMP Echo Request |
| ewb-sg-app-b | ewb-vpc-b | SSH (TCP 22) + All ICMP |

### EC2 Instances

| Name | Instance ID | Type | Subnet | Private IP | State |
|---|---|---|---|---|---|
| ewb-bastion | i-0dd3d7716b3f9e9e4 | t3.micro | ewb-a-public | 10.10.1.x | ✅ Running |
| ewb-app-a | i-0fc64d7ed2e4fb0e3 | t3.micro | ewb-a-private | 10.10.2.10 | ✅ Running |
| ewb-app-b | i-046f4e5ea72c38ca4 | t3.micro | ewb-b-private | 10.20.1.10 | ✅ Running |

---

## 🧪 Testing Results

| Test | Source | Destination | Result |
|---|---|---|---|
| SSH via Bastion to App-A | Local → Bastion → App-A | 10.10.2.10 | ✅ Success |
| App-A → App-B (no ICMP rule) | 10.10.2.10 | 10.20.1.10 | ❌ 100% packet loss (expected) |
| App-A → App-B (ICMP rule added) | 10.10.2.10 | 10.20.1.10 | ✅ 0% packet loss |
| App-B → App-A (bidirectional) | 10.20.1.10 | 10.10.2.10 | ✅ 0% packet loss |
| ICMP restore verification | 10.10.2.10 | 10.20.1.10 | ✅ Ping resumes |

**Cross-VPC latency:** ~0.17–0.19 ms RTT (AWS private backbone)

---

## 🛠️ Implementation Steps

1. **Create VPCs** — `ewb-vpc-a` (10.10.0.0/16) and `ewb-vpc-b` (10.20.0.0/16)
2. **Create Subnets** — public + private in VPC-A; private in VPC-B
3. **Attach Internet Gateway** — `ewb-igw-a` → VPC-A (for Bastion SSH access)
4. **Create VPC Peering** — Request from VPC-A → Accept in VPC-B → Status: Active
5. **Configure Route Tables** — Add peering routes in both VPCs for each other's CIDR
6. **Create Security Groups** — SSH + ICMP rules per instance role
7. **Launch EC2 Instances** — Bastion (public subnet), App-A & App-B (private subnets)
8. **Test Connectivity** — SSH chain + cross-VPC ping with ICMP blocking/restoration

---

## 💡 Key Concepts Demonstrated

- ✅ **Non-overlapping CIDRs** are required for VPC Peering
- ✅ **Route tables must be updated on BOTH sides** of the peering
- ✅ **Security groups are stateful** — inbound ICMP allows echo reply automatically
- ✅ **VPC Peering is NOT transitive** — direct peering must exist between each pair
- ✅ **Bastion host pattern** — private instances accessed securely via public jump-box
- ✅ **Zero-internet cross-VPC traffic** — all peered traffic stays on AWS backbone

---

## 📄 Full Report

See [`Project_Report.md`](./Project_Report.md) for the complete 12-section technical report including architecture details, configuration tables, step-by-step procedure, all terminal test screenshots, results analysis, and conclusion.

---

*AWS VPC Peering Lab · EWB Courses · ap-south-1 (Mumbai) · November 2026*
