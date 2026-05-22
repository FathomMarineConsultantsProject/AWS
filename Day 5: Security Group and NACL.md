

# Day 5: Security Groups vs. NACLs – Securing Your Cloud Network

Now that you understand VPCs and subnets (Day 4), it’s time to lock down the traffic flowing in and out of your resources. AWS gives you **two powerful firewalls** that work at different layers: **Security Groups (SGs)** and **Network Access Control Lists (NACLs)**. They often confuse newcomers, but a simple analogy makes them crystal clear.

---

## The “Apartment Complex” Analogy

Imagine your AWS VPC as a huge, gated apartment complex.

- **VPC** → The entire complex, surrounded by a perimeter wall.  
- **Subnet** → One individual building inside the complex.  
- **EC2 Instance** → A specific apartment inside that building.  
- **NACL (Network Access Control List)** → The security guard stationed at the building’s main entrance. They check the ID of *everyone* entering and *everyone* leaving the building.  
- **Security Group (SG)** → The deadbolt lock and peephole on your apartment door. Even if someone gets past the building guard, they still can’t enter your apartment unless the door rules allow it.

If a malicious hacker wants to reach your EC2 instance, they must pass **two independent checkpoints**:

1. **The building gate (NACL)** – a subnet-level filter.  
2. **The apartment door (Security Group)** – an instance-level filter.

This layered approach is called **defense in depth** and is a core AWS security principle.

---

## Deep Dive: Security Groups (The Apartment Door)

A Security Group is a virtual firewall that controls traffic **directly to and from an EC2 instance** (or other resources like RDS databases, load balancers, etc.). You can think of it as a set of rules attached to each instance’s network interface.

### Key Characteristics

1. **Instance‑level scope**  
   A Security Group sits right at the instance boundary. You can apply the same SG to multiple instances, but every instance still gets its own firewall logic.

2. **Stateful – “Memory”**  
   Security Groups are **stateful**. If you allow an inbound request, the corresponding outbound response is automatically allowed, regardless of outbound rules. It’s like inviting a friend into your apartment; you don’t need a separate rule to let them leave.

3. **Allow rules only**  
   SGs support **only `Allow`** rules. You cannot create a “Deny” rule. If there is no explicit Allow, traffic is implicitly denied by default. This simplifies management but means you can’t block a specific bad IP directly with an SG.

4. **Default behavior**  
   - **Inbound:** Deny all traffic.  
   - **Outbound:** Allow all traffic.

   You must explicitly add inbound rules for any desired traffic (e.g., SSH on port 22, HTTP on port 80). Outbound you can restrict if needed, but by default everything outbound is open.

### How Security Group Rules Work

A rule specifies:
- **Protocol** (TCP, UDP, ICMP, etc.)  
- **Port range** (e.g., `22` or `80-100`)  
- **Source (inbound)** or **Destination (outbound)** – can be an IP range (`0.0.0.0/0`) or another Security Group ID.  
- **Description** (optional but helpful).

**Important:** When using another SG as the source, you’re allowing traffic from any instance that has that SG attached, not a specific IP. This is incredibly powerful for building secure microservice architectures—for example, allowing the web-server SG to talk to the app-server SG without ever exposing IPs.

### Example: Web Server Security Group

| Rule Type | Protocol | Port | Source | Purpose |
|-----------|----------|------|--------|---------|
| Inbound | TCP | 80 | `0.0.0.0/0` | Allow HTTP from internet |
| Inbound | TCP | 443 | `0.0.0.0/0` | Allow HTTPS from internet |
| Inbound | TCP | 22 | `203.0.113.0/24` (your office IP) | Allow SSH only from office |

Outbound is left as default (all traffic allowed).

---

## Deep Dive: NACLs (The Building Gate)

A Network Access Control List is a firewall that controls traffic **entering and leaving a subnet**. It’s an additional, optional layer that applies to all resources inside that subnet.

### Key Characteristics

1. **Subnet‑level scope**  
   A NACL is associated with a subnet. Every instance in that subnet is automatically governed by the NACL’s rules, regardless of its Security Group settings.

2. **Stateless – “No Memory”**  
   NACLs are **stateless**. If you allow an inbound request, you must also explicitly allow the corresponding outbound response (ephemeral ports), otherwise the return traffic will be blocked. The guard checks IDs both ways—every time.

3. **Both Allow and Deny rules**  
   NACLs support **both `Allow` and `Deny`** rules. This is the biggest differentiator from SGs. You can explicitly block a malicious IP address or a range.

4. **Rule evaluation by number**  
   NACL rules are evaluated in **ascending numerical order** (lowest rule number first). As soon as a rule matches the traffic, it is applied, and the rest are skipped.  
   - A rule with number `100` is evaluated before `200`.  
   - If rule `100` says **Deny** `1.2.3.4` and rule `200` says **Allow** `0.0.0.0/0`, traffic from `1.2.3.4` will still be denied because rule 100 matches first. This is the opposite of “most specific wins” – it’s **first match wins**.

5. **Default behavior**  
   - The **default NACL** that comes with your VPC allows all inbound and outbound traffic.  
   - A **custom NACL** you create denies *all* traffic until you add rules.

### How NACL Rules Work

Each NACL has separate inbound and outbound rule tables. Each rule specifies:
- **Rule number**  
- **Protocol**  
- **Port range**  
- **Source (inbound) or Destination (outbound)** – IP CIDR only (cannot use SG IDs).  
- **Allow/Deny**

Because NACLs are stateless, you must open both inbound and outbound ports for the same session. For example, if you allow inbound SSH (port 22), you also need to allow outbound traffic on ephemeral ports (typically `1024-65535`) back to the client, unless you have a broader outbound rule.

### Example: NACL for a Public Subnet

**Inbound Rules**

| Rule # | Type | Protocol | Port Range | Source | Allow/Deny |
|--------|------|----------|------------|--------|------------|
| 100 | HTTP | TCP | 80 | `0.0.0.0/0` | ALLOW |
| 110 | HTTPS | TCP | 443 | `0.0.0.0/0` | ALLOW |
| 120 | SSH | TCP | 22 | `0.0.0.0/0` | ALLOW |
| * | All traffic | ALL | ALL | `0.0.0.0/0` | DENY |

**Outbound Rules**

| Rule # | Type | Protocol | Port Range | Destination | Allow/Deny |
|--------|------|----------|------------|-------------|------------|
| 100 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | ALLOW |
| 110 | HTTP | TCP | 80 | `0.0.0.0/0` | ALLOW |
| 120 | HTTPS | TCP | 443 | `0.0.0.0/0` | ALLOW |
| * | All traffic | ALL | ALL | `0.0.0.0/0` | DENY |

The outbound rules allow response traffic on ephemeral ports and also allow the instance itself to initiate HTTP/HTTPS requests to the internet.

> **Note:** The `*` rule is a catch-all deny at the end. Since rules are evaluated in order, if no numbered rule matches, the default `*` rule (always present) denies the traffic.

---

## The Critical Comparison: Security Groups vs. NACLs

This table makes the differences stick.

| Feature | Security Group (SG) | Network ACL (NACL) |
|---------|---------------------|---------------------|
| **Scope** | Instance level | Subnet level |
| **State** | **Stateful** – return traffic automatically allowed | **Stateless** – return traffic must be explicitly allowed |
| **Rule types** | Allow only | Allow and Deny |
| **Rule evaluation** | All rules are evaluated (most permissive wins) | Rules evaluated in numerical order (first match wins) |
| **Apply to** | EC2, RDS, ELB, etc. (resources with network interfaces) | Every instance in the associated subnet |
| **Default** | Denies all inbound, allows all outbound | Default NACL allows all; custom NACL denies all |
| **Use cases** | Fine-grained application-level access control (open only specific ports for a specific app) | Broad subnet-level filtering, IP blacklisting, defense-in-depth |

---

## Practical Scenario: Defense in Depth

The video demonstrates how these two firewalls work together to create a robust security posture. Let’s explore a realistic example.

### The Setup
You have a web application with:
- A **public subnet** for your web servers (EC2 instances).
- A **private subnet** for your database (RDS).

The web servers are in a Security Group `WebSG` that allows HTTP/HTTPS from anywhere, and SSH from a bastion host. The database is in a Security Group `DBSG` that allows MySQL (port 3306) only from `WebSG`.

However, you notice a specific IP address (`198.51.100.25`) is repeatedly attempting brute‑force SSH attacks on your web servers, and you want to block it outright.

### Why you need NACLs
1. **Security Groups can’t block the IP** because they only have Allow rules. You could remove the SSH rule, but that would block legitimate SSH traffic from your office. You need a targeted Deny.
2. You go to the **NACL** associated with your public subnet and add an **inbound rule**:
   - Rule # 50: `Deny` TCP port 22 from `198.51.100.25/32`
   - The rest of the rules remain unchanged (e.g., rule # 100 allows SSH from anywhere).
3. Because the NACL rules are evaluated starting from the lowest number, the malicious IP hits rule 50 and is blocked **before it even reaches the EC2 instance**. The SG never sees the traffic.

### Layered protection in action
- Even if a developer mistakenly opens a dangerous port in the Security Group (e.g., port 22 to `0.0.0.0/0`), a properly configured NACL can still override that at the subnet boundary.
- Conversely, if the NACL is too permissive, the SG adds a second, more granular filter. Together, they implement the security principle of **least privilege** at two different layers.

### A stateless reminder
If you deny inbound SSH with a NACL rule, you don’t necessarily need to worry about return traffic for that blocked session because the inbound request never reaches the instance. But for permitted traffic, remember to open ephemeral ports outbound. For example, if your web server needs to call an external API (outbound HTTPS), the NACL outbound rules must allow TCP 443 to the destination, and the inbound rules must allow TCP ephemeral ports back from that API. Security Groups handle this automatically because of their stateful nature, but NACLs require explicit rules.

---

## Best Practices and Design Tips

1. **Start with Security Groups for most filtering** – They’re stateful, easier to manage, and integrate with IAM and service‑linked roles. Use them for application‑level access control.

2. **Use NACLs for subnet‑wide guardrails** – If you have a subnet where no internet access should ever occur (e.g., a database subnet), set its NACL to deny all inbound/outbound internet traffic. Even if an instance’s SG accidentally allows it, the NACL blocks it.

3. **Block malicious IPs at the NACL** – Because you can Deny, NACLs are your go‑to for quickly blacklisting bad actors. Automate this using AWS services like GuardDuty and Lambda if needed.

4. **Be careful with rule numbering** – Leave gaps between rule numbers (e.g., 100, 200, 300) so you can insert new rules later without reshuffling.

5. **Don’t forget ephemeral ports** – For NACLs, ensure outbound rules allow `1024-65535` back to clients if your instances serve requests, and inbound rules allow those ports if your instances initiate connections. The exact range may vary (e.g., NAT Gateway uses `1024-65535`). Always check.

6. **Use the default NACL as a baseline, then harden** – Many VPCs start with the default allow‑all NACL. As you mature, replace it with custom NACLs that are more restrictive.

7. **Test with the AWS Policy Simulator or flow logs** – Not directly for SGs/NACLs, but use VPC Flow Logs to see which traffic is accepted or rejected, helping you debug firewalls.

---

## Final Thoughts

Understanding Security Groups and NACLs is a milestone in your AWS networking journey. Together, they form the backbone of network security, giving you both flexible, stateful protection at the instance level and stateless, rule‑ordered filtering at the subnet level.

When you design a new system, think of the apartment complex analogy:  
- The apartment door (SG) decides who enters a specific flat.  
- The building gate (NACL) decides who even gets into the building.  

By using both wisely, you build a defense that is deep, resilient, and able to adapt as threats evolve.

Next time you hear “defense in depth,” you’ll know exactly what it means in the AWS world—and exactly how to implement it.
