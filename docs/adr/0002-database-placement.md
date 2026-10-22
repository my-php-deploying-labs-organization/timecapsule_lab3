# ADR-0002: Database Placement

**Status:** Accepted  
**Project:** TimeCapsule  
**Date:** 2026-10-22

## Context

In Lab 2, the TimeCapsule application and PostgreSQL database were deployed on the same EC2 instance inside the default VPC.

In Lab 3, we introduced a custom VPC (`timecapsule-dev-vpc`, `10.27.0.0/16`) with public and private subnets across two Availability Zones.

We need to choose a database placement strategy that improves network isolation while keeping the application accessible and manageable.

## Considered Options

### Option 1: Database on the application server

Both the application and PostgreSQL run on the same EC2 instance.

**Advantages:**
- Simple deployment and configuration.
- No additional database EC2 instance.
- Local database connections.

**Disadvantages:**
- Application and database share resources.
- Failure of one instance affects both components.
- Limited isolation.

### Option 2: Separate EC2 in a public subnet

PostgreSQL runs on a dedicated EC2 instance in a public subnet.

**Advantages:**
- Separate compute resources.
- Independent application and database maintenance.
- Straightforward access to package repositories.

**Disadvantages:**
- Potential exposure to the internet if networking rules are misconfigured.
- More network security configuration is necessary.
- Additional infrastructure cost.

### Option 3: Separate EC2 in a private subnet

PostgreSQL runs on `timecapsule-dev-db` in `timecapsule-dev-private-a` (`10.27.10.0/24`).

**Advantages:**
- No public IPv4 address for the database server.
- No permanent default route to the internet.
- Database connections restricted by Security Groups and PostgreSQL authentication.
- Independent application and database management.

**Disadvantages:**
- Additional EC2 and EBS costs.
- More complex networking and deployment.
- Software updates require temporary outbound connectivity.

## Decision

We selected **Option 3: a dedicated PostgreSQL server in a private subnet**.

The application runs on `timecapsule-dev-app` in the public subnet `timecapsule-dev-public-a`.

The database runs on `timecapsule-dev-db` in `timecapsule-dev-private-a`.

The application connects to PostgreSQL through the private DNS name `db.timecapsule.internal` on port 5432.

The database Security Group allows PostgreSQL connections only from the application Security Group. Administrative SSH access is provided through `timecapsule-dev-bastion`.

## Consequences

### Database updates

The database has no permanent internet access. When operating system updates are required, a temporary NAT Gateway is created in `public-a`.

A default route through the NAT Gateway is temporarily added to `timecapsule-dev-private-rt`. After running `sudo dnf -y update`, the route and NAT Gateway are removed and the associated Elastic IP is released.

The NAT Gateway incurs hourly and data-processing charges while it exists. The laboratory estimates approximately $1.25 per day if it is accidentally left running.

### Application Elastic IP

The application server uses `timecapsule-dev-app-eip` to provide a stable public IPv4 address across EC2 stop/start cycles.

Public IPv4 addresses incur ongoing hourly charges, including when the instance is stopped.

If a load balancer is introduced in front of the application, users can access the service through the load balancer's DNS name. A dedicated Elastic IP on the application instance may then become unnecessary.

### Availability limitations

The application and database each run on a single EC2 instance in `eu-central-1a`. Failure of that Availability Zone can make the application unavailable.

A future production architecture could add application instances across multiple Availability Zones, a load balancer, and a highly available database configuration.

## Result

The selected design improves database isolation by separating public application traffic from private database traffic while retaining administrative access through the bastion host.