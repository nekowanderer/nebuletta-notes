# CIDR Planning and Calculation Methods

[English](../en/03_how_to_arrange_cidr.md) | [繁體中文](../zh-tw/03_how_to_arrange_cidr.md) | [日本語](../ja/03_how_to_arrange_cidr.md) | [Back to Index](../README.md)

## CIDR Basic Concepts

CIDR (Classless Inter-Domain Routing) is a method for IP address allocation and routing, used for more efficient management of network resources.

### Notation Explanation

```
10.0.0.0/16
    ↑     ↑
   IP   Prefix Length
```

**Prefix Length** indicates the number of bits for the network portion:
- `/16` = First 16 bits are the network portion
- `/24` = First 24 bits are the network portion
- `/32` = Host address (single IP)

---

## Bit Organization and Calculation

### 1. IP Address Structure
```
IP Address: 32 bits = 4 bytes

Example: 10.0.0.0
Binary: 00001010.00000000.00000000.00000000
          ↑        ↑        ↑        ↑
        1st byte  2nd byte  3rd byte  4th byte
```

### 2. Network and Host Portion Division
```
Division of 10.0.0.0/16:

Network portion: First 16 bits (00001010.00000000)
Host portion: Last 16 bits (.00000000.00000000)

Variable range: 10.0.0.0 ~ 10.0.255.255
```

---

## Available IP Count Calculation

### Calculation Formula
**Available IP Count = 2^(32-Prefix Length) - 2**

> **Why subtract 2?**
> - First IP: Network address (unusable)
> - Last IP: Broadcast address (unusable)

### Common CIDR Capacity Table

| CIDR | Network Bits | Host Bits | Total IPs | Available IPs | Use Case |
|------|--------------|-----------|-----------|---------------|----------|
| /16  | 16           | 16        | 65,536    | 65,534        | Large networks |
| /24  | 24           | 8         | 256       | 254           | General subnets |
| /25  | 25           | 7         | 128       | 126           | Small subnets |
| /26  | 26           | 6         | 64        | 62            | Micro subnets |
| /27  | 27           | 5         | 32        | 30            | Tiny subnets |
| /28  | 28           | 4         | 16        | 14            | Device groups |

### Practical Calculation Examples

#### Example 1: VPC Planning
```
VPC: 10.0.0.0/16

Calculation process:
Host bits: 32 - 16 = 16 bits
Available IPs: 2^16 - 2 = 65,534
IP range: 10.0.0.1 ~ 10.0.255.254
```

#### Example 2: Subnet Planning
```
Subnet: 10.0.1.0/24

Calculation process:
Host bits: 32 - 24 = 8 bits
Available IPs: 2^8 - 2 = 254
IP range: 10.0.1.1 ~ 10.0.1.254
```

---

## Subnet Division Calculation

### Dividing Large Networks into Small Networks

#### Example: Dividing /16 into /24
```
Original network: 10.0.0.0/16

Number of /24 subnets that can be created:
2^(24-16) = 2^8 = 256

Subnet list:
10.0.0.0/24    → 10.0.0.1   ~ 10.0.0.254   (254 IPs)
10.0.1.0/24    → 10.0.1.1   ~ 10.0.1.254   (254 IPs)
10.0.2.0/24    → 10.0.2.1   ~ 10.0.2.254   (254 IPs)
...
10.0.255.0/24  → 10.0.255.1 ~ 10.0.255.254 (254 IPs)
```

### Subnet Boundary Checking

Check if an IP is a valid subnet starting address:

```bash
# Check if 10.0.11.0/24 is valid

Step 1: Analyze IP address structure
10.0.11.0/24
- Network portion: First 24 bits (10.0.11)
- Host portion: Last 8 bits (0)

Step 2: Check if host portion is 0
4th byte = 0 = 00000000
Host portion is all zeros ✓

# Conclusion: 10.0.11.0/24 is a valid subnet starting address
```

#### Another Example: Checking Invalid Cases
```bash
# Check if 10.0.11.5/24 is valid

Step 1: Analyze IP address structure  
10.0.11.5/24
- Network portion: First 24 bits (10.0.11)
- Host portion: Last 8 bits (5)

Step 2: Check if host portion is 0
4th byte = 5 = 00000101
Host portion is not zero ✗

# Conclusion: 10.0.11.5/24 is not a valid subnet starting address
# The correct subnet should be: 10.0.11.0/24
```

#### More Complex Example: Checking /26 Subnets
```bash
# Check if 10.0.1.64/26 is valid

Step 1: Analyze /26 structure
/26 = 26-bit network portion + 6-bit host portion
Host portion: Last 6 bits must be 0

Step 2: Check the 4th byte
64 = 01000000 (binary)
Last 6 bits = 000000 ✓ (all zeros)

# Conclusion: 10.0.1.64/26 is a valid subnet

# Valid /26 subnet boundaries:
# 10.0.1.0/26   (0   = 00000000)
# 10.0.1.64/26  (64  = 01000000) 
# 10.0.1.128/26 (128 = 10000000)
# 10.0.1.192/26 (192 = 11000000)
```

---

## Multi-Environment CIDR Planning Practice

### Environment Isolation Strategy

Recommended multi-environment planning:

```
Dev Environment:    10.0.x.x/16  (10.0.0.0   ~ 10.0.255.255)
Staging Environment: 10.1.x.x/16  (10.1.0.0   ~ 10.1.255.255)
Production:         10.2.x.x/16  (10.2.0.0   ~ 10.2.255.255)
Testing:            10.3.x.x/16  (10.3.0.0   ~ 10.3.255.255)
```

### Specific Planning Example

#### Dev Environment: 10.0.0.0/16
```
VPC Total Capacity: 65,534 IPs

Public Subnets:
├── AZ-a: 10.0.1.0/24  → 254 IPs
└── AZ-b: 10.0.2.0/24  → 254 IPs

Private Subnets:
├── AZ-a: 10.0.11.0/24 → 254 IPs  
└── AZ-b: 10.0.12.0/24 → 254 IPs

Database Subnets:
├── AZ-a: 10.0.21.0/24 → 254 IPs
└── AZ-b: 10.0.22.0/24 → 254 IPs

Total Usage: 6 × 254 = 1,524 IPs
Utilization: 1,524 / 65,534 = 2.3%
```

#### Complete Environment Planning
```
🏗️  Dev Environment (10.0.x.x/16)
├── Public:   10.0.1.0/24,  10.0.2.0/24
├── Private:  10.0.11.0/24, 10.0.12.0/24
└── Database: 10.0.21.0/24, 10.0.22.0/24

🚀 Staging Environment (10.1.x.x/16)
├── Public:   10.1.1.0/24,  10.1.2.0/24
├── Private:  10.1.11.0/24, 10.1.12.0/24
└── Database: 10.1.21.0/24, 10.1.22.0/24

🏭 Production Environment (10.2.x.x/16)
├── Public:   10.2.1.0/24,  10.2.2.0/24
├── Private:  10.2.11.0/24, 10.2.12.0/24
└── Database: 10.2.21.0/24, 10.2.22.0/24
```

---

## Planning Advantages and Considerations

### Advantages of This Planning

1. **Avoid IP Conflicts**
   - Different environments use different /16 blocks
   - No conflicts when connecting via VPC Peering or Transit Gateway

2. **Maintain Consistent Patterns**
   - All environments use the same subnet numbering rules
   - Easy to remember and maintain

3. **Easy Environment Identification**
   ```
   Seeing 10.0.x.x → Know it's Dev environment
   Seeing 10.1.x.x → Know it's Staging environment
   Seeing 10.2.x.x → Know it's Production environment
   ```

4. **Sufficient Expansion Space**
   - Each environment has 65,534 available IPs
   - Can add more subnets without re-planning

### Considerations

1. **Reserve Sufficient Space**
   ```bash
   # Avoid over-segmentation
   ❌ Bad: Use up all /24 subnets
   ✅ Good: Only use needed subnets, reserve space for future
   ```

2. **Consider Routing Requirements**
   - Communication needs between different subnets
   - Route table complexity management

3. **Security Group Planning**
   - Set security rules based on CIDR ranges
   - Principle of least privilege

---

## Practical Tools and Tips

### Quick Calculation Tips

```bash
# Remember common capacities
/16 = 65K IPs   (suitable for VPC)
/24 = 256 IPs   (suitable for general subnets)
/26 = 64 IPs    (suitable for small microservices)
/28 = 16 IPs    (suitable for database clusters)

# CIDR calculator commands
$ ipcalc 10.0.0.0/16
$ sipcalc 10.0.1.0/24
```

### Checking Tools
```python
# Python example
import ipaddress

network = ipaddress.IPv4Network('10.0.0.0/16')
print(f"Network address: {network.network_address}")
print(f"Broadcast address: {network.broadcast_address}")
print(f"Available IPs: {network.num_addresses - 2}")

# Check subnet
subnet = ipaddress.IPv4Network('10.0.1.0/24')
print(f"Is subnet of: {subnet.subnet_of(network)}")
```

---

## Summary

Good CIDR planning enables:

1. **Improved Operational Efficiency**: Clear naming conventions and structure
2. **Avoid Future Conflicts**: Sufficient address space reservation
3. **Simplified Network Management**: Consistent architectural patterns
4. **Support Environment Expansion**: Flexible growth space

Remember: **Plan for current needs, but leave space for the future!**