# Video 6 – Practical: Creating Public and Private Subnets

---

## 1. Goal

Build the four-subnet, two-AZ design from Video 5 in the AWS Console:

```
                          INTERNET
                              │
                       ┌──────┴───────┐
                       │   IGW (IG)   │
                       └──────┬───────┘
                              │
   VPC: my VPC  192.168.0.0/24
   ┌──────────────────────────────────────────────┐
   │  AZ-1a              AZ-1b                    │
   │  ┌──────────────┐   ┌──────────────┐         │
   │  │ public sub-1 │   │ public sub-2 │         │
   │  │ /26          │   │ /26          │         │
   │  └──────────────┘   └──────────────┘         │
   │                                              │
   │  ┌──────────────┐   ┌──────────────┐         │
   │  │ private sub-1│   │ private sub-2│         │
   │  │ /26          │   │ /26          │         │
   │  └──────────────┘   └──────────────┘         │
   └──────────────────────────────────────────────┘
```

CIDR plan (computed with a subnet calculator from `192.168.0.0/24`):

| Subnet | AZ | CIDR |
|--------|------|------|
| `public subnet 1` | `ap-south-1a` | `192.168.0.0/26` |
| `public subnet 2` | `ap-south-1b` | `192.168.0.64/26` |
| `private subnet 1` | `ap-south-1a` | `192.168.0.128/26` |
| `private subnet 2` | `ap-south-1b` | `192.168.0.192/26` |

Each `/26` has 64 IPs total (about **59 usable** after AWS's 5 reserved IPs).

---

## 2. Step 1 – Create the VPC

1. AWS Console → VPC → **Create VPC**.
2. Name: `my VPC`.
3. CIDR: `192.168.0.0/24`.
4. **Create VPC**.

---

## 3. Step 2 – Create Four Subnets in One Screen

1. VPC Console → **Subnets → Create subnet**.
2. Pick the VPC: `my VPC`.
3. Use **Add new subnet** four times to enter all four rows:

| Name | AZ | CIDR |
|------|------|------|
| `public subnet 1` | `ap-south-1a` | `192.168.0.0/26` |
| `public subnet 2` | `ap-south-1b` | `192.168.0.64/26` |
| `private subnet 1` | `ap-south-1a` | `192.168.0.128/26` |
| `private subnet 2` | `ap-south-1b` | `192.168.0.192/26` |

4. **Create subnets**.

> **Important:** at this moment, all four subnets are *technically private* — none of them has internet yet, because the VPC has no Internet Gateway. The "public" label is just our naming convention until we wire up the route table.

---

## 4. Step 3 – Create and Attach the Internet Gateway

1. VPC Console → **Internet Gateways → Create internet gateway**.
2. Name: `IG`.
3. **Create internet gateway**.
4. Status starts as **Detached**.
5. **Actions → Attach to VPC** → select `my VPC` → **Attach**.

The VPC now has internet potential, but no subnet uses it yet — because the route table has no `0.0.0.0/0 → IGW` route.

---

## 5. Step 4 – Understand the Main Route Table

- AWS auto-created a **Main Route Table** for the VPC.
- All four subnets are **implicitly associated** with it (they appear under "Subnets without explicit associations").
- You **cannot delete** the Main Route Table.

If we add a `0.0.0.0/0 → IGW` route directly into the Main Route Table, **all four subnets** would become public.

But our goal: **only 2 subnets should be public**, the other 2 must stay private.

### Solution

Create a **second route table** dedicated to the public subnets, and point only that one to the IGW.

---

## 6. Step 5 – Create a Custom Route Table for Public Subnets

1. VPC Console → **Route Tables → Create route table**.
2. Name: `RT for public`.
3. VPC: `my VPC`.
4. **Create**.

Now you have two route tables:

| Route Table | Used by |
|-------------|---------|
| Main Route Table (auto) | Private subnets (default) |
| `RT for public` (custom) | (still unassociated) |

---

## 7. Step 6 – Associate the Two Public Subnets to `RT for public`

1. Select `RT for public` → **Subnet associations → Edit subnet associations**.
2. Tick `public subnet 1` and `public subnet 2`.
3. **Save associations**.

> **Rule:** Each subnet can be associated with **only one** route table. As soon as you attach a subnet to `RT for public`, it stops using the Main Route Table.

State:

| Route Table | Associated subnets |
|-------------|--------------------|
| Main Route Table | `private subnet 1`, `private subnet 2` (implicit) |
| `RT for public` | `public subnet 1`, `public subnet 2` (explicit) |

---

## 8. Step 7 – Add the Internet Route to `RT for public`

1. Select `RT for public` → **Routes → Edit routes → Add route**.
2. Destination: `0.0.0.0/0`
3. Target: **Internet Gateway** → select the `IG` you created.
4. **Save changes**.
5. Route status: **Active**.

The two public subnets now have inbound + outbound internet (for resources with public IPs). The two private subnets remain isolated from the internet.

---

## 9. Final Architecture After Video 6

```
                          INTERNET
                              │
                       ┌──────┴───────┐
                       │   IGW (IG)   │
                       └──────┬───────┘
                              │
   VPC: my VPC  192.168.0.0/24
   ┌────────────────────────────────────────────────────────────┐
   │                                                            │
   │   RT for public                                            │
   │   ┌──────────────────────────┐                             │
   │   │ 192.168.0.0/24 → local   │                             │
   │   │ 0.0.0.0/0     → IGW      │                             │
   │   └──────────────────────────┘                             │
   │            ▲           ▲                                   │
   │            │           │                                   │
   │  ┌─────────┴──┐   ┌────┴─────────┐                         │
   │  │ public sub │   │ public sub-2 │     (AZ-1a / AZ-1b)     │
   │  │ -1 /26     │   │ /26          │                         │
   │  └────────────┘   └──────────────┘                         │
   │                                                            │
   │   Main Route Table                                         │
   │   ┌──────────────────────────┐                             │
   │   │ 192.168.0.0/24 → local   │                             │
   │   └──────────────────────────┘                             │
   │            ▲           ▲                                   │
   │            │           │                                   │
   │  ┌─────────┴──┐   ┌────┴─────────┐                         │
   │  │ private    │   │ private      │     (AZ-1a / AZ-1b)     │
   │  │ subnet-1   │   │ subnet-2     │                         │
   │  └────────────┘   └──────────────┘                         │
   │                                                            │
   └────────────────────────────────────────────────────────────┘
```

This is the canonical AWS two-tier VPC layout.

---

## 10. The Next Problem (cliff-hanger)

We can launch web servers in the public subnets and reach them via SSH/HTTP from the internet.

But suppose we put a database server into a private subnet:

- It has **no public IP**.
- It has **no inbound or outbound internet**.

So:

- **How do I SSH into it from my office laptop?** → Video 7.
- **How does it get OS updates / install MySQL?** → Video 8 (NAT Gateway).

---

## Key Takeaways – Video 6

1. The four-subnet, two-AZ design is built by **subnetting** `192.168.0.0/24` into four `/26` blocks.
2. After creation **all subnets are technically private** until you wire up internet.
3. The **Internet Gateway is attached to the VPC**, not to a subnet.
4. To make some subnets public and others private, **create a second route table** for the public ones; let private ones stay on the Main Route Table.
5. **Each subnet ↔ one route table.** Adding it to a custom RT removes it from the Main RT automatically.
6. The route `0.0.0.0/0 → IGW` is what makes a subnet "public".
