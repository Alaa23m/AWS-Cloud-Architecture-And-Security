# Inventory Manager on AWS

Documentation for Inventory Manager, a three-tier stock-control web application with an Angular frontend, an ASP.NET Core API and a MySQL database. This document describes the application, a smaller validation deployment, and a proposed AWS production architecture.

> **Scope:** The production architecture is a proposed design. It should not be presented as fully deployed unless its components have been implemented and verified. The repository layout at the end of this document lists the files currently documented here; application code, database scripts and deployment configuration should be added to that list if they are published.

## Application

Inventory Manager is a small stock-control system for office and IT supplies. It keeps a product catalogue, tracks units in stock, logs stock movements and flags products that have reached their reorder level.

| Area | Behavior |
|---|---|
| Authentication | Username and password login. Protected API calls require a valid JWT; the login and health-check endpoints are exceptions. |
| Dashboard | Totals for products, units and stock value, products to reorder, and latest stock movements. |
| Products | Search and filter by name, SKU, category or low stock. Create, edit and delete products. View stock history per product. |
| Stock movements | IN (received), OUT (issued) and ADJUST (physical count). Stock cannot go negative and changes are logged. |
| Categories and suppliers | Maintained separately. A category that still has products cannot be deleted. |
| Roles | Admin has full access. Viewer has read-only access. |
| Health check | `GET /health` is intended to return HTTP 200 when the database is reachable and HTTP 503 when it is not. |

### Stack

| Layer | Technology | Runs on in the EC2-based deployment |
|---|---|---|
| Frontend | Angular 21, Bootstrap 5 | Static files served by nginx on EC2 |
| Web server and proxy | nginx 1.24 | EC2, port 80 behind the load balancer in the proposed design |
| Backend API | ASP.NET Core Web API (.NET 8), Entity Framework Core, JWT authentication | EC2, systemd service on `localhost:5000` |
| Database | MySQL 8.4 LTS | Amazon RDS |
| Operating system | Ubuntu Server 24.04 LTS | EC2 |

The versions above describe the intended stack. Check them against the application files before treating this document as a reproducible build specification.

### Request flow

**Validation deployment:** The smaller deployment serves the Angular build and API from an EC2 instance through nginx.

1. The browser requests the site through nginx.
2. nginx returns the Angular build from `/var/www/inventory-ui`.
3. Angular sends login credentials to `/api/auth/login`. The API checks the stored password hash and returns a signed JWT when authentication succeeds.
4. Subsequent protected API requests include the token. nginx proxies `/api/` and `/health` to the API on `127.0.0.1:5000`.
5. The API validates tokens and permissions, then communicates with the RDS database on port 3306.

**Proposed production design:** Users access CloudFront over HTTPS. AWS WAF filters requests through a web ACL attached to CloudFront. CloudFront forwards requests to an internet-facing Application Load Balancer (ALB), which sends them to nginx on the EC2 instances. nginx serves the Angular build and proxies API requests to the local ASP.NET Core service.

CloudFront must be configured not to cache authenticated API responses. The configuration must also forward the headers and other request values required by the API, including the authorization header.

### Data model

The application design uses an `inventory_db` database with the following objects:

| Object | Purpose |
|---|---|
| `users` | Login accounts: unique username, salted PBKDF2 password hash, role (Admin or Viewer), active flag |
| `categories` | Product groups |
| `suppliers` | Companies the products are bought from |
| `products` | SKU, name, price, `quantity_in_stock`, `reorder_level`, category and supplier references |
| `stock_movements` | Every quantity change: type (IN, OUT, ADJUST), amount, note, date |
| `v_low_stock_products` | View of products at or below their reorder level, with the shortage |

The original deployment instructions refer to `database/inventory_schema.sql` and two seeded demo accounts, one Admin and one Viewer. **That SQL file is not shown in the repository layout below.** Add it to the repository before describing database initialization as reproducible from this repository. Demo accounts must be changed or removed outside a lab.

## Architecture

![High-level architecture](docs/architecture.png)

The editable source is [`docs/architecture.drawio`](docs/architecture.drawio).

> Update both diagram files if the image still shows WAF as a separate network hop before CloudFront, S3 as the frontend origin, or an Elastic IP attached to each Auto Scaling instance. The description below is the intended corrected design.

The proposed design targets the Europe (Ireland) Region, `eu-west-1`. It places CloudFront and AWS WAF in front of an ALB, runs the application on two EC2 instances in an Auto Scaling group across two Availability Zones, and keeps the database in a private subnet.

```text
Users
  |
  v
CloudFront (AWS WAF web ACL attached)
  |
  v
Internet-facing Application Load Balancer
  |
  v
EC2 Auto Scaling group (2 Availability Zones)
  |  nginx -> Angular files
  |        -> ASP.NET Core API on localhost:5000
  v
RDS for MySQL (Single-AZ, private subnet)
```

CloudFront uses the ALB as its origin in this version of the design. nginx on EC2 serves the Angular build; **S3 is not also claimed as a frontend origin**. The Internet Gateway provides connectivity for public subnets, but it is not an application-processing step between CloudFront and the ALB.

### Components

| Area | Service | Role in this proposed design |
|---|---|---|
| Edge | Amazon CloudFront | Public entry point. Terminates viewer TLS and forwards requests to the ALB. Static frontend files may be cached under an appropriate cache policy; `/api/*` and `/health` must not cache dynamic or authenticated responses. |
| Edge | AWS WAF | Web ACL attached to the CloudFront distribution. |
| Edge | AWS Certificate Manager | Certificate for the CloudFront custom domain in `us-east-1`; separate certificate for the ALB origin hostname in the ALB's Region. |
| Edge | Domain name | Public hostname for CloudFront and a hostname matching the ALB origin certificate. |
| Network | VPC | Network for the workload. |
| Network | Subnets | Two public subnets for the ALB and application instances, and two private subnets in different Availability Zones for the RDS DB subnet group. |
| Network | Internet Gateway | Internet connectivity for resources in public subnets when their routing and addressing permit it. |
| Network | Security groups | ALB-SG, App-SG and DB-SG restrict inbound traffic between tiers. |
| Compute | Application Load Balancer | HTTPS listener forwards traffic to a target group on instance port 80. Its target group health-checks an application endpoint. |
| Compute | EC2 | Proposed baseline of two `m6i.xlarge` instances, one per Availability Zone, each running nginx and the .NET 8 API. Instance sizing requires workload testing before production use. |
| Compute | Auto Scaling group | Spans both application subnets. Proposed minimum and desired capacity: two instances. Configure ALB health checks for replacement; scaling policies require defined metrics and thresholds. |
| Storage | EBS | Proposed encrypted 60 GB application volume per instance. Volume sizing requires validation. |
| Database | RDS for MySQL | Proposed MySQL 8.4, `db.m5.4xlarge`, Single-AZ, 50 GB storage. RDS is not publicly accessible. Database sizing and availability requirements need validation. |
| Security | IAM policies | Instance role with permissions scoped to Systems Manager, application secrets and monitoring as needed. No long-lived access keys on instances. |
| Security | AWS KMS | Encryption keys as configured for EBS, RDS and Secrets Manager. |
| Security | AWS Secrets Manager | Proposed storage for database credentials and the JWT signing key. |
| Security | Amazon GuardDuty | Proposed threat-detection service. |
| Management | AWS CloudTrail | Account API activity logging. |
| Management | Amazon CloudWatch | Proposed metrics, logs and alarms for EC2, ALB and RDS. |
| Management | AWS Systems Manager | Session Manager for administration without an inbound SSH rule, subject to the instance having the required network access and permissions. |
| Backup | AWS Backup | Proposed backup plan for the EBS volumes and RDS instance. |

In the proposed design, the instance retrieves application secrets from Secrets Manager and passes them to the API service as `ConnectionStrings__InventoryDb` and `Jwt__Key`. The standard ASP.NET Core configuration system can read these environment variables. This secret-loading step must be implemented in the instance setup; it is not achieved merely by adding Secrets Manager to the architecture diagram.

**Availability limitation:** Two EC2 instances improve availability of the application tier, but the proposed Single-AZ RDS database remains a Single-AZ dependency. A database high-availability requirement would need a different RDS configuration and cost estimate.

### Network layout

| Resource | CIDR | Placement | Contents |
|---|---|---|---|
| VPC | `10.0.0.0/16` | Region | Application network |
| Public subnet A | `10.0.1.0/24` | AZ A | ALB, application instance 1 |
| Public subnet B | `10.0.2.0/24` | AZ B | ALB, application instance 2 |
| Private subnet A | `10.0.11.0/24` | AZ A | RDS instance for the proposed Single-AZ deployment |
| Private subnet B | `10.0.12.0/24` | AZ B | Second subnet in the DB subnet group |

The RDS DB subnet group includes subnets in at least two Availability Zones, even though the proposed database instance is Single-AZ.

The Auto Scaling instances do **not** require individual Elastic IPs for incoming application traffic; that traffic enters through the ALB. Because this design places instances in public subnets, their outbound access to AWS services and package repositories must still be configured explicitly, for example through automatically assigned public IPv4 addresses and suitable route tables. Their security group must not permit direct inbound application access from the internet.

### Security groups

| Group | Applied to | Inbound |
|---|---|---|
| ALB-SG | Internet-facing ALB | TCP 443 from the AWS-managed CloudFront origin-facing prefix list |
| App-SG | EC2 instances | TCP 80 from ALB-SG |
| DB-SG | RDS instance | TCP 3306 from App-SG |

For an internet-facing ALB, restrict direct access rather than assuming that CloudFront automatically hides it. In addition to the ALB security-group restriction, configure CloudFront to send a private custom origin header and configure the ALB listener to forward only requests with that header. Protect and rotate its value. The ALB can forward traffic to nginx on port 80 within the VPC; this is different from HTTPS between CloudFront and the ALB.

### Baseline and target design

The smaller validation deployment and the proposed production design have different scopes.

| Concern | Validation baseline | Proposed production design |
|---|---|---|
| Availability | One application instance and one database | Two application instances behind an ALB across two AZs. Database remains Single-AZ. |
| Transport security | Direct HTTP access in the validation setup | HTTPS from users to CloudFront and from CloudFront to the ALB |
| Entry point | Instance endpoint | CloudFront with WAF, forwarding to an ALB |
| Secrets | Application configuration used by the validation deployment | Secrets Manager with instance-role access and implemented secret retrieval |
| Encryption at rest | Not established by the validation description | Encryption configured for EBS and RDS |
| Administration | Not established by the validation description | Session Manager; no inbound SSH rule |
| Detection and audit | Not established by the validation description | CloudTrail, GuardDuty and CloudWatch alarms |
| Recovery | Not established by the validation description | AWS Backup plan for EBS and RDS |

## Cost estimate

**The previous cost table is not a valid total for this corrected architecture.** It priced the EC2 instances with Windows licensing despite the documented Ubuntu deployment, included two Elastic IPs removed from this design, and omitted several proposed services and usage-based charges.

Before publishing a monetary estimate, create a dated AWS Pricing Calculator estimate for `eu-west-1` using the final architecture and explicit usage assumptions:

| Item to estimate | Assumption to confirm |
|---|---|
| EC2 | Two Linux/Ubuntu `m6i.xlarge` instances, estimated monthly running hours |
| EBS | 60 GB per instance, selected volume type and any applicable usage |
| RDS | MySQL 8.4, proposed instance type, Single-AZ, 50 GB storage and backup assumptions |
| ALB | Running hours and expected capacity usage |
| Public IPv4 | Number of public IPv4 addresses actually assigned to the ALB and/or instances |
| CloudFront | Requests and data transfer |
| AWS WAF | Web ACL, rules and request volume |
| Secrets Manager and KMS | Secrets, keys and expected requests |
| CloudWatch, CloudTrail and GuardDuty | Logs, monitoring and detection usage |
| AWS Backup | Backup storage and retention |
| Support, domain and data transfer | Include only the selected support plan and applicable charges |

Do not label a subtotal excluding these items as the **total production cost**. The proposed instance sizes also need justification based on expected load and testing.

## Deployment

### Validation environment

A smaller deployment of the application was validated on AWS resources with smaller instance and database sizes. This validation does **not** by itself verify the CloudFront, WAF, ALB, Auto Scaling or backup components of the proposed production design.

### Runtime layout on EC2

The following layout describes the EC2-based application setup. Paths, service names and configuration values should be checked against the actual deployment files.

| Item | Value |
|---|---|
| API files | `/var/www/inventory-api/` (`InventoryApp.dll`) |
| API service | `inventory-api.service`: runs `dotnet InventoryApp.dll` as `www-data`; intended settings include `Restart=always`, `RestartSec=10` and `ASPNETCORE_ENVIRONMENT=Production` |
| API listener | `http://localhost:5000` |
| Frontend files | `/var/www/inventory-ui/` |
| nginx site | `/etc/nginx/sites-available/inventory`, enabled through `sites-enabled` |
| nginx routing | `/` serves static files with a fallback to `index.html`; `/api/` and `/health` are proxied to `127.0.0.1:5000` |
| Caching | Do not cache `index.html` as an immutable asset; review caching for versioned `.js` and `.css` files at both nginx and CloudFront |
| Configuration keys | `ConnectionStrings:InventoryDb`, `Jwt:Key`, `Jwt:Issuer`, `Jwt:Audience`, `Jwt:ExpiryMinutes` (480), `Urls` |

### Deployment files

The original instructions refer to these files:

- `database/inventory_schema.sql`
- `inventory-api.service`
- `nginx-inventory.conf`

They are **not present in the repository layout documented below**. Add and review them before publishing commands that claim a reader can reproduce the deployment from this repository.

If the SQL script is added and reviewed, a database initialization command would take this form:

```bash
mysql -h "$RDS_ENDPOINT" -u "$DB_ADMIN_USER" -p < database/inventory_schema.sql
```

The variables must be set to the actual endpoint and administrative username. Run database initialization only when appropriate for that database; the script may create tables or insert data.

If the service and nginx configuration files are added and reviewed, their setup can then be documented with commands such as:

```bash
# API service
sudo cp inventory-api.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now inventory-api

# nginx site
sudo cp nginx-inventory.conf /etc/nginx/sites-available/inventory
sudo ln -s /etc/nginx/sites-available/inventory /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

These commands alone do not install dependencies, deploy the application files, configure secrets or create AWS resources.

## Verification

These are checks to perform on a deployment, **not evidence that the complete proposed architecture has already been deployed**.

| Check | Command or action | Expected result |
|---|---|---|
| API service | `sudo systemctl status inventory-api` | `active (running)` |
| nginx configuration | `sudo nginx -t` | Configuration test passes |
| Health endpoint | Request `/health` through the deployed entry point | Documented success response when the database is reachable |
| Database | Check the schema after initialization | Five tables and the view `v_low_stock_products`, if the described SQL schema was applied |
| Authentication | Log in as Admin, then Viewer; test write requests for both | Admin can perform permitted changes; Viewer cannot write |
| ALB, if deployed | Check target-group health | Targets pass the configured health check |
| CloudFront, if deployed | Test the public hostname and API routes | Frontend and API requests reach the intended origin; authenticated responses are not cached |
| Direct ALB access, if deployed | Request the ALB without the configured origin header | Request is rejected |

Because `/health` checks database connectivity, a database outage may cause **all** ALB targets to fail health checks even if their EC2 instances are running. Consider whether separate liveness and readiness checks would better suit the final deployment.

## Repository layout

The repository layout currently documented for this project is:

```text
.
|-- README.md
`-- docs/
    |-- architecture.drawio
    `-- architecture.png
```

Update this section when application source, database scripts, service files or nginx configuration are added. Verify that both architecture files exist and match the design described above before publishing.
