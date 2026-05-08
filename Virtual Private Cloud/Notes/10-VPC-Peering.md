# Video 10 – VPC Peering Connection

---

## 1. Why does VPC Peering exist?

Open with the questions the instructor asks at the start of the video:

- *"I have **two AWS accounts**, each with a VPC. Can these VPCs talk to each other?"* → **No.**
- *"I have **two VPCs in the same AWS account**. Can they talk to each other?"* → **Still No.**

By default **no two VPCs can communicate**, even if they sit in the same account. VPCs are designed to be **logically isolated**, just like two homes in the same city — being neighbours doesn't mean you share keys.

If you ever need them to talk, you must **explicitly connect them**. That mechanism is **VPC Peering**.

> Even more impressive: VPC Peering works **across accounts** and **across regions**. The instructor jokes — *"if both parties are angry with each other, they can still set up peering."*

---

## 2. What is VPC Peering?

**VPC Peering = a private, one-to-one network connection between two VPCs that lets resources route to each other using their private IP addresses, as if they were on the same network.**

### Key Features (from PDF + transcript)

- 🔄 **One-to-one** relationship — a peering connection joins exactly **two** VPCs.
- 🌍 Can peer **within the same region** or **across regions** (inter-region peering).
- 🔐 All traffic stays on the **AWS private network** — never traverses the public internet.
- ✅ Works across **separate AWS accounts** and within **AWS Organizations**.

---

## 3. Use Cases

- Sharing common services (logging, monitoring, security) between VPCs
- Multi-account setups where each business unit has its own VPC
- Multi-region active-active or DR (disaster recovery) deployments
- Migrating workloads between two VPCs without exposing them to the internet

---

## 4. Demo Architecture (used in the video)

The instructor uses **one AWS account** but **two regions** to keep the demo simple (no need to provision a second account).

```
                 AWS Account
   ┌────────────────────────────────────────────────────────────┐
   │                                                            │
   │   Region: Mumbai (ap-south-1)        Region: N.Virginia    │
   │   ┌──────────────────────────┐       ┌──────────────────┐  │
   │   │ VPC: my India VPC        │       │ VPC: my US VPC   │  │
   │   │ CIDR: 192.168.0.0/24     │◄─────►│ CIDR:            │  │
   │   │                          │ peer  │ 192.168.1.0/24   │  │
   │   │  ┌────────────────┐      │       │ ┌──────────────┐ │  │
   │   │  │ in subnet      │      │       │ │ us subnet    │ │  │
   │   │  │ 192.168.0.0/24 │      │       │ │192.168.1.0/24│ │  │
   │   │  │ ┌────────────┐ │      │       │ │┌────────────┐│ │  │
   │   │  │ │ in server  │ │      │       │ ││ us server  ││ │  │
   │   │  │ │ (priv only)│ │      │       │ ││(priv only) ││ │  │
   │   │  │ └────────────┘ │      │       │ │└────────────┘│ │  │
   │   │  └────────────────┘      │       │ └──────────────┘ │  │
   │   └──────────────────────────┘       └──────────────────┘  │
   │                                                            │
   └────────────────────────────────────────────────────────────┘
```

---

## 5. Step-by-Step Setup

### Step A – Build the India VPC (Mumbai)

1. Switch console to **Mumbai (ap-south-1)**.
2. VPC → **Create VPC** → name `my India VPC`, CIDR `192.168.0.0/24` → Create.
3. Create one **private subnet** named `in subnet` in `ap-south-1a`, CIDR `192.168.0.0/24`.
   - No Internet Gateway is created — peering uses the AWS private network, not the internet.
4. Launch an EC2 instance `in server` in this subnet:
   - Amazon Linux, t2.micro, key pair `pairing-key`.
   - **Auto-assign Public IP: disabled** (not talking to internet).
   - Security group: **Allow All Traffic** (so we can test ping/SSH cleanly without rule confusion).

### Step B – Build the US VPC (N. Virginia)

1. Switch the console to **N. Virginia (us-east-1)** in another tab. Be careful which tab/region you're in.
2. Create VPC `my US VPC`, CIDR `192.168.1.0/24`.
   > **CRITICAL RULE:** When you plan to peer two VPCs, **their CIDR ranges MUST NOT overlap**. If both used `192.168.0.0/24`, peering would be impossible (or fail at route-table time).
3. Create subnet `us subnet` in `us-east-1`, CIDR `192.168.1.0/24`.
4. Launch EC2 `us server` (Amazon Linux, t2.micro, same key, no public IP, all traffic allowed).

### Step C – Access the private India server

Both servers have no public IP — how do you SSH in?

The instructor reuses the technique from Video 7 of the previous series: **EC2 Instance Connect Endpoint**.

1. In Mumbai → VPC → **Endpoints → Create endpoint** → name `my EC2 endpoint`.
2. Type: **EC2 Instance Connect Endpoint**, VPC = `my India VPC`, Security group = default, Subnet = `in subnet`.
3. Wait until status = **Available**.
4. EC2 → select `in server` → **Connect → EC2 Instance Connect Endpoint** → choose the new endpoint → **Connect**.

(You can also create a similar endpoint in the US region for the reverse direction.)

### Step D – Try to ping US server from India server (without peering)

```bash
# inside in server (Mumbai)
ping 192.168.1.52         # private IP of us server
```

The command **hangs / no reply**. Reason: by default the two VPCs cannot talk to each other.

---

## 6. Creating the Peering Connection

Peering is a **two-way handshake** — one side **requests**, the other side **accepts**.

### Request (from Mumbai)

1. VPC Console (Mumbai) → **Peering Connections → Create peering connection**.
2. Name tag: `India USA`.
3. **VPC (Requester):** `my India VPC`.
4. Account: **My account** (here both VPCs are in the same account).
5. Region: **Another region** → `us-east-1`.
6. **VPC (Accepter):** paste the **VPC ID** of `my US VPC` (find this from the US region tab).
7. Click **Create Peering Connection**. Status: **Pending Acceptance** in Mumbai, **Initiating Request** for the other side.

### Accept (from N. Virginia)

1. Switch to N. Virginia console → VPC → **Peering Connections**.
2. Select the new pending request → **Actions → Accept request**.
3. Status will transition through **Provisioning** → **Active** (takes 2–3 minutes).
4. Both regions should show **Active**.

---

## 7. Why ping still fails after peering becomes Active

Even with status **Active**, the two servers still cannot communicate. Why?

> **Peering only enables the *possibility* of connectivity. You still must add routes in BOTH route tables so traffic knows where to go.**

This is why the instructor warned earlier about non-overlapping CIDRs — the route entry you'll add must point to the *other* VPC's range. If both VPCs share `192.168.0.0/24`, the local route already owns it and the remote route can't be added.

### Step E – Add a route in the India route table

1. In Mumbai → VPC → Route Tables → main RT for `my India VPC` → **Edit Routes**.
2. Existing entry: `192.168.0.0/24 → local` (untouchable).
3. **Add route:**
   - Destination: `192.168.1.0/24` (the US VPC's CIDR)
   - Target: **Peering connection** → select `India USA` (status Active).
4. **Save changes**.

### Step F – Add a mirror route in the US route table

1. In N. Virginia → VPC → Route Tables → main RT for `my US VPC` → **Edit Routes**.
2. Existing: `192.168.1.0/24 → local`.
3. **Add route:**
   - Destination: `192.168.0.0/24` (the India VPC's CIDR)
   - Target: peering connection `India USA`.
4. **Save changes**.

> Symmetry matters — **both sides** must have a route to the other's CIDR. Forgetting either side will silently drop traffic.

---

## 8. Verify

```bash
# inside in server (Mumbai)
ping 192.168.1.52         # works ✅ — replies coming back
```

The two EC2 instances, in different VPCs, in different regions, with no public IPs, can now talk over private IPs through the AWS backbone.

---

## 9. Tearing it Down

If you delete the peering connection:

1. VPC → Peering Connections → select → **Actions → Delete peering connection**.
2. AWS asks if you want to **delete the related routes** in the route tables — accept.
3. Communication breaks immediately.

---

## 10. Important Rules / Gotchas

| Rule | Why it matters |
|------|----------------|
| **Non-overlapping CIDRs are mandatory** | Local routes always win; peering would be unroutable. |
| **One-to-one only** | Three VPCs cannot share one peering connection — you create separate peerings per pair (A↔B, A↔C, B↔C). |
| **No transitive peering** | If A is peered with B, and B is peered with C, then A is **not** automatically connected to C. You must add A↔C separately. (For mesh / hub-and-spoke at scale, use **Transit Gateway**.) |
| **Routes are required on both sides** | Active status is necessary but not sufficient. |
| **No bandwidth bottleneck on the AWS side** | Peering uses the AWS backbone; cross-region peering has data-transfer costs. |
| **Security still applies** | Security groups & NACLs on each side still gate traffic. Peering is just routing. |

---

## 11. Where Peering Fits

Peering is the **simplest** VPC-to-VPC connection mechanism. As your network grows, manual peering becomes painful:

- **2 VPCs** → 1 peering.
- **5 VPCs** in full mesh → 10 peerings.
- **N VPCs** in full mesh → N×(N-1)/2 peerings — explodes quickly.

For larger architectures, AWS recommends **AWS Transit Gateway** (covered in future videos), which acts as a central hub.

---

## Key Takeaways – Video 10

1. Two VPCs **cannot communicate by default**, even in the same account.
2. **VPC Peering** enables private connectivity using private IPs across accounts/regions.
3. Use **non-overlapping CIDR ranges** when planning peering.
4. Setup is **request → accept → add routes on BOTH sides**.
5. Active status alone is not enough — without route table entries, traffic still fails.
6. Peering is **non-transitive** and **one-to-one**; large meshes are better handled by **Transit Gateway**.
7. All peering traffic stays on the **AWS private network** — never on the public internet.
