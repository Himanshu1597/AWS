# Video 5 – VPC Two-Tier Architecture (Public & Private Subnets) — Theory

---

## 1. Why we need this design

In Videos 1-4 we built a VPC with **two public subnets** and put EC2 instances inside.
That works, but it is **not secure enough** for real-world applications.

Why? Because in production, an application has multiple layers:

- **Web tier** → faces the internet, must be reachable by users.
- **Database tier** → stores user data, must **never** be exposed to the internet.

A flat "everything is public" VPC cannot enforce that separation.

---

## 2. Real-World Application Layout (Two Tiers)

```
                ┌─────────┐
   Internet --->│  USERS  │
                └────┬────┘
                     │  open website
                     ▼
            ┌────────────────┐
            │   Web Server   │  ← needs Public IP, exposed to internet
            └────────┬───────┘
                     │  internal calls
                     ▼
            ┌────────────────┐
            │ Database Server│  ← NO public IP, only internal access
            └────────────────┘
```

- **Web server** must have a public IP — it is the entry point for every user.
- **Database server** holds the most valuable asset (user data). It should **never** be reachable directly from the internet.
- The flow is:  *User → Web server → Database server.*  Users never touch the database directly.

This is the **classic Two-Tier (Web + DB) architecture**.

---

## 3. Mapping Two Tiers to AWS

To implement this safely in AWS:

```
                              INTERNET
                                  │
                                  │
                          ┌───────┴──────┐
                          │ Internet     │
                          │ Gateway      │
                          └───────┬──────┘
                                  │
   VPC
   ┌────────────────────────────────────────────────────────────┐
   │                                                            │
   │  ┌─────────────────────┐      ┌─────────────────────┐      │
   │  │ PUBLIC subnet (1a)  │      │ PUBLIC subnet (1b)  │      │
   │  │  ┌────────────┐     │      │  ┌────────────┐     │      │
   │  │  │ Web Server │     │      │  │ Web Server │     │      │
   │  │  │   (pub IP) │     │      │  │   (pub IP) │     │      │
   │  │  └─────┬──────┘     │      │  └─────┬──────┘     │      │
   │  └────────┼────────────┘      └────────┼────────────┘      │
   │           │                            │                   │
   │           ▼                            ▼                   │
   │  ┌─────────────────────┐      ┌─────────────────────┐      │
   │  │ PRIVATE subnet (1a) │      │ PRIVATE subnet (1b) │      │
   │  │  ┌────────────┐     │      │  ┌────────────┐     │      │
   │  │  │ DB Server  │     │      │  │ DB Server  │     │      │
   │  │  │ (no pub IP)│     │      │  │ (no pub IP)│     │      │
   │  │  └────────────┘     │      │  └────────────┘     │      │
   │  └─────────────────────┘      └─────────────────────┘      │
   │                                                            │
   └────────────────────────────────────────────────────────────┘
```

### Why **two** of each subnet?

- AWS best practice for **High Availability**: deploy across at least **2 Availability Zones**.
- If one AZ fails, the application stays up via the other AZ.

So you need:

- **2 public subnets** (one per AZ) — for web servers
- **2 private subnets** (one per AZ) — for database servers

Total: **4 subnets**.

---

## 4. Public Subnet vs. Private Subnet

| Property | Public Subnet | Private Subnet |
|----------|---------------|----------------|
| Connected to Internet Gateway? | **Yes** (via Route Table) | **No** |
| Inbound internet allowed? | Yes (if EC2 has public IP) | **No** |
| Outbound internet allowed? | Yes | **Only via NAT Gateway** (covered next) |
| Typical workloads | Web servers, load balancers, bastion hosts | Databases, app/back-end servers, internal services |
| Public IP on EC2? | Usually yes | Usually no |

### Definition recap

- **Public Subnet** = a subnet whose **route table** has a route `0.0.0.0/0 → Internet Gateway`. **Resources here are reachable from the internet** (when they have public IPs).
- **Private Subnet** = a subnet **without** a route to the Internet Gateway. **Resources here are not reachable from the internet.**

> The difference is **purely the route table**, not the subnet itself. A "public subnet" is just a subnet associated with a route table that has an IGW route.

---

## 5. The New Challenges We Have to Solve

Once we move DB servers to a private subnet, two new problems pop up:

1. **How do I SSH into a DB server?** It has no public IP and no inbound internet.
2. **How does the DB server get OS updates / install packages?** It needs **outbound** internet to download software, but we don't want **inbound** internet.

### The answers (covered in upcoming videos)

- **For inbound access** to a private EC2:
  - Option A: **EC2 Instance Connect Endpoint** (Video 7).
  - Option B: **Bastion Host** (Video 7).
- **For outbound internet** from a private EC2:
  - **NAT Gateway** (Video 8).

---

## 6. Why This Design is "AWS Best Practice"

- **Defense in depth**: front-end exposed, back-end isolated.
- **High availability**: two AZs, so no single AZ failure can take down the application.
- **Least exposure**: the database is invisible from the internet — hackers cannot scan or attack it directly.

---

## Key Takeaways – Video 5

1. Real apps have at least **two tiers**: web (public) and database (private).
2. AWS best practice: **2 public + 2 private subnets across 2 AZs**.
3. A subnet is "public" or "private" based on the **route table** it is associated with.
4. The DB tier must **not** have inbound internet, but it usually still **needs outbound** internet for updates → solved by **NAT Gateway**.
5. To SSH into a private EC2 you use either the **EC2 Instance Connect Endpoint** or a **Bastion Host**.
