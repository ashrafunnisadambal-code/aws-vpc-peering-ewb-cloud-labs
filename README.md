# AWS VPC Peering – EWB Cloud Labs

> Two Amazon VPCs in **ap-south-1 (Mumbai)** connected with **VPC peering**, private EC2 instances reached through a **bastion host with SSH ProxyJump**, and a controlled **security-group ICMP failure and recovery** test. Every result is backed by real AWS console and terminal screenshots.

**Author:** Ashrafunnisa Dambal  
**Programme:** EWB Cloud Labs – Cloud Computing and Networking  
**Completed:** 31 October 2026

📄 **[Read the full project report (PDF)](report/AWS_VPC_Peering_Project_Report.pdf)** · ✏️ [Editable version (DOCX)](report/AWS_VPC_Peering_Project_Report.docx)

---

## Project overview

The goal was to build two isolated networks, place private application servers in them, and let the servers talk to each other over private IP addresses only, with no NAT Gateway and no public IP on the application instances. A single public bastion host is the only entry point for SSH.

![Architecture](diagrams/architecture.png)

## AWS services used

| Service | Purpose |
|---|---|
| Amazon VPC (2 VPCs, 3 subnets) | Isolated private networks |
| Route tables | Local, internet and peering routes in both VPCs |
| Internet Gateway | Internet path for the public subnet in VPC A only |
| VPC peering connection | Private connectivity between VPC A and VPC B |
| Security groups | SSH and ICMP Echo Request rules (stateful) |
| Amazon EC2 (3 × t3.micro, Amazon Linux 2023) | Bastion, App-A, App-B |
| OpenSSH ProxyJump | One-command access to private instances through the bastion |

## Network design

| Resource | Name | CIDR / IP |
|---|---|---|
| VPC A | `ewb-vpc-a` | `10.10.0.0/16` |
| Public subnet A | `ewb-a-public` | `10.10.1.0/24` |
| Private subnet A | `ewb-a-private` | `10.10.2.0/24` |
| VPC B | `ewb-vpc-b` | `10.20.0.0/16` |
| Private subnet B | `ewb-b-private` | `10.20.1.0/24` |
| Bastion EC2 | `ewb-bastion` | `10.10.1.10` (+ public IPv4) |
| App-A EC2 | `ewb-app-a` | `10.10.2.10` (no public IP) |
| App-B EC2 | `ewb-app-b` | `10.20.1.10` (no public IP) |
| Peering connection | `ewb-a-b-peer` | Requester VPC A, accepter VPC B, **Active** |

The CIDR blocks do not overlap, which is required for peering. Peering routes exist in **both** directions:

| Route table | Destination | Target |
|---|---|---|
| `ewb-rt-a-public` | `10.20.0.0/16` | peering connection |
| `ewb-rt-a-private` | `10.20.0.0/16` | peering connection |
| `ewb-rt-b-private` | `10.10.0.0/16` | peering connection |

A ping needs a route to the destination **and** a route back; without both, it fails even if the peering shows *Active*.

## Test results

| # | Test | Command | Result | Evidence |
|---|---|---|---|---|
| A | Baseline, before peering (App-A → App-B) | `ping -c 4 10.20.1.10` | Failed as expected, 100% loss | [E08](screenshots/E08-baseline-ping-fails.jpg) |
| B | App-A → App-B after peering | `ping -c 4 10.20.1.10` | **PASS**, 4/4 replies, 0% loss | [E13](screenshots/E13-icmp-restored-ping-recovers.jpg) |
| C | App-B → App-A | `ping -c 4 10.10.2.10` | **PASS**, 4/4 replies, 0% loss | [E10](screenshots/E10-ping-app-b-to-app-a.jpg) |
| D | SSH to App-B through the bastion | `ssh ewb-app-b` | **PASS**, shell on `ip-10-20-1-10` | [E11](screenshots/E11-ssh-to-app-b-via-bastion.jpg) |

### Controlled ICMP failure and recovery

Only the inbound **ICMP Echo Request** rule on `ewb-sg-app-b` was changed; routes and peering stayed the same.

| Step | Result | Evidence |
|---|---|---|
| Rule removed, `ping -c 4 10.20.1.10` from App-A | 100% packet loss | [E12](screenshots/E12-icmp-removed-ping-fails.jpg) |
| Rule restored, same ping | 4/4 replies, 0% loss | [E13](screenshots/E13-icmp-restored-ping-recovers.jpg) |

This isolates the security group as the control that allows the traffic. Security groups are stateful, so only the inbound request rule is needed; the reply is allowed automatically.

## SSH ProxyJump (concept)

The laptop reaches private instances through the bastion with a single command. Example `~/.ssh/config` pattern (key paths and real addresses omitted):

```
Host ewb-bastion
    HostName <bastion public IPv4>
    User ec2-user
    IdentityFile <path to your key file>

Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    ProxyJump ewb-bastion
    IdentityFile <path to your key file>
```

```bash
ssh ewb-app-b      # laptop -> bastion -> App-B
```

App-B's "Last login … from 10.10.1.10" in [E11](screenshots/E11-ssh-to-app-b-via-bastion.jpg) shows the connection arrived from the bastion's private IP.

## Cleanup

All three instances were terminated and the project VPCs, subnets and route tables were deleted so no charges continue ([E14a](screenshots/E14a-cleanup-instances-terminated.jpg), [E14b](screenshots/E14b-cleanup-route-tables.jpg), [E14c](screenshots/E14c-cleanup-subnets.jpg), [E14d](screenshots/E14d-cleanup-vpcs.jpg)). The Internet Gateway, security groups and peering connection were removed as part of the cleanup. These resources no longer exist, so the IDs in the report cannot be used to reach anything.

## Evidence index

| ID | Screenshot |
|---|---|
| E01 | [VPCs and CIDR](screenshots/E01-vpcs-and-cidr.jpg) |
| E02 | [Subnet A public](screenshots/E02a-subnet-a-public.jpg) · [Subnet B private](screenshots/E02b-subnet-b-private.jpg) |
| E03 | [Internet gateway attached to VPC A](screenshots/E03-internet-gateway-attached.jpg) |
| E04 | Route tables: [A public](screenshots/E04a-rt-a-public-association.jpg) · [routes](screenshots/E04b-rt-a-public-routes.jpg) · [initial](screenshots/E04c-rt-a-public-initial-routes.jpg) · [A private](screenshots/E04d-rt-a-private-association.jpg) · [routes](screenshots/E04e-rt-a-private-routes.jpg) · [B private](screenshots/E04f-rt-b-private-association.jpg) · [routes](screenshots/E04g-rt-b-private-routes.jpg) |
| E05 | [Bastion SG](screenshots/E05a-sg-bastion-inbound.jpg) · [App-B SG](screenshots/E05b-sg-app-b-inbound.jpg) |
| E06 | [Bastion EC2](screenshots/E06a-ec2-bastion.jpg) · [App-B EC2](screenshots/E06b-ec2-app-b.jpg) |
| E07 | [Peering connection Active](screenshots/E07-peering-active.jpg) |
| E08 | [Baseline ping fails](screenshots/E08-baseline-ping-fails.jpg) |
| E10 | [App-B → App-A ping](screenshots/E10-ping-app-b-to-app-a.jpg) |
| E11 | [SSH to App-B via bastion](screenshots/E11-ssh-to-app-b-via-bastion.jpg) |
| E12 | [ICMP removed, ping fails](screenshots/E12-icmp-removed-ping-fails.jpg) |
| E13 | [ICMP restored, ping recovers (also App-A → App-B)](screenshots/E13-icmp-restored-ping-recovers.jpg) |
| E14 | Cleanup: [instances](screenshots/E14a-cleanup-instances-terminated.jpg) · [route tables](screenshots/E14b-cleanup-route-tables.jpg) · [subnets](screenshots/E14c-cleanup-subnets.jpg) · [VPCs](screenshots/E14d-cleanup-vpcs.jpg) |

## Skills demonstrated

VPC and subnet design · CIDR planning · route tables and peering routes · public vs private subnets · security group rules and stateful behaviour · ICMP and SSH troubleshooting · bastion host architecture · SSH ProxyJump · controlled failure testing · AWS resource cleanup and cost control

## Repository structure

```
aws-vpc-peering-ewb-cloud-labs/
├── README.md
├── report/        # full project report (PDF and DOCX)
├── diagrams/      # architecture diagram
├── screenshots/   # AWS console and terminal evidence (E01 to E14d)
└── .gitignore     # blocks key files such as *.pem from being committed
```

## Privacy note

The AWS account ID, the bastion's public IP, the laptop user and host name and the SSH host-key fingerprint are masked in all screenshots. No keys, passwords or access keys are included.

## Author

**Ashrafunnisa Dambal** – EWB Cloud Labs cloud computing intern
