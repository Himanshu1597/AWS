# Video 7 – Accessing EC2 Instances in Private Subnets

---

## 1. Setup for this Video

We already have:

- VPC with 2 public + 2 private subnets across 2 AZs.
- Internet Gateway attached.
- `RT for public` with `0.0.0.0/0 → IGW`.

Now we launch two EC2 instances:

| Name | Subnet | Public IP? | Purpose |
|------|--------|-----------|---------|
| `web` | `public subnet 1` | ✅ Yes | front-end web server |
| `database server` | `private subnet 1` | ❌ No | back-end DB server |

The **web** server is reachable from the office laptop using its public IP — no problem:

```bash
ssh -i my-aws-key.pem ec2-user@<web-public-ip>   # works ✅
```

But the **database server** has no public IP and no inbound internet:

```bash
ssh -i my-aws-key.pem ec2-user@<db-private-ip>   # fails ❌
```

> Private IPs are **not routable on the internet**, so SSH from your laptop using a private IP will never connect.

So how do we reach the DB server? The video covers **two solutions**.

---

## 2. Solution 1 – EC2 Instance Connect Endpoint

A relatively new AWS feature (~6 months old at the time of recording) that lets you SSH to a private EC2 instance through a managed AWS endpoint, **without** any bastion host.

### How it works

```
Your Browser / AWS Console
        │  (signed in to AWS — IAM authentication)
        ▼
EC2 Instance Connect Endpoint  (created inside the private subnet)
        │  (uses AWS API call)
        ▼
   Private EC2 (database server)
```

### Steps to create

1. VPC Console → **Endpoints → Create endpoint**.
2. Name: `my private ec2 access`.
3. Many endpoint types are available. For now, choose **EC2 Instance Connect Endpoint**.
4. VPC: `my VPC`.
5. Security group: default (all inbound + outbound allowed) — for now.
6. Subnet: `private subnet 1` (the subnet that contains the DB server).
7. **Create endpoint**.
8. Wait until status changes from **Pending → Available**.

### Use it to log in

1. EC2 Console → select `database server` → **Connect**.
2. Choose tab **Connect using EC2 Instance Connect Endpoint**.
3. Select the endpoint you just created.
4. Click **Connect**.

You are now inside the private EC2 instance — no public IP needed.

> No password is asked because you are **already authenticated as the AWS root/IAM user** in the console.

### Limitation – why this isn't always enough

The endpoint authenticates with **AWS API calls (IAM)**. To use it, the person logging in must have **AWS credentials** in your account.

#### Real-world problem

You hire a **freelancer** to manage the database server. You'd rather hand them a `.pem` file and an IP address than an AWS IAM user.

In that case the EC2 Instance Connect Endpoint is awkward — you don't want to give away AWS credentials. So we use Solution 2.

---

## 3. Solution 2 – Bastion Host

A **bastion host** is a small EC2 instance that lives in the **public subnet** and serves as a "jump server" into the private subnet.

```
   Your laptop                                   AWS VPC
       │                  ┌────────────────────────────────────────────┐
       │  ssh public IP   │  ┌────────────┐         ┌────────────────┐ │
       └──────────────────┼─►│ Bastion    │   ssh   │ database server│ │
                          │  │ (public IP)├────────►│ (private only) │ │
                          │  └────────────┘ private │                │ │
                          │                  IP     └────────────────┘ │
                          └────────────────────────────────────────────┘
```

### Steps

1. **Rename or launch** an EC2 in `public subnet 1` with a public IP. Call it `Bastion Host`.
   - The video repurposes the existing `web` instance and renames it to `Bastion Host`.
   - Technical naming matters — say "bastion host" in interviews, not "the public EC2 I jump through".
2. From your laptop, SSH into the bastion host:
   ```bash
   ssh -i my-aws-key.pem ec2-user@<bastion-public-ip>
   ```
3. Inside the bastion, you'd normally run:
   ```bash
   ssh -i my-aws-key.pem ec2-user@<db-private-ip>
   ```
   But the bastion **does not have the `.pem` key file** — your `.pem` lives on your laptop.

### Copy your key file to the bastion (SCP)

On Windows 10/11 the `scp` command is built-in. From a PowerShell/Command Prompt on your laptop:

```bash
scp -i my-aws-key.pem  my-aws-key.pem  ec2-user@<bastion-public-ip>:/home/ec2-user/
```

Breakdown:

- `-i my-aws-key.pem` → authenticate with this private key.
- `my-aws-key.pem` (second occurrence) → the file you want to copy.
- `ec2-user@<bastion-public-ip>:/home/ec2-user/` → destination path (on the bastion).

When the transfer shows **100%**, the key file is on the bastion. Verify with `ls` after SSH-ing in.

### SSH into the database server from the bastion

```bash
ssh -i my-aws-key.pem ec2-user@<db-private-ip>
```

#### Permission denied error?

If you get `Permission denied (publickey)`, your key file's permissions are too open. AWS recommends running `chmod` on it. The exact command is shown when you click **Connect → SSH client** on the EC2 page, e.g.:

```bash
chmod 400 my-aws-key.pem
```

Now the key file is read-only. Re-run the SSH command and you'll be in.

> The video shows a "nested" prompt — your shell is now on `database server` (`192.168.0.186`) instead of `bastion` (`192.168.0.39`). Type `exit` to return to the bastion.

### Why this is preferred for freelancers

You only need to share the `.pem` file. No AWS account access required. The freelancer SSHes into the bastion and from there into the DB server.

---

## 4. The Next Problem (cliff-hanger)

You're now logged into the database server. Try:

```bash
ping google.com               # ❌ fails, no internet
yum install httpd             # ❌ hangs, no internet
yum install docker.io         # ❌ stuck downloading
```

**Why?** The DB server is in a private subnet. Private subnets have **no internet** — by design.

But we still need to:

- Update the OS
- Install MySQL/MariaDB and other packages
- Update antivirus signatures, etc.

So we **don't want inbound internet** (hackers must not reach our DB), but we **do want outbound internet** for legitimate downloads.

This is the **NAT Gateway** problem, solved in [Video 8](08-NAT-Gateway.md).

---

## 5. Side-by-Side Comparison of the Two Solutions

| Feature | EC2 Instance Connect Endpoint | Bastion Host |
|---------|-------------------------------|--------------|
| Auth | AWS IAM credentials | `.pem` private key |
| Needs an EC2 in the public subnet? | No | Yes (extra cost) |
| Good for external admins/freelancers? | ❌ — needs AWS user | ✅ — just share key |
| Setup complexity | Lower (managed endpoint) | Higher (manage EC2, key files) |
| Audit/IAM control | Strong (CloudTrail) | Weaker (key sharing) |

---

## Key Takeaways – Video 7

1. EC2 instances in **private subnets cannot be reached directly** over the internet — no public IP, no inbound route.
2. **Two ways** to access them:
   - **EC2 Instance Connect Endpoint** — quick, IAM-authenticated, but needs AWS credentials.
   - **Bastion Host** — a public EC2 used as a jump server; only requires a `.pem` key.
3. The bastion host **does not automatically have your private key**. Use `scp` to copy it from your laptop, then `chmod 400` it.
4. Even after logging into the private EC2, **it has no internet** — package installs and OS updates fail until a NAT Gateway is configured (Video 8).
