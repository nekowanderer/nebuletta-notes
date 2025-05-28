# Network ACL Impact on Private Subnet Communication - Debugging Experience

[English](01_network_acl_private_subnet_troubleshooting.md) | [繁體中文](../zh-tw/01_network_acl_private_subnet_troubleshooting.md) | [日本語](../ja/01_network_acl_private_subnet_troubleshooting.md) | [Back to Index](../README.md)

---

## Context
- Experiment Date: 2025/05/21
- Difficulty: 🤬🤬🤬
- Description: In an AWS VPC environment, EC2 instances in private subnets cannot properly communicate with external networks, even though all seemingly related configurations are correct.

## Observed Symptoms

- EC2 can be accessed normally through Session Manager
- `ping www.google.com` returns no response
- `curl https://google.com` hangs and cannot complete the connection

## Debugging Process

### Phase 1: Confirming Infrastructure Configuration

Verified that all the following items are configured correctly:

| Item | Status | Description |
|------|--------|-------------|
| EC2 Instance | ✅ | No Public IP, correctly located in Private Subnet |
| Private Subnet | ✅ | Has corresponding Route Table |
| Route Table | ✅ | Has `0.0.0.0/0` ➝ NAT Gateway configured |
| NAT Gateway | ✅ | Deployed in Public Subnet with Elastic IP |
| Security Group | ✅ | Outbound set to `0.0.0.0/0` all traffic |

### Phase 2: DNS Resolution Testing

Running `nslookup google.com` returned normal results:

```bash
Server:         10.0.0.2
Address:        10.0.0.2#53

Non-authoritative answer:
Name:   google.com
Address: 142.251.42.142
Name:   google.com
Address: 2404:6800:4004:827::200e
```

**Conclusion**: DNS resolution is normal, indicating that VPC DNS configuration and `/etc/resolv.conf` are working properly.

### Phase 3: Eliminating Possible Causes

Since DNS is normal but HTTPS connections fail, possible causes include:

1. ❌ NAT Gateway not connected to Internet Gateway
2. ❌ Public Subnet Route Table for NAT Gateway doesn't have `0.0.0.0/0` ➝ IGW
3. ✅ **NACL (Network ACL) blocking connections** ← The real problem!
4. ❌ EC2 Source/Dest Check not disabled (only applies to NAT Instance)

## Root Cause

### Network ACL Configuration Error

Checking Network ACL rules bound to the private subnet:

**Inbound rules (problematic):**
| Rule # | Source | Protocol | Port | Allow/Deny |
|--------|--------|----------|------|------------|
| 100 | 10.0.0.0/16 | All | All | ✅ Allow |
| * | 0.0.0.0/0 | All | All | ❌ **Deny** ← Problem here!|

**Outbound rules (seemingly normal but with pitfalls):**
| Rule # | Destination | Protocol | Port | Allow/Deny |
|--------|-------------|----------|------|------------|
| 100 | 0.0.0.0/0 | All | All | ✅ Allow |
| * | 0.0.0.0/0 | All | All | ❌ **Deny** ← Still takes effect!|

## Problem Analysis

### ICMP (Ping) Workflow

1. **Outbound ICMP Request**:
   - EC2 sends ping packet to external address
   - Matches Outbound Rule 100: `0.0.0.0/0 Allow`
   - Packet can be sent successfully

2. **Inbound ICMP Reply**:
   - External server returns ping response
   - Source IP is not VPC CIDR (`10.0.0.0/16`)
   - Blocked by Inbound Rule `* 0.0.0.0/0 Deny`
   - **Response packet cannot enter, causing ping failure**

### Impact on Other Protocols

The same problem also affects:
- HTTPS connections (port 443)
- HTTP connections (port 80)  
- Any communication requiring external responses

## Solution

### Required Additional Inbound Rules

To allow EC2 instances in private subnets to communicate normally with external networks, the following inbound rules must be added to the Network ACL:

1. **Allow ICMP responses**:
   ```hcl
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

2. **Allow Ephemeral Port response traffic**:
   ```hcl
   ingress {
     protocol   = "6"  # TCP
     rule_no    = 90
     action     = "allow"
     cidr_block = "0.0.0.0/0"
     from_port  = 1024
     to_port    = 65535
   }
   ```

3. **Allow HTTPS responses**:
   ```hcl
   ingress {
     protocol   = "6"  # TCP
     rule_no    = 100
     action     = "allow" 
     cidr_block = "0.0.0.0/0"
     from_port  = 443
     to_port    = 443
   }
   ```

## Key Learning Points

### Network ACL vs Security Group

| Feature | Network ACL | Security Group |
|---------|-------------|----------------|
| Operation Level | Subnet level | Instance level |
| State | **Stateless** | Stateful |
| Response Handling | Need to explicitly allow response traffic | Automatically allows response traffic |
| Default Behavior | Default deny all | Default deny inbound, allow outbound |

### Importance of Stateless

- **Stateless** means Network ACL does not remember connection states
- Outbound allow does not mean Inbound responses will also be allowed
- **Must explicitly configure bidirectional rules**

### Importance of Ephemeral Ports

- When TCP connections are established, clients use random high port numbers (1024-65535)
- Server responses are sent to these high ports
- If inbound doesn't allow these ports, connections will fail

## Prevention Measures

1. **Prioritize bidirectional communication when designing Network ACL**
2. **Don't ignore ephemeral port ranges**
3. **Test both outbound and inbound traffic during testing**
4. **Remember Network ACL's stateless characteristics**

## Conclusion

This debugging experience reminds us that in AWS network configuration, Network ACL is an easily overlooked but highly impactful component. Especially for private subnet configuration, all possible response traffic must be carefully considered to ensure services can operate normally.

**Remember**: Being able to ping out doesn't mean you can receive responses - stateless Network ACL requires explicit bidirectional rule configuration! 