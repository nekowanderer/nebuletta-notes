# Dynamic AZ and CIDR Calculation Guide

[English](../en/05_calculate_az_and_cidr_dynamically.md) | [繁體中文](../zh-tw/05_calculate_az_and_cidr_dynamically.md) | [日本語](../ja/05_calculate_az_and_cidr_dynamically.md) | [Back to Index](../README.md)

This document explains how to implement dynamic Availability Zone (AZ) selection and automatic CIDR block calculation in the Nebuletta project.

---

## Background
- Experiment Date: 2025/07/21
- Difficulty: 🤬
- Description: The initial implementation relied heavily on manual calculation and configuration, which was not an ideal solution.

---

### Original Hard-Coded Approach
```hcl
# Old implementation method
azs = ["ap-northeast-1a", "ap-northeast-1c"]
public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]
private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24"]
```

**Problems:**
- Cannot be used across regions (each region has different AZs)
- Requires manual maintenance of AZ lists
- Requires manual calculation of CIDR blocks
- Prone to configuration errors

---

## Solution: Dynamic Calculation

### 1. Automatic AZ Detection

```hcl
# data.tf
data "aws_availability_zones" "available" {
  state = "available"
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}

# locals.tf
available_azs = slice(
  data.aws_availability_zones.available.names, 
  0, 
  min(var.max_azs, length(data.aws_availability_zones.available.names))
)
```

**Key Concepts:**
- `state = "available"` - Only select available AZs
- `opt-in-status = "opt-in-not-required"` - Only select AZs that don't require additional opt-in
- `min()` function ensures we don't exceed the actual number of available AZs

### 2. Dynamic CIDR Calculation

#### Step 1: VPC CIDR Decomposition
```hcl
vpc_cidr_prefix = split("/", var.vpc_cidr)[0]  # "10.0.0.0"
vpc_cidr_mask   = tonumber(split("/", var.vpc_cidr)[1])  # 16
```

#### Step 2: IP Address Segmentation
```hcl
# Split "10.0.0.0" into ["10", "0", "0", "0"]
split(".", local.vpc_cidr_prefix)

# Take first two segments ["10", "0"]
slice(split(".", local.vpc_cidr_prefix), 0, 2)

# Reassemble to "10.0"
join(".", slice(split(".", local.vpc_cidr_prefix), 0, 2))
```

#### Step 3: Subnet CIDR Generation
```hcl
# Public subnets: 10.x.1.0/24, 10.x.2.0/24, ...
public_subnet_cidrs = [
  for i in range(length(local.available_azs)) :
  "${join(".", slice(split(".", local.vpc_cidr_prefix), 0, 2))}.${i + 1}.0/24"
]

# Private subnets: 10.x.11.0/24, 10.x.12.0/24, ...
private_subnet_cidrs = [
  for i in range(length(local.available_azs)) :
  "${join(".", slice(split(".", local.vpc_cidr_prefix), 0, 2))}.${i + 11}.0/24"
]
```

---

## Function Detailed Explanation

### Terraform Function Reference

| Function | Purpose | Example |
|----------|---------|---------|
| `slice(list, start, end)` | Extract a portion from a list | `slice(["a","b","c"], 0, 2)` → `["a","b"]` |
| `split(separator, string)` | String splitting | `split("/", "10.0.0.0/16")` → `["10.0.0.0", "16"]` |
| `join(separator, list)` | List concatenation | `join(".", ["10","0"])` → `"10.0"` |
| `tonumber(string)` | String to number conversion | `tonumber("16")` → `16` |
| `min(a, b)` | Get smaller value | `min(4, 3)` → `3` |
| `length(list)` | Get list length | `length(["a","b"])` → `2` |
| `range(n)` | Generate number sequence | `range(3)` → `[0, 1, 2]` |

### Complete Calculation Example

Given:
- `var.vpc_cidr = "10.0.0.0/16"`
- `var.max_azs = 3`
- Available AZs in ap-northeast-1: `["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]`

**Calculation Process:**

1. **AZ Selection:**
   ```hcl
   available_azs = ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
   ```

2. **CIDR Decomposition:**
   ```hcl
   vpc_cidr_prefix = "10.0.0.0"
   join(".", slice(split(".", "10.0.0.0"), 0, 2)) = "10.0"
   ```

3. **Public Subnet Calculation:**
   ```hcl
   i=0: "10.0.1.0/24"   # ap-northeast-1a
   i=1: "10.0.2.0/24"   # ap-northeast-1c  
   i=2: "10.0.3.0/24"   # ap-northeast-1d
   ```

4. **Private Subnet Calculation:**
   ```hcl
   i=0: "10.0.11.0/24"  # ap-northeast-1a
   i=1: "10.0.12.0/24"  # ap-northeast-1c
   i=2: "10.0.13.0/24"  # ap-northeast-1d
   ```

---

## Benefits Summary

### ✅ Problems Solved
1. **Cross-Region Compatibility** - Automatically adapts to any AWS Region's AZ configuration
2. **Reduced Configuration Complexity** - Only need to specify the `max_azs` parameter
3. **Reduced Human Errors** - Automatic CIDR calculation prevents conflicts
4. **Flexible Control** - Can adjust AZ count based on requirements
5. **Fault Tolerance** - Automatically handles cases with insufficient AZs

### ✅ Practical Benefits
- **Improved Maintainability** - Reduced hard-coded configurations
- **Deployment Flexibility** - Same codebase can deploy to different regions
- **Scalability** - Easy to adjust AZ count
- **Consistency** - Unified CIDR allocation rules

---

## Special Cases

### Actual Situation in ap-northeast-1
- Theoretically has 4 AZs (a, b, c, d)
- Actually only 3 are available (ap-northeast-1b is deprecated)
- System automatically detects and uses: `["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]`

### CIDR Allocation Rules
- **VPC CIDR**: `10.0.0.0/16` (65536 IPs)
- **Public Subnet**: `10.0.1.0/24`, `10.0.2.0/24`, ... (256 IPs each)
- **Private Subnet**: `10.0.11.0/24`, `10.0.12.0/24`, ... (256 IPs each)
- **Reserved Space**: `10.0.0.0/24`, `10.0.4-10.0/24`, `10.0.14+.0/24` available for other purposes

This design ensures flexibility and maintainability of the network architecture.