# WordPress on AWS
### Amazon EC2 · RDS for MySQL · Application Load Balancer

Reference documentation for a WordPress lab using an EC2 web server, an RDS MySQL database, and an internet-facing Application Load Balancer.

> **Configuration values:** Values such as `<instance-type>` are placeholders. Replace them with the actual lab settings before presenting this document as a record of the deployed environment.

---

## Overview

The deployment separates the public entry point, application server, and database.

- **Application Load Balancer:** Receives incoming HTTP requests.
- **Amazon EC2:** Runs Apache, PHP, and WordPress.
- **Amazon RDS for MySQL:** Stores WordPress database content.
- **Security groups:** Restrict communication between the tiers.

This lab demonstrates a load-balanced entry point and a separate database tier. A load balancer in front of one instance does not make the web tier highly available.

## Architecture

### Request path

1. A visitor requests the ALB DNS name.
2. The ALB forwards the request to its target group.
3. The registered EC2 instance serves WordPress through Apache and PHP.
4. WordPress reads or writes application data in RDS over TCP port 3306.

### Placement

| Layer | Resource | Placement |
| :--- | :--- | :--- |
| Entry point | Internet-facing ALB | Public subnets in `<AZ-a>` and `<AZ-b>` |
| Web tier | EC2 instance | `<web-subnet>` |
| Database tier | RDS for MySQL | DB subnet group spanning two Availability Zones |

**Important distinction:** A DB subnet group spanning two Availability Zones does not mean that the database itself uses a Multi-AZ deployment.

## Components

| Component | Purpose | Lab Configuration |
| :--- | :--- | :--- |
| VPC | Network boundary | CIDR: `<vpc-cidr>` |
| ALB | Public HTTP entry point | Internet-facing; listener on TCP 80 |
| Target group | Routes requests and checks target health | HTTP 80; path: `<health-check-path>` |
| EC2 | WordPress application server | AMI: `<ami>`; type: `<instance-type>` |
| Web software | Serves the application | Apache, PHP, WordPress |
| RDS | Managed MySQL database | Version: `<mysql-version>`; class: `<db-instance-class>` |

---

## Network and Access Controls

Security group references restrict application traffic between the tiers.

| Security Group | Inbound Traffic | Allowed Source |
| :--- | :--- | :--- |
| ALB | TCP 80 | `0.0.0.0/0` |
| Web | TCP 80 | ALB security group |
| Web — administration | TCP 22, if SSH is used | `<admin-ip>/32` |
| Database | TCP 3306 | Web security group |

### Access boundaries

- Visitors reach WordPress through the ALB.
- The web server accepts application traffic from the ALB security group.
- The database accepts MySQL traffic from the web security group.
- Any SSH rule is restricted to the administrative source address.

These rules describe inbound access. Routing, outbound rules, and any network ACLs must also permit the required communication.

## WordPress Database Configuration

WordPress uses the following settings in `wp-config.php`:

| Parameter | Value |
| :--- | :--- |
| `DB_NAME` | `<db-name>` |
| `DB_USER` | `<db-user>` |
| `DB_PASSWORD` | Excluded from this repository |
| `DB_HOST` | `<rds-endpoint>:3306` |

Do not commit a populated `wp-config.php`, database passwords, private keys, or other credentials.

If a configuration example is included, use placeholders rather than working credentials.

---

## Verification Checklist

| Check | Expected Outcome |
| :--- | :--- |
| Open the ALB DNS name | WordPress page or installer loads |
| Inspect target group health | Registered EC2 instance is `healthy` |
| Connect to RDS from the web instance | Connection succeeds with valid credentials |
| Test from a reachable source not authorized by the database security group | Connection does not succeed |
| Inspect browser redirects | Requests remain on the intended public hostname |

**Connection-test note:** A security group block commonly produces a timeout rather than a “connection refused” response. An unsuccessful connection alone does not prove which network control blocked it.

Record actual results separately if this checklist is used as deployment evidence.

## Operational Notes

### Health checks

Before installation is complete, WordPress may return a redirect at the site root.

If the health check expects only HTTP `200`, inspect the actual response before changing the target group's success codes.

Options include:

- Using a suitable health-check endpoint.
- Allowing an expected redirect code when appropriate.

Do not broaden success codes simply to hide an application problem.

### WordPress public URL

The WordPress `siteurl` and `home` settings should use the intended public address:

- The ALB DNS name for the lab, or
- A configured domain name.

If these settings point to the EC2 instance address, WordPress can redirect visitors away from the load balancer.

### Local application files

WordPress uploads, plugins, and themes can reside on the EC2 instance disk.

Before adding more web instances, define how application files and updates will remain consistent across them.

Shared storage such as Amazon EFS can be part of that design. Offloading media to S3 addresses media storage, but does not by itself synchronize plugins, themes, or all application files.

---

## Limitations and Improvements

| Current Limitation | Improvement to Consider |
| :--- | :--- |
| HTTP-only public listener | Configure HTTPS with an ACM certificate and an ALB listener on port 443 |
| One web instance | Run multiple healthy instances across Availability Zones; use Auto Scaling to manage capacity and replacement |
| Instance-local WordPress files | Define shared storage and a consistent deployment process |
| Single-AZ database, if configured | Use an appropriate RDS Multi-AZ deployment for database failover |
| Placeholder configuration values | Replace them with verified, non-sensitive lab settings |

The document does not claim that these improvements were implemented.

## Cleanup

Remove only resources created for this lab. Before deletion, decide whether database snapshots or application backups must be retained.

Suggested order:

1. Delete the Application Load Balancer and its listeners.
2. Delete the target group.
3. Terminate the lab EC2 instance.
4. Delete the RDS instance, choosing the required final-snapshot option.
5. Delete the DB subnet group if it is no longer used.
6. Remove unused security groups after their dependencies and references are cleared.
7. Review any remaining lab volumes, snapshots, and other chargeable resources.

Do not delete shared network resources or security groups used by other deployments.

## Key Takeaways

- Separate the public entry point, web application, and database.
- Use security group references to limit communication between tiers.
- Verify target health and application behavior separately.
- Keep WordPress URLs aligned with the public entry point.
- Plan shared state before scaling the web tier.
- Distinguish a multi-AZ subnet layout from an actually highly available deployment.
- Keep credentials out of version control.
