# Securing VPC Resources with Security Groups and Network ACLs

A hands-on AWS networking and security lab covering traffic filtering, security group references, network ACL rule priority, and administrative access to a private EC2 instance through a bastion host and AWS Systems Manager Session Manager.

Completed as part of AWS Academy Cloud Architecting — Lab 4.1.

## Overview

The environment contains two public-facing proxy instances and an application server in a private subnet.

The lab progressively restricts access to the application server:

1. Examine the existing VPC, routes and security groups.
2. Test HTTP access through both proxy instances.
3. Restrict application access to one proxy's private IP address.
4. Replace the IP-based rule with a security group reference.
5. Test network ACL deny rules and rule-number priority.
6. Connect to the private application server through a bastion host.
7. Use Session Manager to administer the same server without requiring inbound SSH access.

> **Scope:** This is a lab configuration, not a production deployment guide. Broad HTTP access and temporary public SSH access are used to demonstrate specific behaviors. Production considerations are listed separately below.

## Environment

| Resource | Configuration |
|---|---|
| VPC | `LabVPC`, CIDR `10.0.0.0/16` |
| Public subnets | `PublicSubnetA` and `PublicSubnetB` |
| Private subnet | `PrivateSubnet` |
| Public routing | Default route to an Internet Gateway |
| Private routing | Default route to a NAT Gateway in `PublicSubnetA` |
| `ProxyServer1` | Public EC2 instance in `PublicSubnetA`, initially associated with `ProxySG` |
| `ProxyServer2` | Public EC2 instance in `PublicSubnetB`, initially associated with `ProxySG2`; later repurposed as `Bastion` |
| `AppServer` | Private EC2 instance without a public IPv4 address, associated with `AppServerSG` |
| Application | Webpage hosted on `AppServer`, reached through the proxy instances |
| Administration | SSH through a bastion host and browser-based Session Manager |

## Architecture

### Initial HTTP request path

```text
Internet client
      |
      v
Internet Gateway
      |
      +--> ProxyServer1 — PublicSubnetA — ProxySG
      |          |
      |          +-----------------------+
      |                                  |
      +--> ProxyServer2 — PublicSubnetB   |
                 |          ProxySG2     |
                 |                       |
                 +-----------------------+
                                         |
                                         v
                               AppServer — PrivateSubnet
                                         AppServerSG
```

The public instances forward HTTP requests to the private application server. The browser connects to a proxy's public IP address, not directly to `AppServer`.

### Private-subnet outbound path

```text
AppServer
    |
    v
NAT Gateway in PublicSubnetA
    |
    v
Internet Gateway
    |
    v
Internet
```

The private route table sends `10.0.0.0/16` traffic through the local VPC route and other IPv4 traffic through the NAT Gateway. The route table was renamed from `changeme` to `Private`.

A NAT Gateway provides an outbound connectivity path; it does not make the application server directly reachable through a public IP address.

## Implementation and Results

### 1. Inspect the existing network

Reviewed:

- The VPC and its three subnets.
- Public and private route-table associations.
- The NAT Gateway and Internet Gateway.
- Instance public and private IPv4 addresses.
- Existing inbound and outbound security group rules.

The initial `AppServerSG` HTTP rule allowed TCP port 80 from `0.0.0.0/0`.

Although this rule was broad, `AppServer` still had no public IPv4 address or direct public inbound path. A security group rule permits traffic; it does not create network reachability.

### 2. Enable and test both proxy paths

Added an inbound HTTP rule to `ProxySG2` so that both proxies accepted TCP port 80 from `0.0.0.0/0`.

| Test | Result |
|---|---|
| Open `ProxyServer1` public IP in a browser | Application webpage loads |
| Open `ProxyServer2` public IP in a browser | Same application webpage loads |

These tests confirmed that both proxies could reach the application server under the initial rules.

### 3. Restrict HTTP access to one private IP

Replaced the broad HTTP source in `AppServerSG` with:

```text
<ProxyServer1PrivateIP>/32
```

| Source path | Result |
|---|---|
| Browser → `ProxyServer1` → `AppServer` | Loads |
| Browser → `ProxyServer2` → `AppServer` | Fails to load; connection times out |

In this lab's proxy configuration, the connection arriving at `AppServer` uses the proxy's private IP as its source.

The `/32` rule allows only the specified IPv4 address. It works for a single known proxy but requires manual updates when approved source addresses change.

### 4. Replace the IP rule with a security group reference

Changed the HTTP source in `AppServerSG` from the single private IP to `ProxySG`.

Then changed `ProxyServer2` from `ProxySG2` to `ProxySG`.

```text
ProxyServer1 — ProxySG ──┐
                        ├── TCP 80 ──> AppServer — AppServerSG
ProxyServer2 — ProxySG ──┘
```

| Test after the change | Result |
|---|---|
| Access through `ProxyServer1` | Loads |
| Access through `ProxyServer2` | Loads |

The application rule now identifies the approved source security group rather than maintaining a list of proxy IP addresses.

A security group reference does not copy the referenced group's rules into the application group. It identifies eligible source network interfaces associated with that group; routing and other applicable controls must still permit the connection.

### 5. Test network ACL rule priority

Added an inbound HTTP deny rule to the lab network ACL:

| Rule number | Protocol / port | Source | Action |
|---|---|---|---|
| 99 | TCP 80 | `0.0.0.0/0` | DENY |

Access through `ProxyServer1` timed out even though the security groups allowed HTTP.

Next, added a lower-numbered allow rule:

| Rule number | Protocol / port | Source | Action |
|---|---|---|---|
| 98 | TCP 80 | `0.0.0.0/0` | ALLOW |
| 99 | TCP 80 | `0.0.0.0/0` | DENY |

The webpage loaded again.

Network ACL rules are evaluated from the lowest rule number upward. Rule `98` matched first, so rule `99` was not applied to that traffic.

> A network ACL affects its associated subnets, not automatically every subnet in the VPC. Check subnet associations when determining the scope of a rule.

### 6. Access the private server through a bastion host

Repurposed `ProxyServer2` as `Bastion`:

- Renamed the instance.
- Created `BastionSG`.
- Replaced its `ProxySG` association with `BastionSG`.
- Added inbound SSH to `AppServerSG` from `<BastionPrivateIP>/32`.

For the lab exercise, `BastionSG` allowed inbound TCP port 22 from `0.0.0.0/0`.

**This broad SSH source is a lab setting, not a recommended production configuration.**

#### SSH connection

From the lab terminal:

```bash
# Start an SSH agent and load the lab key
exec ssh-agent bash
ssh-add ~/.ssh/labsuser.pem

# Replace BASTION_PUBLIC_IP with the actual public IP address
ssh -A ec2-user@BASTION_PUBLIC_IP
```

From the bastion:

```bash
# Replace APP_SERVER_PRIVATE_IP with the actual private IP address
ssh ec2-user@APP_SERVER_PRIVATE_IP

# Create the lab validation file
touch newfile.txt
```

Exit each session when finished:

```bash
exit
```

The `-A` option enables SSH agent forwarding. This allows authentication to the private server without copying the private key onto the bastion.

This is **agent forwarding**, not an SSH local port-forwarding command. Agent forwarding should be used only with trusted intermediate hosts.

### 7. Administer AppServer through Session Manager

Opened a browser-based Session Manager shell on `AppServer` from the EC2 console.

Updated the application webpage:

```bash
sudo sed -i \
  's/instance!/instance! Session manager was used to edit this file./g' \
  /var/www/html/index.html
```

Then opened `ProxyServer1` in a browser and confirmed that the webpage displayed:

```text
Session manager was used to edit this file.
```

This demonstrated administration of the private instance through Session Manager without using the bastion SSH path.

A standard Session Manager shell is not an SSH session and does not require inbound TCP port 22.

#### Session Manager prerequisites

Session Manager requires:

- SSM Agent installed and running on the instance.
- Appropriate instance permissions for Systems Manager.
- IAM permissions for the person starting the session.
- Network connectivity to the required Systems Manager endpoints, generally through outbound HTTPS.
- Security groups and network ACLs that permit the required traffic, including return traffic.

For a private instance, this connectivity can be provided through a NAT path or appropriately configured VPC endpoints.

> The supplied lab instructions refer to an existing network ACL block on port 22 in the final task, but the preceding steps only describe adding HTTP rules on port 80. No additional SSH-blocking test is claimed here.

## Security Group Configuration at the End of the Lab

| Group | Associated instance | Inbound rules configured during the lab |
|---|---|---|
| `ProxySG` | `ProxyServer1` | TCP 80 from `0.0.0.0/0` |
| `BastionSG` | `Bastion`, formerly `ProxyServer2` | TCP 22 from `0.0.0.0/0` — lab-only setting |
| `AppServerSG` | `AppServer` | TCP 80 from `ProxySG`; TCP 22 from `<BastionPrivateIP>/32` |
| `ProxySG2` | No longer associated with `ProxyServer2` | HTTP rule added earlier; group no longer used by that instance |

Once `ProxyServer2` becomes the bastion and loses `ProxySG`, it is no longer the second HTTP proxy in the final configuration.

## Security Groups vs. Network ACLs

| Characteristic | Security group | Network ACL |
|---|---|---|
| Association | Network interface | Subnet |
| Rules | Allow rules only | Allow and deny rules |
| Evaluation | Applicable allow rules are combined | Lowest-numbered matching rule wins |
| Connection tracking | Stateful | Stateless |
| Return traffic | Automatically permitted for tracked, allowed connections | Must be permitted by the applicable rules |
| Source configuration | Can reference another security group where supported | Uses CIDR ranges rather than security group references |

Network ACLs and security groups are separate controls. An allow in a security group does not override a matching network ACL deny.

## Verification Summary

| Configuration or action | Result |
|---|---|
| Broad application HTTP rule | Both proxy paths work |
| Application HTTP restricted to `ProxyServer1` private IP | Only the first proxy path works |
| Application HTTP source set to `ProxySG`; both proxies use it | Both proxy paths work |
| Network ACL rule `99` denies HTTP | Tested HTTP path fails |
| Network ACL rule `98` allows HTTP before rule `99` | Tested HTTP path works again |
| SSH through bastion with agent forwarding | Private application server reached; `newfile.txt` created |
| Session Manager shell | Application file edited without using the bastion SSH path |
| Reload webpage through `ProxyServer1` | Updated text appears |

Use a fresh request or hard refresh during HTTP testing to avoid mistaking cached content for successful connectivity.

## Production Considerations

The lab demonstrates control behavior, not a complete production security baseline.

For a production deployment:

- Restrict bastion SSH access to approved administrative source addresses instead of `0.0.0.0/0`.
- Where requirements permit, use Session Manager and remove unnecessary inbound SSH access.
- Use HTTPS for application traffic.
- Prefer security group references for application-tier access when appropriate.
- Review outbound rules, network ACL return paths and subnet associations.
- Do not copy private keys onto bastion hosts or commit them to Git.
- Configure Session Manager logging deliberately. CloudTrail records API activity; supported shell-session output can be sent to CloudWatch Logs or S3 when configured.
- Do not assume session transcripts are enabled by default.
- Review the remaining allow rule `98`: it restores HTTP access and therefore neutralizes the later matching deny rule `99`.

## Key Takeaways

- A private subnet and a restrictive security group solve different parts of network isolation.
- A security group rule does not create a route or assign a public IP address.
- IP-based rules can restrict a single source, while security group references simplify access for an approved group of instances.
- Network ACL rule numbers determine which matching rule takes effect.
- Network ACL changes must be evaluated in the context of subnet associations and return traffic.
- SSH agent forwarding avoids copying private keys to the bastion but still requires trust in that host.
- Session Manager provides a separate administrative access path without requiring inbound SSH.
