# Route 53 vs Global Accelerator: Properties and Differences

[English](02_route53_vs_global_accelerator.md) | [繁體中文](../zh-tw/02_route53_vs_global_accelerator.md) | [日本語](../ja/02_route53_vs_global_accelerator.md) | [Back to Index](../README.md)


## Background

The new project uses Keycloak, with separate admin/public nodes:
- Admin node: Limited to company intranet, not directly accessible via Internet
- Public node: Directly accessible via Internet, but typically accessed through the company's frontend system
- Keycloak must implement a cross-region architecture design


## Solution

#### ELB
- ELB is a Region-level resource
- Whether it's Classic ELB, Application Load Balancer (ALB), or Network Load Balancer (NLB), they can only exist in a single AWS Region. This means:
- You cannot use the same ELB to distribute traffic to EC2 instances in both Tokyo (ap-northeast-1) and Singapore (ap-southeast-1) regions.
- Each Region needs its own ELB.

#### Route 53
- Can define ELB DNS for multiple Regions in Route 53
- Utilize Route 53's Latency-based routing or Geolocation routing
- Add health checks for automatic failover

#### Global Accelerator
- A global service that provides you with an anycast IP
- Automatically routes traffic to the nearest, healthy ELB
- More suitable for scenarios requiring low latency and high availability


## About Route 53

Route 53 is a global DNS service, not a Region-bound component. Here are its properties and classifications:

| Classification Angle | Description                                |
| ------------------- | ------------------------------------------ |
| **Service Type**    | DNS Service (Domain Name System)           |
| **Scope Level**     | Global (Cross-Region, single service covers worldwide) |
| **Usage Category**  | Domain name resolution, traffic routing, health checks, high availability, fault tolerance |
| **Operation Mode**  | Based on Authoritative DNS                 |
| **Non-Region Bound**| Hosted Zone and record sets don't belong to any specific AWS Region |

#### What features does it provide?
- DNS Hosting Service (Hosted Zone)
  - Hosts DNS records (A, AAAA, CNAME, MX, TXT, etc.) for your domain
- Traffic Routing Policies
  - Simple Routing (single IP/service)
  - Latency-based Routing (based on lowest latency)
  - Geolocation Routing (based on user location)
  - Weighted Routing (weighted distribution)
  - Failover Routing (primary-backup switching)
- Health Checks
  - Can set up periodic checks for ALB, EC2, or any endpoint, automatically removing unhealthy targets
- Integration with AWS Global Accelerator or ELB
  - Provides stable global network access points

#### Key Points
| Feature              | Description                                |
| ------------------- | ------------------------------------------ |
| **Global Resource** | Not bound to any single AWS Region         |
| **Cross-Region Routing** | Can implement cross-region failover with health checks |
| **ELB Integration** | Can directly resolve ELB DNS names         |
| **CloudFront Integration** | Can serve as CDN source routing layer      |



## About Global Accelerator

Amazon Global Accelerator and Route 53 are both global-level services, but they have different properties and distinct purposes.

| Classification Angle | Description                                |
| ------------------- | ------------------------------------------ |
| **Service Type**    | Global Application Acceleration Service (Anycast IP + Smart Traffic Routing) |
| **Scope Level**     | Global (Global network layer)              |
| **Usage Category**  | Traffic optimization, high availability, automatic failover, cross-region speed improvement |
| **Operation Level** | TCP/UDP layer, L4 network packet routing   |
| **Non-Region Bound**| Global service itself, but targets connect to specific Region resources (like ALB) |

#### What features does it provide?
- Provides fixed Anycast IPs
  - You get fixed public IPv4 addresses after deploying Global Accelerator
  - Users worldwide automatically route to the nearest AWS edge location
- Routes traffic to nearest Region
  - Through AWS global backbone network (faster and more stable than public Internet)
  - Routes to most suitable backend based on user distance, health status, and priority
- Supports health checks and automatic failover
  - If one Region has issues, traffic automatically switches to another healthy target
- Supports ALB, NLB, EC2 as backends
  - Targets can be Load Balancers in multiple different Regions

#### Suitable Scenarios
- Applications requiring low latency and real-time routing (like online games, voice chat, financial services)
- Global users accessing the same service, expecting stable and fast network experience
- Wanting a fixed set of IPs without exposing Load Balancer DNS names
- Wanting to deploy applications in multiple Regions with automatic latency-based routing


## About Costs

Global Accelerator is significantly more expensive than Route 53, typically only suitable for latency-sensitive services or high-availability services with global users.

#### 💰 Route 53 Cost Concept (Cheap)
| Item          | Cost (USD)                    |
| ------------- | ----------------------------- |
| Hosted Zone   | \$0.50/month/domain           |
| DNS Queries   | ~\$0.40-\$0.60/million queries |
| Health Checks | ~\$0.50/endpoint/month        |
- Even for medium-sized websites, Route 53 might only cost a few dollars per month.

#### 💰 Global Accelerator Cost Concept (Expensive)
| Item                    | Cost (USD)                                    |
| ---------------------- | --------------------------------------------- |
| Base Fee (per Accelerator) | \$0.025/hour ≈ \$18/month                    |
| Accelerated IPs (each) | \$0.025/hour ≈ \$18/month × 2 IPs = \$36     |
| Traffic Fee (in/out)   | Varies by source and destination (about \$0.015-\$0.12/GB) |
- Running a global web application with GA might cost \$50-\$70 per month just for keeping it running, with additional costs for traffic (per GB)


## Comparison

| Feature        | Route 53                | Global Accelerator     |
| ------------- | ----------------------- | ---------------------- |
| **Main Function** | DNS resolution and traffic routing | TCP/UDP traffic acceleration and global routing |
| **Suitable Traffic Types** | Various services (Web, Email, API) | Latency-sensitive, long connections, game servers, real-time interactive services |
| **IP Stability** | Depends on DNS records (not fixed) | Provides fixed Anycast IPs (IPv4) |
| **Switch Time** | May have DNS TTL delay (tens of seconds to minutes) | Almost instant (seconds) |
| **Health Check & Fault Tolerance** | Yes, but depends on DNS level | Built-in automatic health checks and real-time failover |
| **Routing Logic** | DNS record routing, multiple strategies available | Automatic routing to nearest, healthiest endpoint |
| **Cost** | Relatively cheap (based on DNS queries and monitoring) | Relatively expensive (based on traffic, accelerated IPs, etc.) |
| **Analogy** | Smart DNS routing | International network VIP dedicated line + router |


## References

- [Amazon Route 53 Pricing](https://aws.amazon.com/route53/pricing/)
- [AWS Global Accelerator Pricing](https://aws.amazon.com/global-accelerator/pricing/) 
