# Video 13 – Stateful vs Stateless Firewalls

The most-confused part of the Security Group vs Network ACL discussion. This entire video is dedicated to it so even non-technical readers can understand.

---

## 1. Why this matters

In AWS:

- **Security Group = Stateful firewall**
- **Network ACL = Stateless firewall**

They both have **inbound** and **outbound** rules, but they **handle return traffic very differently**. Misunderstanding this leads to "I allowed inbound, why does my reply still fail?" type of confusion.

---

## 2. The Common Setup (firewalls have inbound + outbound)

Whether a firewall is stateful or stateless, it sits in the path of traffic and has two rule sets:

```
                  ┌────────────────────┐
                  │     Firewall       │
                  │  ┌──────────────┐  │
   Outside ─────► │  │  INBOUND     │  │ ─────► EC2
                  │  │  rules       │  │
                  │  └──────────────┘  │
                  │  ┌──────────────┐  │
   Outside ◄───── │  │  OUTBOUND    │  │ ◄───── EC2
                  │  │  rules       │  │
                  │  └──────────────┘  │
                  └────────────────────┘
```

The key question is: **what happens to the return packet of an allowed flow?**

---

## 3. Direction 1 – Internet → EC2 (someone visits your web server)

```
   USER (internet) ────HTTP request────► [Firewall] ────► Web EC2
```

Both stateful and stateless behave the **same** here:

- You **must allow inbound HTTP (port 80)**, otherwise the request is blocked.
- The reply going outbound **also needs to leave**.

For stateful (SG) the outbound reply is **auto-allowed**.
For stateless (NACL) the outbound reply needs an explicit outbound rule (typically the **ephemeral port range**, e.g., `1024-65535`).

So in this direction, both behave fine **as long as you remembered the outbound rule for NACL**.

---

## 4. Direction 2 – EC2 → Internet (your EC2 calls Google)

This is where stateful vs stateless **really diverges**.

### Stateful (Security Group)

```
   Web EC2 ───call google.com──► [SG] ───► INTERNET
   Web EC2 ◄─── reply ◄─── [SG] ◄─── INTERNET   ← AUTO-ALLOWED
```

- You allow **outbound** in the SG.
- When the reply comes back, the SG **remembers the connection state** and lets it back in **without an explicit inbound rule**.
- **Even if your inbound rule is empty (zero rules), the reply still comes through** because of state tracking.

> The firewall is "intelligent" — it notes down the state when the connection was initiated and lets the matching reply pass automatically.

### Stateless (Network ACL)

```
   EC2 ───call google.com──► [NACL] ───► INTERNET
   EC2 ◄─── reply ◄─── [NACL] ◄─── INTERNET   ← MUST have inbound rule
```

- You allow **outbound** in the NACL.
- The reply coming **back is treated as a brand-new flow**.
- You **also need** an explicit **inbound** rule allowing the reply (typically the ephemeral port range that the OS chose for the reply destination port).
- Forget the inbound rule → the reply is dropped → your EC2 thinks Google is unreachable.

---

## 5. Side-by-Side: Allowing Outbound HTTP Calls from EC2

| What you must configure | Security Group (Stateful) | Network ACL (Stateless) |
|--------------------------|----------------------------|--------------------------|
| Outbound rule | Allow port 80 to internet | Allow port 80 to internet |
| Inbound rule for the **reply** | **Not needed** (auto) | **Required** — allow ephemeral ports `1024-65535` from internet |

This is exactly why **NACLs feel painful** — the administrator has to think about both directions.

---

## 6. Why Stateful is Lower Admin Burden

- Less to remember — only the request direction matters.
- Less chance of bug — easy to forget the ephemeral port range.
- Easier to reason about — fewer rules.

This is why **most teams rely on Security Groups** for day-to-day security and treat **NACLs as a coarser, secondary defense layer** (e.g., to block a known-bad IP across an entire subnet, or to block a port at the perimeter).

---

## 7. Quick Visual Summary

```
                  ┌────────────────────────────┐
                  │   STATEFUL  (SG)           │
                  │   ──────────────────       │
                  │   request ─►   ✓ allow     │
                  │   reply   ◄─   AUTO ✓      │
                  └────────────────────────────┘

                  ┌────────────────────────────┐
                  │   STATELESS (NACL)         │
                  │   ──────────────────       │
                  │   request ─►   ✓ allow     │
                  │   reply   ◄─   ✗ unless    │
                  │                you wrote   │
                  │                inbound rule│
                  └────────────────────────────┘
```

---

## 8. Where the Confusion Comes From

- Both firewalls *have* inbound and outbound rule lists, so people assume they behave identically.
- Both *require* you to allow the request direction.
- The difference is **invisible** until you test return traffic — then NACL "silently breaks" while SG "just works".

---

## 9. Practical Examples

### Example 1 – Web server with Security Group only

- SG inbound: allow HTTP (80) from `0.0.0.0/0`.
- SG outbound: empty (default `Allow all`).
- A user visits the site → request matches inbound → **the reply goes back automatically** because the SG is stateful.

### Example 2 – Same server but using NACL only

- NACL inbound: allow HTTP (80) from `0.0.0.0/0`.
- NACL outbound: empty.
- A user visits the site → request enters → web server tries to reply → reply is **dropped** because there's no outbound rule.
- Fix: add NACL **outbound** rule for ephemeral ports (`1024–65535`) to `0.0.0.0/0`.

### Example 3 – EC2 needs to download package updates

- Outbound HTTPS request (port 443) from EC2 to repository.
- **SG**: outbound rule for 443 → reply auto-allowed → updates work.
- **NACL**: outbound rule for 443 + **inbound rule for ephemeral ports** → updates work.

---

## 10. Why Ephemeral Ports?

When a client (or your EC2 acting as a client) opens a connection, the OS picks a random **source port** in the **ephemeral range** (commonly `1024-65535` on Linux). The destination server replies to that ephemeral port.

- For inbound rules in a stateless NACL, you cannot know in advance which ephemeral port the client used, so you allow the **entire ephemeral range**.
- AWS' documented ephemeral range to be safe: `1024-65535`.
- This is exactly the kind of detail Security Groups *hide* by being stateful.

---

## 11. Cheat Sheet

| Aspect | Stateful (SG) | Stateless (NACL) |
|--------|----------------|-------------------|
| Tracks connection state | ✅ | ❌ |
| Need explicit inbound rule for reply (when you initiated outbound)? | ❌ Auto-allowed | ✅ Required |
| Need explicit outbound rule for reply (when you allowed inbound)? | ❌ Auto-allowed | ✅ Required |
| Admin overhead | Lower | Higher |
| Where to use | Day-to-day per-resource control | Subnet-wide guardrails / deny lists |

---

## Key Takeaways – Video 13

1. **Stateful firewalls (Security Groups) auto-allow return traffic** based on the original direction.
2. **Stateless firewalls (Network ACLs) treat each direction independently** — you must write rules for both.
3. For NACLs, return traffic typically uses **ephemeral ports `1024-65535`**, so add those in the opposite direction.
4. Stateful = **lower admin burden**, which is why most production setups put their fine-grained rules in **Security Groups** and use NACLs only for coarse subnet policies.
5. Forgetting the second-direction rule on a stateless NACL is the **#1 cause of "my traffic is silently dropping" issues** in AWS networking.
