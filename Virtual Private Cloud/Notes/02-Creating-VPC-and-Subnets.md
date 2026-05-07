# Video 2 – Creating VPC, Subnets and Understanding IP Addressing

---

## 1. The 5-Step VPC Setup Process

Every custom VPC is built using this five-step recipe:

```
Step 1 → Create the VPC (give it a name)
Step 2 → Provide an IP address range (CIDR block)
Step 3 → Create Subnets inside the VPC
Step 4 → Attach an Internet Gateway
Step 5 → Configure the Route Table
```

This video covers Steps 1, 2 and starts on Step 3.

---

## 2. Why we delete the Default VPC

The Default VPC has limitations:

- You cannot freely choose your own IP address range.
- It is sized for small/personal use, not for big corporate apps like Swiggy or Zomato.

Real projects build their **own custom VPC** to fit their security and IP requirements. Most engineers **delete the default VPC** first to keep the console clean and avoid confusion between many subnets.

### How to delete the Default VPC

1. AWS Console → VPC → select Default VPC.
2. Click **Delete VPC**.
3. The VPC must be **empty** (no EC2 / no resources).
4. AWS asks you to type `delete default VPC` to confirm.

> Once deleted, EC2 → Launch Instance shows a blank Network (VPC) field and the launch fails. You must have a VPC to launch any EC2 instance.

### How to get the Default VPC back (if needed)

VPC Console → **Actions → Create default VPC**. AWS rebuilds it in about a minute.

---

## 3. Step 1 – Create the VPC

1. VPC Console → **Create VPC**.
2. Choose **VPC only** (the simpler option).
3. Name: e.g. `my corp VPC`.
4. (Continue to Step 2 — IP range — on the same screen.)

---

## 4. Step 2 – IP Address Range (CIDR)

When you build your VPC you must pick an IP range, just like an on-premises company picks a private network range.

For VPCs we use **Private IP ranges** (free, never routed on the internet).

### The Three Private IP Classes

| Class | Range | Default CIDR mask | Best for |
|-------|-------|------------------|----------|
| **A** | `10.0.0.0` to `10.255.255.255` | `/8` | Huge infrastructure |
| **B** | `172.16.0.0` to `172.31.255.255` | `/16` | Medium-sized infrastructure |
| **C** | `192.168.0.0` to `192.168.255.255` | `/24` | Small infrastructure |

**Example used in the video:** `192.168.0.0/24` (Class C).

After clicking **Create VPC**, the VPC is ready and Steps 1 + 2 are done.

---

## 5. Picture so far

```
       Region: ap-south-1 (Mumbai)
   ┌────────────────────────────────────────────────┐
   │                                                │
   │   VPC: "my corp VPC"  CIDR: 192.168.0.0/24     │
   │   ┌──────────────────────────────────────┐     │
   │   │                                      │     │
   │   │   AZ-1a       AZ-1b       AZ-1c      │     │
   │   │   ┌────┐      ┌────┐      ┌────┐     │     │
   │   │   │    │      │    │      │    │     │     │
   │   │   └────┘      └────┘      └────┘     │     │
   │   │                                      │     │
   │   └──────────────────────────────────────┘     │
   │                                                │
   └────────────────────────────────────────────────┘
```

The VPC exists, but no subnets yet.

---

## 6. Try to Launch an EC2 Now (and fail)

If you go to EC2 → Launch Instance:

- Network (VPC) → shows the new VPC ✅
- Subnet → **blank** ❌

Without a subnet, you cannot use any AZ. Without an AZ, no EC2 instance. **You must create at least one subnet first.**

---

## 7. Step 3 – Create a Subnet

A **subnet** is a slice of your VPC's IP range, placed inside **exactly one Availability Zone**.

> **Important rule (AWS-specific):** A subnet **cannot span multiple AZs**. (Azure allows that — AWS does not.) You can, however, create **multiple subnets in the same AZ**.

### Steps shown in the video

1. VPC Console → **Subnets → Create subnet**.
2. Pick the VPC: `my corp VPC`.
3. Name: `subnet 1`.
4. AZ: `ap-south-1a`.
5. CIDR: must come from the **VPC's range**. You cannot use any other series.
   - Example: `192.168.0.0/24` (re-using the whole VPC range for this one subnet — for now).

After creation:

```
   VPC: my corp VPC  192.168.0.0/24
   ┌────────────────────────────────────────────┐
   │                                            │
   │   AZ ap-south-1a                           │
   │   ┌─────────────────────────────┐          │
   │   │  subnet 1: 192.168.0.0/24   │          │
   │   └─────────────────────────────┘          │
   │                                            │
   │   AZ ap-south-1b   (no subnet yet)         │
   │   AZ ap-south-1c   (no subnet yet)         │
   │                                            │
   └────────────────────────────────────────────┘
```

Now EC2 → Launch Instance → the Subnet dropdown shows `subnet 1`. You can launch into AZ `ap-south-1a`.

---

## 8. The High Availability Problem

> "Bhavesh, you only have one EC2 instance in `ap-south-1a`. If that AZ goes down, your application is dead. Add redundancy."

The fix would be a second subnet in `ap-south-1b`, with another EC2 instance.

### The blocker

When you try to create `subnet 2` in `ap-south-1b`:

- VPC has the range `192.168.0.0/24`.
- The **whole** range was already given to `subnet 1`.
- You cannot reuse the same range for another subnet.
- AWS shows an error.

**Result:** You are stuck — there are no IPs left to give to `subnet 2`.

---

## 9. Why this happens (quick math)

A `/24` CIDR has only one block of 256 IPs (`192.168.0.0` – `192.168.0.255`). Once you assign that block to `subnet 1`, there is nothing left for `subnet 2` unless you either:

- Add another CIDR range to the VPC, **or**
- **Subnet** the existing range into smaller pieces.

Both solutions are explained in [Video 3 – Subnetting](03-Subnetting.md).

---

## Key Takeaways – Video 2

1. Custom VPC creation follows a **5-step process**.
2. We delete the Default VPC because of its limits and to avoid clutter.
3. **Class A/B/C private IPs** are used for VPCs (`10.x`, `172.16-31.x`, `192.168.x`).
4. A **subnet must live inside one Availability Zone** in AWS.
5. Without a subnet, you cannot use an AZ → cannot launch EC2.
6. For high availability, you need **at least two subnets in two AZs**.
7. If you give the whole VPC range to one subnet, you have no room for a second — leading to the **subnetting** problem solved in Video 3.
