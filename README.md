# fake-service-app-v1

A multi-VPC AWS infrastructure project built with Terraform, simulating a retail banking microservice environment. The architecture demonstrates VPC peering, load-balanced high availability, secure private-subnet deployments, and DNS/TLS termination using Route 53 and ACM.

## Architecture

The project simulates three microservices for a retail banking platform:

- **Customer Profile Service** — public-facing entry point
- **Account Service** — internal service called by Customer Profile
- **Statement Service** — internal service called by Account

Each service lives in its own VPC, peered together to allow controlled cross-VPC communication, mirroring how real microservice architectures isolate blast radius while still enabling service-to-service calls.

```
Internet
   │
   ▼
Customer ALB (public)
   │
   ▼
Customer Instances (private) ──VPC Peering──► Account ALB (internal)
                                                    │
                                                    ▼
                                              Account Instances (private) ──VPC Peering──► Statement ALB (internal)
                                                                                                  │
                                                                                                  ▼
                                                                                          Statement Instances (private)
```

## Key Design Decisions

- **Two instances per service** — deployed across two Availability Zones for high availability and failover testing
- **All application instances run in private subnets** — no direct internet exposure
- **Application Load Balancers** distribute traffic and perform health checks for each service
- **VPC Peering** connects the three VPCs (Customer ↔ Account ↔ Statement) with scoped CIDR-based routing
- **Bastion host** in a public subnet is the only SSH entry point into the private network
- **Route 53 + ACM** provide a custom DNS name and HTTPS termination, instead of exposing the ALB's default AWS endpoint
- **Infrastructure as Code** — the entire stack (VPCs, subnets, route tables, security groups, ALBs, target groups, instances, DNS, and certificates) is provisioned and version-controlled through Terraform

## Tech Stack

| Component | Tool |
|---|---|
| IaC | Terraform (>= 1.12) |
| Cloud Provider | AWS (>= 6.0 provider) |
| Compute | EC2 (Ubuntu) |
| Load Balancing | Application Load Balancer |
| Networking | VPC, VPC Peering, NAT Gateway, Bastion Host |
| DNS / TLS | Route 53, AWS Certificate Manager |
| Test Service | [fake-service](https://github.com/nicholasjackson/fake-service) — simulates microservice call chains and latency |

## Project Structure

```
fake-service-app-v1/
├── versions.tf            # Terraform & provider version constraints
├── variables.tf            # Input variables (CIDR blocks, instance type, etc.)
├── vpc.tf                  # VPCs, subnets, route tables, security groups, ALBs
├── instance.tf             # EC2 instances, target group attachments
├── key-pair.tf              # SSH key pair for instance access
├── data.tf                  # Data sources (e.g. latest Ubuntu AMI)
├── route53-acm.tf            # Route 53 hosted zone, DNS records, ACM certificate
├── scripts/
│   ├── customer-profile.sh   # Bootstraps the Customer Profile service
│   ├── account.sh             # Bootstraps the Account service
│   └── statement.sh           # Bootstraps the Statement service
└── .gitignore
```

## Getting Started

### Prerequisites

- Terraform >= 1.12
- AWS CLI configured with a named profile
- A registered domain (for Route 53 / ACM)

### Deploy

```bash
terraform init
terraform plan
terraform apply
```

### Access the application

Once applied, the app is reachable via the custom domain configured in Route 53:

```bash
curl https://fake-service.swundev.online
```

### SSH Access (via Bastion)

```bash
ssh-add fake-service-app.pem
ssh -A -i fake-service-app.pem ubuntu@<bastion-public-ip>

# from the bastion, hop into any private instance
ssh ubuntu@<private-instance-ip>
```

## What This Project Demonstrates

- Designing and provisioning multi-VPC network topologies with Terraform
- Connecting isolated VPCs via peering with correctly scoped route tables and security groups
- Building highly available services behind Application Load Balancers
- Automating instance configuration with `user_data` and `templatefile()` for dynamic upstream wiring
- Securing private infrastructure with a bastion host pattern
- Issuing and validating TLS certificates with ACM, and wiring custom domains through Route 53
- Debugging real infrastructure issues — AZ placement errors, security group misconfigurations, target group health checks, and DNS propagation

## Notes

This project uses [fake-service](https://github.com/nicholasjackson/fake-service) to simulate realistic microservice request chains, response times, and upstream call dependencies without needing real banking application code.
