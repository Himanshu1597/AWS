# Video 4 – Internet Gateway and Route Tables

---

## 1. Where we are in the 5-step VPC build

```
✅ Step 1: Create VPC
✅ Step 2: Provide IP range (192.168.0.0/24)
✅ Step 3: Create Subnets (subnet 1 in AZ-1a, subnet 2 in AZ-1b)
🔄 Step 4: Attach Internet Gateway   ← THIS VIDEO
🔄 Step 5: Configure Route Table     ← THIS VIDEO
```

Two EC2 instances exist:

- `server-1a` → `subnet 1` → public IP `192.168.0.28`
- `server-1b` → `subnet 2` → public IP `192.168.0.222`

Their security groups already allow SSH (port 22) and ICMP (ping). The keys are correct. But SSH **still times out** from a laptop. Why? **No internet for the VPC.**

---

## 2. The Real-World Analogy

You may have a fully furnished house — bedrooms, living room, computer — but no internet.
To get internet you must call your **ISP** and have them plug in a connection.

In AWS, the "ISP cable" for your VPC is the **Internet Gateway (IGW)**.

---

## 3. What is an Internet Gateway?

- A **horizontally scaled, highly available, redundant** AWS-managed component.
- It is the **only** way for a VPC to talk to the public internet.
- It supports **both IPv4 and IPv6**.
- Provides **inbound and outbound** internet — but only for resources that have a **public IP**.

### Pricing

- **No cost for the IGW itself.**
- You pay only for the **outbound internet bandwidth** used by your EC2 instances. **Inbound traffic is free.**
- This bandwidth shows up in your normal EC2/data-transfer bill.

### Capacity

- AWS does not state hard limits — capacity is **virtually unlimited** and the IGW is HA by design (you don't manage it).

---

## 4. How to attach an Internet Gateway

### Step A – Create the IGW

1. VPC Console → **Internet Gateways → Create Internet Gateway**.
2. Name: e.g. `IG`.
3. Click **Create**. Status will be **Detached**.

> An IGW must be **attached to a VPC**, not to a subnet. One IGW can attach to **only one VPC**.

### Step B – Attach to your VPC

1. Select the IGW → **Actions → Attach to VPC**.
2. Pick `my corp VPC` → **Attach**.
3. Status changes to **Attached**.

---

## 5. The Route Table

Think of a **Route Table** as the **router** for your subnets — it decides where traffic goes.

### Main Route Table

- AWS automatically creates **one default Route Table per VPC**, called the **Main Route Table**.
- You **cannot delete** the Main Route Table.
- Every subnet you create is **automatically associated** with the Main Route Table unless you change it.
- In the console, association can be **explicit** (you set it manually) or **implicit** ("Subnets without explicit associations" — automatic).

### What's in the Main Route Table by default?

A single local route:

```
Destination: 192.168.0.0/24    Target: local
```

This local route lets all subnets in the VPC talk to each other privately. **It cannot be removed.**

---

## 6. Step 5 – Add a route to the Internet Gateway

Even with an IGW attached, traffic still doesn't go anywhere because the route table doesn't know it exists. Tell it:

1. VPC Console → **Route Tables → Main Route Table → Routes tab → Edit Routes**.
2. Click **Add route**.
3. Destination: `0.0.0.0/0` (means "any destination / internet").
4. Target: **Internet Gateway** → pick the IGW you created.
5. **Save changes**.
6. Route status should be **Active**.

After this:

```
ssh -i my-key.pem ec2-user@192.168.0.28      # works ✅
ssh -i my-key.pem ec2-user@192.168.0.222     # works ✅
ping google.com                              # works ✅ (outbound also up)
```

Both SSH (inbound) and `ping google.com` (outbound) now succeed.

---

## 7. Architecture After Steps 4 & 5

```
                          INTERNET
                              │
                              │
                       ┌──────┴──────┐
                       │  Internet   │
                       │   Gateway   │
                       └──────┬──────┘
                              │   (attached to VPC)
                              │
   VPC: my corp VPC  192.168.0.0/24
   ┌──────────────────────────────────────────────────┐
   │                                                  │
   │           Main Route Table                       │
   │           ┌──────────────────────────┐           │
   │           │ 192.168.0.0/24 → local   │           │
   │           │ 0.0.0.0/0      → IGW     │           │
   │           └──────────────────────────┘           │
   │                                                  │
   │   AZ-1a                  AZ-1b                   │
   │   ┌──────────────┐       ┌──────────────┐        │
   │   │ subnet 1     │       │ subnet 2     │        │
   │   │              │       │              │        │
   │   │ EC2: 0.28    │       │ EC2: 0.222   │        │
   │   │ (public IP)  │       │ (public IP)  │        │
   │   └──────────────┘       └──────────────┘        │
   │                                                  │
   └──────────────────────────────────────────────────┘
```

---

## 8. Common Questions Answered

### Q1. Can two EC2 instances in different subnets talk to each other using **private IPs**?

**Yes.** Both subnets are connected through the VPC's local route (`192.168.0.0/24 → local`).

Demo from the video:

```
# from server-1a (192.168.0.28)
ping 192.168.0.222   → reply

# from server-1b (192.168.0.222)
ping 192.168.0.28    → reply
```

Different subnets in the same VPC can always reach each other privately, by default.

### Q2. What if my EC2 has **no public IP**?

- **No inbound** internet (you cannot SSH in from your laptop over the internet).
- **No outbound** internet (the EC2 cannot reach Google, Gmail, etc.).
- The Internet Gateway **only forwards traffic for resources that have a public IP**.

You'd reach such an instance by hopping from another EC2 inside the VPC (covered in Video 7 – Bastion Host).

### Q3. What is the IGW pricing model again?

- IGW itself: **free**.
- **Inbound bandwidth: free.**
- Outbound bandwidth: **paid**, billed on the EC2/data-transfer line items.

### Q4. Is the IGW highly available?

**Yes.** AWS manages it — you don't worry about capacity, redundancy, or scaling.

---

## 9. The Picture Now Looks Familiar

The architecture you've now built (one VPC, two public subnets in two AZs, IGW + Route Table) is the **first primary design**. Real corporate apps need more — public + private subnets, NAT Gateway, VPC Endpoints — covered next.

---

## Key Takeaways – Video 4

1. **Internet Gateway (IGW)** is the only way a VPC connects to the internet.
2. Attach the IGW to the **VPC** (not a subnet); only one VPC per IGW.
3. The **Main Route Table** is auto-created per VPC and cannot be deleted; it has a default `local` route.
4. To enable internet, add a route `0.0.0.0/0 → IGW` in the route table associated with the subnet(s).
5. IGW is **free**; you pay for **outbound** data transfer only. Inbound is free.
6. IGW only routes traffic for resources that have a **public IP**.
7. EC2s in different subnets of the same VPC can communicate over **private IPs** by default through the local route.
