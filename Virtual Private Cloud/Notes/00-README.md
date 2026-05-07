# AWS Virtual Private Cloud (VPC) - Complete Notes

These notes are made from a 9-video tutorial series on AWS VPC. Every concept, example, and step is preserved in simple English with diagrams.

---

## Table of Contents

| # | Topic | File |
|---|-------|------|
| 1 | Introduction to VPC – Why it exists, isolation concept, default VPC | [01-Introduction-to-VPC.md](01-Introduction-to-VPC.md) |
| 2 | Creating VPC, Subnets, IP Addressing (Class A/B/C) | [02-Creating-VPC-and-Subnets.md](02-Creating-VPC-and-Subnets.md) |
| 3 | Subnetting – Dividing IP Range for Multiple Subnets | [03-Subnetting.md](03-Subnetting.md) |
| 4 | Internet Gateway and Route Tables | [04-Internet-Gateway-and-Route-Tables.md](04-Internet-Gateway-and-Route-Tables.md) |
| 5 | Two-Tier Architecture – Public and Private Subnets (Theory) | [05-Two-Tier-Architecture.md](05-Two-Tier-Architecture.md) |
| 6 | Practical: Creating Public and Private Subnets | [06-Public-Private-Subnets-Implementation.md](06-Public-Private-Subnets-Implementation.md) |
| 7 | Accessing EC2 Instances in Private Subnets (EC2 Instance Connect Endpoint, Bastion Host) | [07-Accessing-Private-Subnet-EC2.md](07-Accessing-Private-Subnet-EC2.md) |
| 8 | NAT Gateway – Outbound Internet for Private Subnets | [08-NAT-Gateway.md](08-NAT-Gateway.md) |
| 9 | VPC Summary – All Components Cheat Sheet | [09-VPC-Summary.md](09-VPC-Summary.md) |

---

## How to read these notes

1. Start in order from Video 1 → Video 9.
2. Each note answers: **What is it? Why does it exist? What problem does it solve? Use cases? Examples?**
3. Diagrams are drawn in ASCII for terminal/markdown rendering.
4. Hands-on/practical steps are kept as clear numbered lists.

---

## Quick "5 Step" Process to Set Up a VPC

```
Step 1: Create VPC
Step 2: Provide IP Address Range (CIDR)
Step 3: Create Subnets
Step 4: Attach Internet Gateway
Step 5: Configure Route Table
```

---

## Key Vocabulary at a Glance

| Term | One-line meaning |
|------|------------------|
| VPC | Your own private, isolated network inside AWS |
| Subnet | A slice of the VPC IP range, lives inside an Availability Zone |
| CIDR | The IP address range you choose for VPC/Subnet (e.g., `192.168.0.0/24`) |
| Internet Gateway (IGW) | The "internet connection" for the entire VPC |
| Route Table | The "router" that decides where traffic goes |
| Public Subnet | A subnet whose route table points to an Internet Gateway |
| Private Subnet | A subnet WITHOUT a route to the Internet Gateway |
| NAT Gateway | Provides only **outbound** internet to private subnets |
| Bastion Host | A jump server in the public subnet used to SSH into private instances |
