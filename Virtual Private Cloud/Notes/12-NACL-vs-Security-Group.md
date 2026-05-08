# Video 12 – Network ACL vs Security Group: Key Differences

A common interview/exam question: "What is the difference between a Network ACL and a Security Group?" The instructor uses a 4-point mind map to clear all doubts in 10 minutes. The fourth point (stateful vs stateless) gets a full dedicated video — see [Video 13](13-Stateful-vs-Stateless-Firewalls.md).

---

## 1. The Four-Point Mind Map

```
   ┌──────────────────────────────────────────────────┐
   │     SG vs NACL — Differences (Mind Map)          │
   ├──────────────────────────────────────────────────┤
   │  1. Assigned To         (Instance / Subnet)      │
   │  2. Rule Application    (Simultaneous / Sequence)│
   │  3. Rule Types          (Allow only / Allow+Deny)│
   │  4. Statefulness        (Stateful / Stateless)   │
   └──────────────────────────────────────────────────┘
```

---

## 2. Difference #1 — Assigned To

| | Security Group | Network ACL |
|---|----------------|-------------|
| **Attached to** | A **network interface** (ENI) — i.e. attached to specific EC2 instance, Load Balancer, EFS mount target, etc. | A **subnet** |
| **Where created** | EC2 dashboard or VPC dashboard | VPC dashboard |
| **Scope** | Per-resource | Whole subnet (every resource in it) |

### Practical scenarios

**Scenario A — Boss says: "No traffic on port 80 for the whole subnet."**
→ Use a **NACL**: subnet-wide block. No EC2 admin can override it.

**Scenario B — Boss says: "Allow port 80 on this one web server only."**
→ Use a **Security Group** attached to that EC2's ENI.

> Think of NACL as the building security desk; Security Group is the lock on a single office door.

---

## 3. Difference #2 — How Rules are Evaluated

### Security Group → All rules apply **simultaneously** (no order)

If your SG inbound rules are:

| Type | Source | Action |
|------|--------|--------|
| HTTP | 0.0.0.0/0 | Allow (implicit) |
| FTP | 0.0.0.0/0 | Allow |
| NFS | 0.0.0.0/0 | Allow |
| SMTP | 0.0.0.0/0 | Allow |

All four rules evaluate at the same time. There is **no priority** — if any rule allows the traffic, it gets through. (Security Groups can only allow, so this means "the union of all allows".)

### Network ACL → Rules apply **in sequence** (rule number order)

NACL rules are evaluated **lowest number first**, stopping at the first match.

#### Failure example (from the video)

| # | Action | Rule |
|---|--------|------|
| 100 | Allow | HTTP from anywhere |
| 200 | Deny  | HTTP from `1.1.1.1` |

Traffic from `1.1.1.1`:

1. Rule 100 matches (HTTP from anywhere) → **allow** → stop.
2. Rule 200 never gets evaluated.

The deny is useless.

#### Working example

| # | Action | Rule |
|---|--------|------|
| 150 | Deny  | HTTP from `1.1.1.1` |
| 200 | Allow | HTTP from anywhere |

Traffic from `1.1.1.1`:

1. Rule 150 matches → **deny** → stop. ✅

Traffic from any other IP:

1. Rule 150 — source mismatch → skip.
2. Rule 200 → **allow**. ✅

> Always make the **most specific / deny rules numerically smaller** than the broad allow rules they should override.

---

## 4. Difference #3 — Allow vs Allow + Deny

| | Security Group | Network ACL |
|---|----------------|-------------|
| **Allow rules** | ✅ | ✅ |
| **Deny rules** | ❌ (cannot) | ✅ |
| **Implicit policy** | Anything not explicitly allowed is **denied** | Anything not matched by any rule hits the final default-deny |

### Why does this matter?

- **Security Group** cannot block a known-bad IP while allowing the rest. You'd have to allow only specific good ranges.
- **NACL** *can* express "allow everyone except this IP" using a deny rule with a smaller number than the allow rule.

> **Exam tip:** "Block a specific source IP" → only NACL can do it cleanly.

---

## 5. Difference #4 — Stateful vs Stateless

| | Security Group | Network ACL |
|---|----------------|-------------|
| Stateful? | ✅ Yes | ❌ No (stateless) |

- **Stateful (Security Group):** When you allow outbound traffic, the **return inbound reply is automatically allowed** (and vice versa). You don't write rules for return paths.
- **Stateless (NACL):** Every direction must be explicitly allowed. Outbound for the request and inbound for the reply (or vice versa).

This deep-dive deserves its own page → [Video 13 – Stateful vs Stateless Firewalls](13-Stateful-vs-Stateless-Firewalls.md).

---

## 6. Side-by-Side Cheat Sheet

| Property | Security Group | Network ACL |
|----------|----------------|-------------|
| Attaches to | Instance / ENI | Subnet |
| Number per resource | Up to 5 SGs per ENI | 1 NACL per subnet |
| Rule evaluation | Simultaneous (no order) | Sequential, by rule number, stops at first match |
| Allow rules | ✅ | ✅ |
| Deny rules | ❌ | ✅ |
| Stateful? | ✅ Stateful | ❌ Stateless |
| Default behaviour (new) | Deny inbound, Allow outbound | Allow all (default NACL) / Deny all (custom NACL) |
| Use case | Resource-level fine-grained control | Subnet-wide coarse policy, deny lists, defense-in-depth |

---

## 7. When to Use Which

| Need | Best tool |
|------|-----------|
| Allow port 22 only from your office IP for one EC2 | **Security Group** |
| Block port 25 (SMTP) outbound for the entire subnet so no malware can spam | **NACL** (deny rule) |
| Block traffic from a known-bad IP across all hosts in a subnet | **NACL** (deny rule) |
| Allow your web tier SG to reach your DB tier SG | **Security Group** (SG-as-source feature) |
| Defense-in-depth (use both layers) | **NACL + SG** together |

---

## 8. Common Misunderstandings

1. *"Security Groups have priority"* → No, they all apply at once.
2. *"NACL deny rule will always win"* → Only if its number is **smaller** than any matching allow rule.
3. *"NACL is enough, I don't need a Security Group"* → Both layers are good defence-in-depth. Security Groups give you fine-grained, stateful control per resource that NACLs cannot.
4. *"Security Groups can deny traffic"* → No. They can only allow. To "deny", you simply remove the rule.

---

## Key Takeaways – Video 12

1. **Four pillars** to remember:
   - Assigned to → Instance vs Subnet
   - Rule application → Simultaneous vs Sequential
   - Rule types → Allow only vs Allow + Deny
   - State → Stateful vs Stateless
2. **NACL** → defence at the **subnet level**, supports **deny**, must configure **both directions**.
3. **Security Group** → defence at the **resource level**, **allow-only**, return traffic auto-allowed.
4. Use **both together** for layered protection.
5. The next video explains stateful vs stateless in depth — the most-confused topic of the set.
