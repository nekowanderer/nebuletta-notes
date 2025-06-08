# Network ACL (Stateless) vs Security Group (Stateful) Deep Analysis

[English](03_network_acl_stateless_vs_security_group_stateful.md) | [繁體中文](../zh-tw/03_network_acl_stateless_vs_security_group_stateful.md) | [日本語](../ja/03_network_acl_stateless_vs_security_group_stateful.md) | [Back to Index](../README.md)

---

## Core Concept Differences

### Stateless - Network ACL
**Definition**: Does not remember or track connection states and history

**Key Characteristics**:
- Each packet is **independently examined**
- Does not remember previous related outbound requests
- Inbound and Outbound rules are **processed separately**
- Must explicitly configure bidirectional rules

### Stateful - Security Group
**Definition**: Remembers and tracks connection states

**Key Characteristics**:
- Remembers outbound request connection states
- **Automatically allows** corresponding inbound response traffic
- Only requires unidirectional rule configuration
- Intelligently handles related traffic

## Practical Operation Comparison

### Scenario: EC2 accessing `https://google.com`

#### 🔄 Complete TCP Connection Flow
```
Step 1: [EC2:32451] ---------> [Google:443] (SYN: I want to establish connection)
Step 2: [EC2:32451] <--------- [Google:443] (SYN-ACK: OK, I received it)  
Step 3: [EC2:32451] ---------> [Google:443] (ACK: Received, connection established)
Step 4: [EC2:32451] ---------> [Google:443] (HTTP Request: Give me the webpage)
Step 5: [EC2:32451] <--------- [Google:443] (HTTP Response: Here's the webpage content)
```

**Explanation**:
- `32451` is the ephemeral port randomly selected by EC2 (1024-65535)
- `443` is Google's HTTPS service port

### 🚫 Network ACL (Stateless) Processing Method

**Outbound Rule Example**:
```hcl
egress {
  protocol   = "6"     # TCP
  rule_no    = 100
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 443
  to_port    = 443
}
```

**Processing Result**:
- ✅ Steps 1, 3, 4 (outbound) → **Allowed to pass**
- ❌ Steps 2, 5 (inbound) → **Denied!** 

**Root Cause**:
When Network ACL sees the packet from Step 2, it thinks:
> "There's a packet from 142.251.42.142:443 to 10.0.1.10:32451, I need to check inbound rules...
> Wait, I don't have a rule allowing this source, deny!"

**Required Additional Inbound Rules**:
```hcl
# Allow ephemeral port responses
ingress {
  protocol   = "6"     # TCP  
  rule_no    = 90
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 1024    # Client randomly selected port range
  to_port    = 65535
}

# Allow HTTPS service port responses
ingress {
  protocol   = "6"     # TCP
  rule_no    = 100
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 443
  to_port    = 443
}
```

### ✅ Security Group (Stateful) Processing Method

**Outbound Rule Example**:
```hcl
egress {
  protocol    = "tcp"
  from_port   = 443
  to_port     = 443
  cidr_blocks = ["0.0.0.0/0"]
}
```

**Processing Result**:
- ✅ Steps 1, 3, 4 (outbound) → **Allowed to pass**
- ✅ Steps 2, 5 (inbound) → **Automatically allowed!**

**Intelligent Processing**:
When Security Group sees the packet from Step 2, it thinks:
> "There's a packet from 142.251.42.142:443 to 10.0.1.10:32451...
> Let me check, oh! This is a response to a previous connection from port 32451,
> this is legitimate response traffic, allow it through!"

## Key Differences Summary Table

| Feature | Network ACL (Stateless) | Security Group (Stateful) |
|---------|--------------------------|----------------------------|
| **Operation Level** | Subnet level | Instance level |
| **Connection Memory** | ❌ Does not remember connection state | ✅ Remembers connection state |
| **Rule Requirements** | Must configure bidirectional rules | Only requires unidirectional rules |
| **Response Handling** | Need to explicitly allow response traffic | Automatically allows related responses |
| **Default Behavior** | Default deny all | Default deny inbound, allow outbound |
| **Rule Evaluation** | Evaluated by rule number order | Evaluates all rules (OR logic) |
| **Scope** | All resources within the subnet | Specific EC2 instances |

## Importance of Ephemeral Ports

### What are Ephemeral Ports?

**Definition**: Temporary ports randomly selected by clients when initiating connections

**Range**:
- Linux/Windows: 1024-65535
- Some newer systems: 32768-65535

### Why Allow Ephemeral Ports?

```
Actual Connection Example:
EC2 (IP: 10.0.1.10, Port: 32451) → Google (IP: 142.251.42.142, Port: 443)
EC2 (IP: 10.0.1.10, Port: 32451) ← Google (IP: 142.251.42.142, Port: 443)
```

**Network ACL sees the response packet**:
- **Source**: 142.251.42.142:443
- **Destination**: 10.0.1.10:32451

If there's no inbound rule allowing port 32451, this response will be blocked.

### Common Ephemeral Port Configurations

```hcl
# Allow all ephemeral ports (recommended for most cases)
ingress {
  protocol   = "6"
  rule_no    = 90
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 1024
  to_port    = 65535
}

# Only allow common range (more restrictive)
ingress {
  protocol   = "6"
  rule_no    = 90
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 32768
  to_port    = 65535
}
```

## Common Issues and Solutions

### Issue 1: Can ping out but no response

**Symptoms**:
```bash
$ ping www.google.com
# No response, seems like network is unreachable
```

**Cause**:
```
ICMP Echo Request  (outbound) ✅ → Allowed
ICMP Echo Reply    (inbound)  ❌ → Blocked by Network ACL
```

**Solution**:
```hcl
# Allow ICMP responses
ingress {
  protocol   = "1"  # ICMP
  rule_no    = 80
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 0
  to_port    = 0
  icmp_type  = -1
  icmp_code  = -1
}
```

### Issue 2: HTTPS connection hangs

**Symptoms**:
```bash
$ curl https://google.com
# Hangs and eventually times out
```

**Cause**:
- TCP SYN can be sent out
- TCP SYN-ACK response is blocked
- Cannot complete three-way handshake

**Solution**:
```hcl
# Allow ephemeral port responses
ingress {
  protocol   = "6"
  rule_no    = 90
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 1024
  to_port    = 65535
}
```

### Issue 3: Internal services cannot access each other

**Symptoms**:
Services within the same VPC cannot connect to each other

**Solution**:
```hcl
# Allow VPC internal communication
ingress {
  protocol   = "-1"  # All protocols
  rule_no    = 110
  action     = "allow"
  cidr_block = var.vpc_cidr  # e.g., 10.0.0.0/16
  from_port  = 0
  to_port    = 0
}

egress {
  protocol   = "-1"
  rule_no    = 120
  action     = "allow"
  cidr_block = var.vpc_cidr
  from_port  = 0
  to_port    = 0
}
```

## Best Practice Recommendations

### Network ACL Design Principles

1. **Prioritize bidirectional communication**
   - Every outbound rule should have a corresponding inbound rule
   - Pay special attention to ephemeral port ranges

2. **Use lower rule numbers for important rules**
   - Network ACL evaluates rules in rule number order
   - Lower numbers are evaluated first

3. **Explicitly deny unnecessary traffic**
   - Use explicit deny rules
   - Avoid relying on default deny all

### Security Group Design Principles

1. **Principle of least privilege**
   - Only open necessary ports and sources
   - Regularly review and clean up unnecessary rules

2. **Use descriptive rule names**
   - Clearly indicate the purpose of each rule
   - Facilitate future maintenance and debugging

## Memory Aids

### Network ACL (Stateless)
> **"Amnesia Patient"**
> 
> "I don't remember what you just said, every packet needs to be checked against all rules!
>  You want to come in? Show me your pass! You want to go out? Show me your pass too!"

### Security Group (Stateful)
> **"Thoughtful Butler"**
> 
> "I remember your previous request, since you've already been authenticated to go out,
>  your response can come directly in, no need to check again!"

## Summary

Understanding the difference between **Stateless** vs **Stateful** is key to mastering AWS network security:

- **Network ACL** is like a strict border checkpoint, everyone coming and going must show credentials
- **Security Group** is like an intelligent access control system, remembering who went out and automatically letting them back in

When you encounter "can send out but can't receive responses" issues, nine times out of ten it's caused by Network ACL's stateless characteristics! 