# Video 3 – Subnetting (Dividing the IP Range for Multiple Subnets)

---

## 1. Recap – Where we got stuck

In Video 2 we:

- Created a VPC with CIDR `192.168.0.0/24`.
- Created `subnet 1` in `ap-south-1a` using the **whole** range `192.168.0.0/24`.
- Tried to create `subnet 2` in `ap-south-1b` for high availability → **failed** because no IPs were left.

This video explains how to fix it.

---

## 2. The Chocolate Story (a non-technical example)

> The instructor has two children — **Misri** (daughter) and **Kabir** (son).
>
> One day Misri was at home, Kabir was at school. The instructor came home with **only one chocolate** and gave it all to Misri.
>
> A minute later Kabir came back from school: *"Where is my chocolate?"* But the chocolate was already gone.

Mapping this to AWS:

| Story | AWS |
|-------|-----|
| One chocolate | One CIDR range `192.168.0.0/24` |
| Misri | `subnet 1` (got the whole range) |
| Kabir | `subnet 2` (got nothing) |

Two solutions exist for this problem.

---

## 3. Solution 1 – Add another CIDR Range to the VPC ("buy another chocolate")

You can add up to **5 CIDR ranges** to a single VPC.

### Steps

1. VPC Console → select your VPC → **Actions → Edit CIDRs**.
2. You **cannot edit** the original CIDR. Once decided, it is locked.
3. You **can add** additional CIDR blocks (up to 5 total).
4. Click **Add new IPv4 CIDR**, e.g. `192.168.1.0/24`.
5. Now create `subnet 2` in `ap-south-1b` using `192.168.1.0/24`.
6. Both subnets exist → both children are happy.

### Picture

```
   VPC: my corp VPC
   ┌─────────────────────────────────────────────┐
   │  CIDR-1: 192.168.0.0/24                     │
   │  CIDR-2: 192.168.1.0/24  (added later)      │
   │                                             │
   │   AZ-1a                  AZ-1b              │
   │   ┌──────────────┐       ┌──────────────┐   │
   │   │ subnet 1     │       │ subnet 2     │   │
   │   │ 192.168.0.0/ │       │ 192.168.1.0/ │   │
   │   │      /24     │       │      /24     │   │
   │   └──────────────┘       └──────────────┘   │
   └─────────────────────────────────────────────┘
```

### Limitation of Solution 1

You can only add up to **5 CIDR blocks** total. So you can have at most **5 subnets** if you go this route.

> **Real-world problem:** N. Virginia (`us-east-1`) has **6 AZs**. To put one subnet in each AZ for high availability, you need 6 subnets — but the 5-CIDR limit blocks you.

---

## 4. Solution 2 – Subnet the Existing Range ("split the chocolate")

Instead of adding more CIDRs, **divide** the original range into smaller pieces.

### The tool: a Subnet Calculator

A free desktop subnet calculator (or any online one) helps you split CIDR ranges.

Example used in the video:

- Input CIDR: `192.168.0.0/24`
- Required subnets: **2**
- Output:
  - `192.168.0.0/25`  → first half (128 IPs)
  - `192.168.0.128/25` → second half (128 IPs)

### Steps in AWS

1. Delete the existing oversized `subnet 1` (so the IP range is free again).
2. VPC Console → **Subnets → Create subnet**.
3. Subnet 1:
   - Name: `subnet 1`
   - AZ: `ap-south-1a`
   - CIDR: `192.168.0.0/25`
4. Subnet 2:
   - Name: `subnet 2`
   - AZ: `ap-south-1b`
   - CIDR: `192.168.0.128/25`

Now you have **two subnets in two AZs from one VPC CIDR** — no need to add extra CIDRs.

### Picture

```
   VPC: my corp VPC  CIDR: 192.168.0.0/24
   ┌─────────────────────────────────────────────┐
   │                                             │
   │   AZ-1a                  AZ-1b              │
   │   ┌──────────────┐       ┌──────────────┐   │
   │   │ subnet 1     │       │ subnet 2     │   │
   │   │192.168.0.0/25│       │192.168.0.128 │   │
   │   │              │       │     /25      │   │
   │   └──────────────┘       └──────────────┘   │
   │                                             │
   └─────────────────────────────────────────────┘
```

---

## 5. Why does AWS show 123 usable IPs (not 126)?

A `/25` block has 128 IPs. Standard networking reserves:

- Network address (1st IP)
- Broadcast address (last IP)

That should leave **126 usable IPs**.

But AWS reserves **3 additional IPs** in every subnet:

| IP in subnet | Reserved for |
|--------------|--------------|
| 1st valid IP (e.g. `192.168.0.1`) | **Router / default gateway** |
| 2nd valid IP (e.g. `192.168.0.2`) | **DNS** |
| 3rd valid IP (e.g. `192.168.0.3`) | **Future use** |

So usable IPs in a subnet = `total - 2 (network + broadcast) - 3 (AWS reserved)` → **5 fewer than the raw block size**.

For a `/25`: `128 - 2 - 3 = 123` usable IPs.

---

## 6. Practical – Launch EC2 Instances in Each Subnet

After creating the two subnets, launch one EC2 in each AZ for high availability:

### Server 1 in subnet 1

1. EC2 → Launch Instance → name `server-1a`.
2. AMI: Amazon Linux. Type: `t2.micro`.
3. Key pair: existing key.
4. **Network → Edit:** VPC = my corp VPC, Subnet = `subnet 1`.
5. **Auto-assign public IP:** disabled by default in custom subnets — **enable it manually**.
6. SSH allowed in security group → Launch.

### Server 2 in subnet 2

Before launching, change the **Auto-assign public IPv4 address** setting on `subnet 2`:

1. VPC → Subnets → select `subnet 2` → **Actions → Edit subnet settings**.
2. Tick **Enable auto-assign public IPv4 address**.

Now any EC2 launched in `subnet 2` automatically gets a public IP.

> **Note:** In the Default VPC this setting is already on. In a custom VPC it is **off** unless you enable it.

Launch the second instance just like the first; this time it auto-gets a public IP.

---

## 7. Try to SSH – it fails

From your laptop:

```bash
ssh -i cloud_fox_key.pem ec2-user@<public-ip-of-server-1>
ssh -i cloud_fox_key.pem ec2-user@<public-ip-of-server-2>
```

Both connections **time out**.

### Why?

The VPC is built but it has **no internet connection**. It is like a furnished home with no ISP cable plugged in.

> **Solution = Internet Gateway**, covered in [Video 4](04-Internet-Gateway-and-Route-Tables.md).

---

## Key Takeaways – Video 3

1. **Two ways** to give multiple subnets their own IPs:
   - Add extra CIDR blocks to the VPC (max 5).
   - Subnet (split) the existing CIDR into smaller blocks — preferred for ≥ 6 AZs.
2. AWS reserves **3 extra IPs per subnet** (router, DNS, future use), in addition to network/broadcast.
3. In a custom subnet, **auto-assign public IP is OFF by default**; enable it on the subnet or per-launch.
4. Even with public IPs, EC2 instances cannot reach the internet **until you attach an Internet Gateway and configure a route**.
