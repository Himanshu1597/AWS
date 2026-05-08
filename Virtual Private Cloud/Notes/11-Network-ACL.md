# Video 11 – Network ACL (NACL)

---

## 1. The Two Layers of VPC Security

Inside a VPC, traffic going to an EC2 instance passes through **two firewall layers**:

```
            INTERNET / OTHER SOURCE
                       │
                       ▼
            ┌──────────────────────┐
            │  Layer 1 — NACL       │  ← protects the WHOLE SUBNET
            │  (Network ACL)        │
            └──────────┬────────────┘
                       │
                       ▼
            ┌──────────────────────┐
            │  Layer 2 — SG          │  ← protects a specific RESOURCE (EC2/ENI)
            │  (Security Group)      │
            └──────────┬────────────┘
                       │
                       ▼
                   EC2 Instance
```

### Building Analogy (instructor's example)

> Imagine a building with 50 offices.
>
> - **NACL** = the building's main security desk. Anyone entering or leaving the building must pass it. Protects the **entire subnet (the building)**.
> - **Security Group** = the lock on each individual office. Protects the **specific resource (the office / EC2)**.

To reach an EC2, traffic must pass **both** NACL **and** Security Group.

---

## 2. Why does NACL exist?

Even though Security Groups can lock down traffic at the EC2 level, sometimes you want a **subnet-wide rule** that no individual EC2 administrator can override:

- "Block all port 80 inbound on the database subnet."
- "Block traffic from a known-bad IP on every server in this subnet."
- "Allow only DNS (port 53) outbound from this subnet."

These are **subnet-scope policies** — exactly what NACLs are for.

---

## 3. Default Behaviour

When you create a VPC (even a Default VPC), AWS automatically creates **one Default NACL** and associates it with **every subnet** in that VPC.

The Default NACL has:

- **Inbound rule:** ALL traffic ALLOW
- **Outbound rule:** ALL traffic ALLOW

So out-of-the-box, NACL is effectively **disabled** (everything allowed) — your real protection comes from Security Groups. NACL is there but doing nothing.

---

## 4. Demo Setup (in the video)

- **Default VPC** in Mumbai, with three subnets.
- Two EC2 instances in **`ap-south-1a`** running a simple web server.
- A **single Security Group** attached to both EC2s, with **All Traffic ALLOW**.

Result: opening `http://<public-ip>` works for both servers — no security at all.

---

## 5. Creating a Custom NACL

The video walks through creating one and then watching the consequences.

### Step A – Create the NACL

1. VPC Console → **Network ACLs → Create network ACL**.
2. Name: `test`. VPC: default VPC.
3. **Create**.

### Step B – Associate the NACL with a subnet

A NACL on its own does nothing. **It must be associated with a subnet** to take effect.

1. Select the new NACL → **Subnet associations → Edit**.
2. Tick `ap-south-1a` (the subnet holding both EC2s).
3. Save. The subnet now uses the new NACL instead of the default.

### Step C – New NACL = "deny all" by default

A **brand-new** NACL starts with:

- **Inbound: deny all**
- **Outbound: deny all**

Effects:

- Website doesn't open from either server.
- `ping <public-ip>` fails.
- `ssh -i my-key.pem ec2-user@<public-ip>` fails.

This is the opposite of the Default NACL.

---

## 6. Adding Rules – the Rule Number Matters

NACL rules are **numbered**. AWS evaluates from the **lowest** rule number to the highest, and **stops at the first match**.

### Allow ICMP (ping) inbound

1. Edit inbound rules → **Add rule**.
2. Rule number: `100`. Type: **All ICMP — IPv4**. Source: `0.0.0.0/0`. Action: **Allow**.
3. Save changes.

Try to ping → still fails. **Why?**

> NACLs are **stateless**. A request and its **reply** are treated as separate flows. You allowed inbound ICMP, but the **reply leaves outbound** — and outbound has no allow rule.

### Allow All Outbound (rule 100)

1. Edit outbound rules → Add rule.
2. Number `100`, **All traffic**, Destination `0.0.0.0/0`, **Allow**.
3. Save. Now ping works.

> In Security Groups (stateful), the reply is auto-allowed. In NACLs you must allow each direction explicitly.

### Allow SSH inbound (rule 200)

Add another inbound rule: number `200`, **SSH (port 22)**, source `0.0.0.0/0`, Allow → SSH starts working.

### Allow HTTP inbound (rule 300)

Add: number `300`, **HTTP (port 80)**, source `0.0.0.0/0`, Allow → website opens.

---

## 7. The `Deny` Rule – NACL's Special Power

**Security Groups have only Allow rules.**
**NACLs let you write Allow + Deny rules.**

### Goal

Block HTTP (port 80) **only from your own laptop's IP** (e.g., `119.x.x.x`), while every other source can still open the site.

### First (wrong) attempt

Add the deny rule with a **higher** number than the broad allow:

| # | Type | Source | Action |
|---|------|--------|--------|
| 100 | All traffic | 0.0.0.0/0 | Allow |
| 200 | SSH | 0.0.0.0/0 | Allow |
| 300 | HTTP | 0.0.0.0/0 | Allow |
| 400 | HTTP | `119.x.x.x/32` | **Deny** |

Try in incognito → site **still opens**.

**Why?** Evaluation hits rule **300 first** and it allows HTTP from `0.0.0.0/0`. NACL **stops at the first match**, so rule 400 is never evaluated.

### The fix – give the deny rule a LOWER number (higher priority)

Change the deny rule number from `400` to `250`:

| # | Type | Source | Action |
|---|------|--------|--------|
| 100 | All traffic | 0.0.0.0/0 | Allow |
| 200 | SSH | 0.0.0.0/0 | Allow |
| **250** | **HTTP** | **`119.x.x.x/32`** | **Deny** |
| 300 | HTTP | 0.0.0.0/0 | Allow |

Now evaluation goes:

1. Traffic comes in from your laptop.
2. Rule 100: matches "All traffic"? → **wait** — the way the instructor explains it, NACLs are evaluated **per traffic match (port + source)**, so for HTTP from your IP it checks the more-specific match.

Either way, the practical effect:

- Open the site **from your laptop** → blocked. ✅
- Open it from **mobile data** (different IP) → works. ✅

> **Lesson:** Always put **deny rules with lower numbers than the allow rules** they need to override.

---

## 8. Rule Number Best Practice

- Use **gaps** between rule numbers (100, 200, 300 …). This leaves room to slip in higher-priority rules later (e.g., 250, 150) without renumbering.
- AWS by default uses 100 increments — follow that convention.
- You can have rules numbered up to **32766** for IPv4 (per AWS documented quotas).

---

## 9. Stateless = Two-Way Configuration

Every flow needs:

- An **inbound** rule allowing the request.
- An **outbound** rule allowing the reply (or vice versa).

Without both, traffic looks dropped. This is the most common NACL gotcha.

(See Video 13 for a deep-dive comparison of stateful vs. stateless.)

---

## 10. Hands-on Recap (commands the instructor used)

```bash
# Open the site (HTTP)
http://<public-ip>

# Ping
ping <public-ip>

# SSH
ssh -i my-key.pem ec2-user@<public-ip>
```

After each NACL change, re-run the relevant test. Keep an incognito window handy when testing browser-cached HTTP.

---

## 11. NACL Quick Facts (PDF + transcript)

| Property | Detail |
|----------|--------|
| **Assigned to** | Subnet (one NACL per subnet, one subnet only uses one NACL at a time) |
| **Rule application** | Sequential, **lowest number first**, stops at first match |
| **Rule types** | **Allow + Deny** (unlike Security Group) |
| **Stateful?** | **No — Stateless.** Inbound and outbound must each be configured |
| **Default NACL** | All Allow inbound, All Allow outbound (effectively open) |
| **New custom NACL** | All Deny inbound, All Deny outbound (effectively closed) |
| **Use case** | Subnet-wide guardrails; deny specific bad actors; defence-in-depth |

---

## Key Takeaways – Video 11

1. AWS gives every EC2 **two firewall layers**: **NACL** (subnet-wide) → **Security Group** (resource-level).
2. **NACL = building security; Security Group = office lock.**
3. NACL is **stateless** — every flow needs both inbound and outbound rules.
4. NACL rules are **ordered by rule number** (lowest first) and evaluation **stops at the first match**.
5. Only NACL supports **Deny rules** — Security Groups can't deny.
6. **Default NACL** allows everything by default; **custom NACL** denies everything by default.
7. To "block one IP" while still allowing others, the **deny rule must have a smaller rule number** than the matching allow rule.
