# Video 9 – VPC Summary (Cheat Sheet)

A consolidated summary of all VPC components covered so far. Use this as your "interview / exam day" quick-reference.

> **VPC is one of the most-loved interview and exam topics.** Knowing these 7 areas cold will get you through most questions on public/private subnets, IGW, NAT GW, route tables.

---

## Mind Map

```
                  ┌────────────────────┐
                  │        VPC         │
                  └─────────┬──────────┘
                            │
        ┌───────────────────┼─────────────────┐
        │                   │                 │
   ┌────▼─────┐      ┌──────▼──────┐    ┌─────▼─────┐
   │ Subnets  │      │  Gateways   │    │  Routing  │
   └────┬─────┘      └──────┬──────┘    └─────┬─────┘
        │                   │                 │
   ┌────┴────┐         ┌────┴────┐       ┌────┴────┐
   │ Public  │         │  IGW    │       │  Main   │
   │ Private │         │ NAT GW  │       │ Custom  │
   └─────────┘         └─────────┘       └─────────┘
```

---

## 1. Virtual Private Cloud (VPC)

| Item | Detail |
|------|--------|
| Definition | **VPC = a virtual network dedicated to your AWS account inside the AWS cloud.** Provides logical isolation between AWS resources of different customers. |
| IP addressing | Use private CIDR ranges (Class A `10.x`, Class B `172.16-31.x`, Class C `192.168.x`). |
| CIDR mask range allowed | **`/16` to `/28`** |
| CIDRs per VPC | Up to **5 CIDR blocks** per VPC (the first one is locked once created; you can add up to 4 more, can't change but can remove additional ones). |
| Max VPCs per region | **5 (soft limit)** — contact AWS Support to raise it. |

---

## 2. Subnets

| Item | Detail |
|------|--------|
| Definition | A **segmented portion of a VPC's IP range**, used to group resources and control routing. |
| Located in | **Exactly one Availability Zone**. A subnet can't span multiple AZs. |
| Multiple subnets per AZ? | Yes, allowed. |
| Max subnets per VPC | **200** |
| Reserved IPs (per subnet) | **5** total: network, broadcast, router, DNS, future-use. |

### Two types of subnets

| Type | Internet? | Used for | Required gateway |
|------|-----------|----------|-------------------|
| **Public** | Inbound + outbound (for resources with public IP) | Web servers, load balancers, bastion hosts | **Internet Gateway** |
| **Private** | **No inbound**; outbound only via NAT | Databases, internal app/back-end servers | **NAT Gateway** (for outbound) |

---

## 3. Internet Gateway (IGW)

| Item | Detail |
|------|--------|
| Role | **Connects the VPC to the public internet** (both inbound and outbound). |
| Configuration | **Create IGW → Attach to VPC → Add route `0.0.0.0/0 → IGW` in the public route table.** |
| Public IP requirement | A resource must have a **public IP** to use the IGW for inbound/outbound. |
| Pricing | IGW itself is **free**. You pay for **outbound bandwidth only**; inbound is free. |
| HA | AWS-managed, highly available, virtually unlimited bandwidth. |

---

## 4. NAT Gateway

| Item | Detail |
|------|--------|
| Role | Lets EC2s in **private subnets** access the internet for **OS/package/antivirus updates and downloads** — **outbound only**. |
| Placement | Always inside a **PUBLIC subnet** (needs the IGW for its own internet). |
| Configuration | **Create NAT GW (with Elastic IP) → Add route `0.0.0.0/0 → NAT GW` in the PRIVATE route table.** |
| Inbound traffic? | **Not allowed.** Pure outbound translator. |
| Concurrent connections | Up to **55,000** simultaneous connections per NAT GW. |
| Bandwidth | Up to **45 Gbps**. |
| Pricing | **Chargeable** — delete after practice. Also release the **Elastic IP**, which is also chargeable. |

---

## 5. Route Table

| Item | Detail |
|------|--------|
| Role | Controls **where network traffic from a subnet is directed** — the "router" for the VPC. |
| Default | Each VPC gets a **Main Route Table** automatically; cannot be deleted; all subnets are implicitly associated. |
| Custom route table | You can create your own (e.g., `RT for public`) and **explicitly associate** specific subnets with it. |
| Subnet ↔ RT | One subnet can be associated with **only one route table** at a time. |
| Limits | Up to **200 route tables** per VPC, **50 routes** per route table. |
| Common entries | `local` (auto, for in-VPC traffic), `0.0.0.0/0 → IGW` (public RT), `0.0.0.0/0 → NAT GW` (private RT). |

---

## 6. Putting it All Together (Final Architecture)

```
                                  INTERNET
                                      │
                               ┌──────┴───────┐
                               │  IGW         │  ← attached to VPC, free
                               └──────┬───────┘
                                      │
   VPC: 192.168.0.0/24
   ┌────────────────────────────────────────────────────────┐
   │                                                        │
   │   PUBLIC route table:                                  │
   │     192.168.0.0/24 → local                             │
   │     0.0.0.0/0      → IGW                               │
   │                                                        │
   │   ┌───────────────┐    ┌───────────────┐               │
   │   │ public sub-1  │    │ public sub-2  │               │
   │   │  Web server   │    │  Web server   │               │
   │   │ + NAT Gateway │    │               │               │
   │   └──────┬────────┘    └───────────────┘               │
   │          │                                             │
   │          │  (private RT points 0/0 here)               │
   │          ▼                                             │
   │   PRIVATE (Main) route table:                          │
   │     192.168.0.0/24 → local                             │
   │     0.0.0.0/0      → NAT GW                            │
   │                                                        │
   │   ┌───────────────┐    ┌───────────────┐               │
   │   │ private sub-1 │    │ private sub-2 │               │
   │   │  DB server    │    │  DB server    │               │
   │   └───────────────┘    └───────────────┘               │
   │                                                        │
   └────────────────────────────────────────────────────────┘
```

---

## 7. Quick Q&A Section (interview drills)

**Q1.** Where is a NAT Gateway placed?
→ Always in a **public subnet**.

**Q2.** What kind of traffic does NAT Gateway allow?
→ **Outbound only**. No inbound.

**Q3.** Difference between Internet Gateway and NAT Gateway?
→ IGW is the VPC's gateway to the public internet; provides full inbound + outbound for **public-IP** resources. NAT GW provides **outbound-only** internet for **private** subnets.

**Q4.** Maximum CIDR ranges per VPC?
→ **5**.

**Q5.** Maximum subnets per VPC?
→ **200**.

**Q6.** How many IPs does AWS reserve in each subnet, and why?
→ **5**: network, broadcast, **router** (1st valid), **DNS** (2nd valid), **future use** (3rd valid).

**Q7.** Can a subnet span multiple AZs?
→ **No.** A subnet lives in exactly one AZ. (Different from Azure.)

**Q8.** Can two subnets in the same VPC communicate by default?
→ **Yes**, via the local route in the route table — even using **private IPs**.

**Q9.** What happens if you don't have a Default VPC and try to launch an EC2?
→ Launch fails. You must build/restore a VPC first.

**Q10.** Two ways to access an EC2 in a private subnet?
→ **EC2 Instance Connect Endpoint** (IAM-authenticated) and **Bastion Host** (key-based jump server).

---

## 8. Cheat-Code Summary (one-liners)

- **VPC** = your own private network in AWS.
- **CIDR** = the IP range you pick for the VPC. Locked once chosen, but up to 5 ranges allowed.
- **Subnet** = a slice of the VPC inside one AZ.
- **Public Subnet** = subnet whose RT has `0.0.0.0/0 → IGW`.
- **Private Subnet** = subnet whose RT does NOT route to IGW (often routes to NAT GW).
- **IGW** = single VPC-wide gateway to the internet. Free, HA.
- **NAT GW** = outbound-only forwarder, lives in a public subnet, costs money.
- **Route Table** = the "router". Main RT (auto, default for all subnets) + Custom RT (you create).
- **Bastion / EC2 Instance Connect Endpoint** = ways to SSH into private EC2.

> Read this summary 1–2 days before any AWS exam or interview — that's the cheat code.

---

## What's Next?

This series is the **first milestone**. AWS VPC has many more concepts to come (in upcoming videos):

- VPC Endpoints (Gateway & Interface)
- VPC Peering
- Transit Gateway
- Network ACLs (NACL) vs. Security Groups
- VPC Flow Logs
- DNS in VPC

Once you learn Transit Gateway, the **50-routes-per-RT** limit and many advanced routing scenarios will start to make full sense.

---

🎉 You are now a **Master of VPC fundamentals**. Keep practicing.
