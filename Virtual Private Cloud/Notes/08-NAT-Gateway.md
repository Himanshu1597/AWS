# Video 8 – NAT Gateway (Outbound Internet for Private Subnets)

---

## 1. The Problem

Inside a private subnet, an EC2 instance:

- Has **no public IP**.
- Has **no inbound internet** (good — keeps it secure).
- Has **no outbound internet** (bad — cannot install/update software).

Try this from a private EC2 (via EC2 Instance Connect Endpoint or bastion):

```bash
ping google.com                # fails
yum install httpd              # nothing-to-do or stuck
yum install docker.io          # stuck downloading
```

Without outbound internet you cannot:

- Update the operating system
- Install MariaDB / MySQL / any package
- Install or update antivirus signatures
- Pull any external dependency

**Goal:** Provide **outbound-only** internet for private-subnet EC2s, while keeping **inbound** blocked.

---

## 2. Two Reasons it Currently Doesn't Work

1. The private subnet's route table (the Main RT) **has no route to the IGW**, so traffic to `0.0.0.0/0` has nowhere to go.
2. Even if it did, the **Internet Gateway only services resources with a public IP**. The private EC2 has no public IP.

We could add a `0.0.0.0/0 → IGW` route, but that would also let inbound internet in (defeats the purpose), and the IGW still wouldn't accept private-IP traffic.

**We need a middle-man** — an "agent" inside the public subnet that:

- Has a **public IP** (so it can talk to the IGW).
- Forwards outbound traffic from private EC2s to the IGW.
- Returns the responses back to the private EC2s.
- **Does not** allow inbound connections from the internet.

That agent is the **NAT Gateway**.

---

## 3. What is a NAT Gateway?

- **NAT** stands for **Network Address Translator**.
- AWS-managed component that translates private IPs to a public IP for outbound traffic.
- Provides **outbound-only** internet to resources in private subnets.
- It is similar in concept to the wireless router/NAT device at home.

### Important rule – Where does it live?

> **A NAT Gateway is always placed in a PUBLIC subnet** — not a private one.

Why? The NAT Gateway itself needs internet (to forward traffic to the IGW). If we put it in a private subnet, it has no internet itself, and so cannot forward anything.

In live classes the instructor asks students to vote — "Public or Private?" Most students answer "Private" intuitively. The correct answer is **Public**.

```
                                 INTERNET
                                     │
                              ┌──────┴───────┐
                              │  Internet    │
                              │   Gateway    │
                              └──────┬───────┘
                                     │  (only public-IP traffic)
                                     ▼
                          ┌──────────────────┐
   PUBLIC subnet  ───────►│   NAT Gateway    │   (has Public IP)
                          └────────┬─────────┘
                                   ▲
                                   │ (translates private→public)
                                   │
   PRIVATE subnet  ─────►   private EC2 (no public IP)
```

---

## 4. How NAT Gateway works (step by step)

1. Private EC2 makes a request, e.g., to `google.com`.
2. The private subnet's **route table** has `0.0.0.0/0 → NAT Gateway`, so traffic goes to the NAT GW.
3. NAT GW notices the source IP is private (won't survive the internet) and **rewrites the source** to its **own public IP**.
4. NAT GW sends the request to the **Internet Gateway**.
5. IGW forwards to the public internet (e.g., Google).
6. The reply comes back to the **NAT GW's public IP**.
7. NAT GW translates the destination back to the **original private IP** and forwards the reply to the private EC2.
8. Private EC2 receives the response — **only outbound** initiated traffic ever flowed.

> Inbound connections from the internet **cannot start through a NAT GW**, because there's nothing to "translate" yet — that's why private subnets stay safe.

---

## 5. Hands-on – Creating a NAT Gateway

### Step 1 – Create

1. VPC Console → **NAT Gateways → Create NAT gateway**.
2. Name: any (e.g., `my-natgw`).
3. **Subnet:** select a **public subnet** (e.g., `public subnet 1`).
4. Connectivity type: **Public**.
5. Elastic IP: click **Allocate Elastic IP** — this becomes the NAT GW's public IP.
6. **Create NAT gateway**.
7. Status starts **Pending** → wait until **Available**.

### Step 2 – Add a route in the **private** route table

Until you do this, private subnets still have no path to the NAT GW.

1. VPC Console → **Route Tables** → select the **Main Route Table** (the one used by your private subnets).
2. Routes tab → **Edit routes → Add route**.
3. Destination: `0.0.0.0/0`
4. Target: **NAT Gateway** → pick the NAT GW you just created.
5. **Save changes**.
6. Status should be **Active**.

> **Watch out:** the dropdown also shows your IGW. **Do not pick IGW** here. If you accidentally select a non-existent NAT, the route shows "blackhole". Choose the correct NAT GW; the route should turn **Active**.

### Step 3 – Test from the private EC2

After SSH-ing into the database server (via bastion or endpoint):

```bash
ping google.com                # works ✅
yum install docker.io          # downloads + installs ✅
```

Outbound internet is now functional. Inbound is still blocked.

---

## 6. Final Two-Tier + NAT Architecture

```
                                  INTERNET
                                      │
                               ┌──────┴───────┐
                               │   IGW (IG)   │
                               └──────┬───────┘
                                      │
   VPC: my VPC  192.168.0.0/24
   ┌───────────────────────────────────────────────────────────┐
   │                                                           │
   │   RT for public:  0.0.0.0/0 → IGW                         │
   │   ┌────────────┐    ┌────────────┐                        │
   │   │ public     │    │ public     │                        │
   │   │ subnet-1   │    │ subnet-2   │                        │
   │   │  ┌──────┐  │    │  ┌──────┐  │                        │
   │   │  │ Web  │  │    │  │ Web  │  │                        │
   │   │  └──────┘  │    │  └──────┘  │                        │
   │   │  ┌──────┐  │    │            │                        │
   │   │  │NATGW │  │    │            │                        │
   │   │  └───┬──┘  │    │            │                        │
   │   └──────┼─────┘    └────────────┘                        │
   │          │                                                │
   │          │ (private RT has 0.0.0.0/0 → NAT GW)            │
   │          ▼                                                │
   │   Main RT (private):  0.0.0.0/0 → NAT GW                  │
   │   ┌────────────┐    ┌────────────┐                        │
   │   │ private    │    │ private    │                        │
   │   │ subnet-1   │    │ subnet-2   │                        │
   │   │  ┌──────┐  │    │  ┌──────┐  │                        │
   │   │  │  DB  │  │    │  │  DB  │  │                        │
   │   │  └──────┘  │    │  └──────┘  │                        │
   │   └────────────┘    └────────────┘                        │
   │                                                           │
   └───────────────────────────────────────────────────────────┘
```

---

## 7. NAT Gateway Specs (good to remember)

- **Bandwidth:** up to **45 Gbps** per NAT Gateway.
- **Concurrent connections:** up to **55,000** simultaneously.
- **Highly available** within a single AZ (AWS-managed). For fault tolerance across AZs, deploy **one NAT GW per AZ** and route each private subnet to its own AZ's NAT GW.
- **No inbound** — even if a route existed, NAT does not initiate sessions inbound.

---

## 8. Pricing Warning – CLEAN UP after practice

> **NAT Gateway is chargeable.** Other VPC components in this series (VPC, subnets, IGW, Route Tables) are free, but **NAT Gateway is not**. It is *not* a tiny "less than ₹100/$1" service — it is a real billable resource.

After your hands-on practice, **delete it**:

1. VPC → NAT Gateways → select → **Delete**.
2. Wait until status is **Deleted** (~5 min).
3. Then go to **Elastic IPs** → select the EIP allocated for the NAT GW → **Release Elastic IP address**.
   - You **cannot release** the EIP while the NAT GW is still in "Deleting" state.
   - **Elastic IPs are also chargeable** if they sit unused. Always release them after deleting the NAT GW.

---

## 9. Common Questions

### Q. Why didn't AWS just let the IGW route private IPs directly?

By design: a NAT GW provides **stateful translation** so only outbound-initiated traffic flows. Letting an IGW handle private IPs would either expose them or require complex stateful logic — AWS pushed that responsibility into the NAT GW component.

### Q. Public subnet's "internet" is the same as Internet Gateway, right?

Yes. The **ultimate source of internet** for any AWS resource is the **Internet Gateway**. NAT Gateway is just a forwarder for private subnets — it consumes its internet from the IGW.

### Q. Can I use a NAT Gateway across AZs?

Functionally yes — a private subnet in AZ-A can route to a NAT GW in AZ-B. But if AZ-B fails, every cross-AZ subnet loses outbound internet. **Best practice:** one NAT GW per AZ.

### Q. Does NAT Gateway change the EC2's private IP?

No — the EC2 keeps its private IP. The NAT GW only rewrites the source IP **on the wire** when forwarding outside.

---

## Key Takeaways – Video 8

1. **NAT Gateway = outbound-only internet** for private subnets. Inbound is never possible through it.
2. NAT Gateway must be **placed in a PUBLIC subnet** (it needs the IGW for its own internet).
3. Two configuration steps:
   - Create the NAT Gateway (with an Elastic IP).
   - Update the **private** route table: `0.0.0.0/0 → NAT GW`.
4. Specs: up to **45 Gbps**, **55,000 concurrent connections**.
5. **NAT Gateway is paid.** **Always delete it + release the EIP** after practice.
6. The "ultimate" source of internet remains the **Internet Gateway**; NAT GW just forwards on behalf of private resources.
