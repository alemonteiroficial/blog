---
layout: post
title: "Amazon VPC: networking, subnets, and connectivity on AWS"
description: "A technical SAA-C03 study guide covering CIDR, public and private subnets, route tables, internet and NAT gateways, traffic controls, and private connectivity."
date: 2026-11-12
lang: en
category: AWS · Networking
permalink: /en/aws-vpc-fundamentals-saa-c03/
---

## 1. What is an Amazon VPC?

An **Amazon Virtual Private Cloud (VPC)** is a logically isolated virtual network in AWS. It lets you define an IP address space, create subnets, configure routes, and control how resources communicate with one another, the internet, and external networks.

For the **AWS Certified Solutions Architect – Associate (SAA-C03)** exam, memorizing definitions is not enough. You need to identify which networking component satisfies a requirement and which conditions must be true for traffic to follow the expected path.

> **Mental model:** the VPC defines the network environment; a subnet allocates an IP range within one Availability Zone; a route table determines where traffic goes; security groups and network ACLs control what can pass.

## 2. Scope: Regions, Availability Zones, and subnets

A VPC belongs to **one AWS Region** and can span multiple Availability Zones (AZs). A subnet, however, belongs to **exactly one AZ**.

This matters for resilient architectures: a VPC can contain subnets in different AZs, allowing workloads to be distributed and reducing the impact of a localized failure.

Each subnet receives a CIDR block contained within the VPC CIDR. Subnet ranges within a VPC must not overlap. Plan address allocation for growth, on-premises connectivity, and potential connections to other VPCs.

![A VPC distributed across two Availability Zones]({{ '/assets/images/aws-vpc-two-az.svg' | relative_url }})
<small>Figure 1 — A regional VPC can contain subnets in multiple AZs. Each subnet belongs to a single AZ.</small>

### CIDR and reserved addresses

Example: a VPC with CIDR `10.0.0.0/16` can be divided into smaller subnets such as `10.0.1.0/24` and `10.0.2.0/24`.

AWS reserves **five IPv4 addresses in every subnet**. For a subnet such as `10.0.1.0/24`:

- `10.0.1.0`: network address.
- `10.0.1.1`: reserved for the VPC router.
- `10.0.1.2`: reserved for the VPC DNS resolver.
- `10.0.1.3`: reserved for future AWS use.
- `10.0.1.255`: reserved as the broadcast address (broadcast is not supported in VPCs).

Therefore, an IPv4 `/24` subnet has 256 addresses in its block, but **251 are usable by resources**.

## 3. Public versus private subnets

The distinction is not the subnet's name, nor simply whether an instance has a public IP. It is primarily determined by the **route table associated with the subnet**.

- **Public subnet:** has a route to an Internet Gateway (IGW), commonly `0.0.0.0/0 → igw-...`.
- **Private subnet:** has no direct route to an IGW. It may have outbound internet access through a NAT Gateway, or no internet access at all.

An instance in a public subnet still needs a public IPv4 address or Elastic IP, plus compatible security group and network ACL rules, to establish direct IPv4 internet communication.

![VPC public and private subnets across two Availability Zones]({{ '/assets/images/aws-vpc-two-az.svg' | relative_url }})
<small>Figure 1 — A VPC can span AZs, while each subnet remains scoped to one AZ.</small>

## 4. Internet Gateway (IGW)

An **Internet Gateway** enables communication between a VPC and the internet. It is highly available and scalable, but merely attaching an IGW does not make a subnet public.

For an instance to initiate IPv4 internet communication, check all of the following:

1. The IGW is attached to the VPC.
2. The subnet's route table has a route to the IGW, typically `0.0.0.0/0`.
3. The instance has a public IPv4 address or Elastic IP.
4. Security groups, network ACLs, and the operating system firewall allow the required traffic.

**Exam trap:** creating or attaching an IGW without adding the corresponding route does not provide internet connectivity.

## 5. NAT Gateway: outbound internet access without direct exposure

A **NAT Gateway** allows resources in private subnets to initiate connections to external destinations without allowing those destinations to initiate unsolicited connections back to private resources.

In the common internet egress pattern:

- The NAT Gateway is created in a **public subnet**.
- It uses an Elastic IP for internet communication.
- The NAT Gateway's public subnet has a route to the IGW.
- The private subnet's route table points its default route to the NAT Gateway: `0.0.0.0/0 → nat-...`.

![Traffic flow from a private subnet through a NAT Gateway]({{ '/assets/images/aws-vpc-nat-routing.svg' | relative_url }})
<small>Figure 2 — Private instances initiate outbound connections through the NAT Gateway. It is not a way to publish private services to the internet.</small>

### NAT Gateway details

- NAT Gateway is managed; unlike a NAT instance, you do not administer an EC2 instance for the translation function.
- A public NAT Gateway is created in a public subnet and uses an Elastic IP.
- For high availability and to avoid a single-AZ dependency, a common best practice is one NAT Gateway per AZ, with each private subnet routed to the NAT Gateway in the same AZ.
- A NAT Gateway does not accept unsolicited connections initiated from the internet to private resources.
- NAT Gateway is charged per hour and by data processed; also consider data transfer charges.

**NAT Gateway is not an Internet Gateway:** the IGW connects the VPC to the internet; the NAT Gateway enables private resources to initiate outbound connections using a shared public address.

## 6. Route tables and routing decisions

Every subnet must be associated with a route table. If no explicit association is configured, the subnet uses the VPC's **main route table**.

A route table contains destinations (CIDR prefixes or prefix lists) and targets, such as:

| Destination | Target | Purpose |
|---|---|---|
| `10.0.0.0/16` | `local` | Communication within the VPC |
| `0.0.0.0/0` | Internet Gateway | Default IPv4 internet route |
| `0.0.0.0/0` | NAT Gateway | Private subnet IPv4 egress |
| S3 prefix list | Gateway Endpoint | Private access to S3 |

AWS uses **longest prefix match**: among matching routes, the most specific prefix wins. For example, a route for `10.0.0.0/16` can be selected instead of the default `0.0.0.0/0` route.

The `local` route enables network communication between resources using the VPC CIDR ranges. It does not mean security groups and network ACLs are bypassed.

## 7. Security Groups versus Network ACLs

This comparison is a frequent SAA-C03 topic.

| Feature | Security Group (SG) | Network ACL (NACL) |
|---|---|---|
| Scope | Network interface / associated resources | Subnet |
| State | Stateful | Stateless |
| Rules | Allow | Allow and deny |
| Evaluation | All applicable rules | Lowest rule number first; first match |
| Return traffic | Automatically allowed by connection state | Must be explicitly allowed |

### Security Groups

Security groups are **stateful**. If inbound traffic is allowed, response traffic is automatically allowed, even if there is no specific outbound rule for that response.

They support allow rules, not explicit deny rules. In supported scenarios, another security group can be referenced as a source or destination, making policies such as “the database accepts traffic only from the application security group” easier to maintain.

### Network ACLs

Network ACLs are **stateless** and operate at the subnet level. Both inbound and outbound traffic must be allowed, including return traffic. Rules are evaluated in ascending numerical order, and the first matching rule determines the decision.

This makes NACLs useful for broad subnet-level controls, including explicit IP-range blocks. When troubleshooting a failed connection, remember the ephemeral ports used by TCP responses.

> **Exam shortcut:** SG = stateful, associated with network interfaces/resources, allow only. NACL = stateless, associated with subnets, allow and deny, ordered evaluation.

## 8. VPC Endpoints: reach AWS services without NAT/IGW

VPC Endpoints let a VPC connect to AWS services privately, without requiring traffic to traverse the public internet.

The two types most likely to appear in the exam:

**Gateway Endpoint**
- Available for Amazon S3 and DynamoDB.
- Associated with selected route tables.
- Does not use an ENI or a private IP address inside the subnet.
- No additional endpoint usage charge; normal service and applicable data transfer charges still apply.

**Interface Endpoint (AWS PrivateLink)**
- Creates one or more Elastic Network Interfaces (ENIs) in selected subnets.
- Uses private IP addresses and security groups.
- Supports many AWS services and services published by partners or other accounts.
- Has hourly and data-processing charges.

**Classic scenario:** an application in a private subnet needs S3 access without using a NAT Gateway. An S3 Gateway Endpoint can meet the requirement, provided the route table and endpoint/bucket policies allow the request.

## 9. Connectivity between VPCs and on-premises environments

Know what each option is designed for:

- **VPC Peering:** private connectivity between two VPCs. It is non-transitive; one VPC cannot use another VPC as a router to reach a third VPC through peering.
- **AWS Transit Gateway:** a central hub connecting multiple VPCs and on-premises networks, supporting hub-and-spoke architectures.
- **Site-to-Site VPN:** an encrypted connection over the internet between an on-premises network and AWS.
- **AWS Direct Connect:** a dedicated network connection between an on-premises environment and AWS; it does not provide encryption by itself. VPN can be combined with it when encryption is required.

Choose based on scale, topology, latency, redundancy, cost, and the need for transitive connectivity.

## 10. VPC Flow Logs and troubleshooting

**VPC Flow Logs** capture information about IP traffic going to and from network interfaces, subnets, or VPCs. Depending on configuration, they can publish to CloudWatch Logs, Amazon S3, or Amazon Data Firehose.

Use Flow Logs to investigate traffic patterns and identify accepted or rejected records. They are not packet captures: they do not record the full contents of network communications.

For connectivity troubleshooting, follow this sequence:

1. Confirm source and destination CIDRs, IP addresses, and subnets.
2. Check the route table associated with each subnet.
3. Verify the IGW, NAT Gateway, endpoint, peering, or Transit Gateway expected in the path.
4. Review Security Group rules.
5. Review inbound and outbound NACL rules, including ephemeral ports.
6. Check DNS, the operating system firewall, and whether the application is listening on the expected port.
7. Use VPC Flow Logs as additional evidence.

## 11. Scenarios you should recognize in the SAA-C03

| Requirement in the question | Solution to consider |
|---|---|
| Public instance needs internet access | IGW + subnet route + public IPv4/EIP + rules |
| Private instances need internet downloads | NAT Gateway + private subnet route |
| Access S3 without public internet | S3 Gateway Endpoint |
| Reach an AWS service through private IP/PrivateLink | Interface Endpoint |
| Connect two VPCs without a central hub | VPC Peering, if transit is not required |
| Connect many VPCs in a hub-and-spoke topology | Transit Gateway |
| Connect an office/datacenter to AWS over VPN | Site-to-Site VPN |
| Dedicated connectivity to AWS | Direct Connect |
| Block an IP range at subnet level | Network ACL |
| Allow traffic between tiers based on group identity | Security Groups |
| Investigate accepted/rejected traffic | VPC Flow Logs |

## 12. Check your understanding

Try to answer before expanding the explanation. The goal is to identify the key requirement and eliminate options that solve a different problem.

<details>
<summary><strong>Question 1 — Private S3 access</strong></summary>

An application in a private subnet needs to access Amazon S3 objects. The company wants to avoid a NAT Gateway and does not want traffic to use the public internet. What is the best fit?

**Answer:** An S3 Gateway Endpoint associated with the subnet's route table. Also check endpoint and bucket policies.
</details>

<details>
<summary><strong>Question 2 — Return traffic</strong></summary>

Inbound traffic is allowed by a Security Group. Must you add a specific rule to allow response traffic?

**Answer:** No. Security Groups are stateful and automatically allow return traffic for permitted connections.
</details>

<details>
<summary><strong>Question 3 — Connectivity across three VPCs</strong></summary>

An architecture has multiple VPCs and needs centralized, transitive connectivity. Which service is likely to simplify this topology?

**Answer:** AWS Transit Gateway. VPC Peering is not transitive.
</details>

<details>
<summary><strong>Question 4 — Private instance with no outbound access</strong></summary>

An EC2 instance in a private subnet cannot download updates. Its route table has no default route to a NAT Gateway. What change is expected?

**Answer:** Create or use a NAT Gateway in a public subnet with a route to the IGW, then add a `0.0.0.0/0` route in the private route table pointing to the NAT Gateway.
</details>

## 13. Review checklist

- [ ] A VPC is regional; a subnet belongs to one AZ.
- [ ] A public subnet has a route to an Internet Gateway.
- [ ] Direct public IPv4 connectivity requires a public IP/EIP on the instance.
- [ ] NAT Gateway enables outbound connections initiated by private resources.
- [ ] Route tables use longest prefix match.
- [ ] Security Groups are stateful; NACLs are stateless.
- [ ] NACLs evaluate rules numerically and support allow/deny.
- [ ] Gateway Endpoints support S3 and DynamoDB; Interface Endpoints use ENIs.
- [ ] VPC Peering is non-transitive; Transit Gateway supports hub-and-spoke topologies.
- [ ] Flow Logs record traffic metadata, not packet payloads.

## Official references

- [Amazon VPC — User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- [VPC CIDR blocks](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html)
- [Subnet route tables](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [Internet gateways](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
- [NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [Security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [VPC endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)

---

*This is an independent study note designed to support AWS SAA-C03 preparation. Always consult the official documentation for service details and updates.*
