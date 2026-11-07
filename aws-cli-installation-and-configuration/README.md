# Installing and Configuring the AWS CLI
### AWS CLI · Amazon EC2 · IAM · Linux

A hands-on activity covering AWS CLI installation on a Red Hat Linux EC2 instance, credential configuration, and IAM inspection from the terminal.

**Module 1 — Activity 1: Install and Configure the AWS CLI**

## Overview

The activity uses a preconfigured EC2 instance as a consistent environment for installing and practicing the AWS CLI.

The workflow includes:

- Connecting to the instance through SSH.
- Installing AWS CLI version 2.
- Inspecting an existing IAM user and policy.
- Configuring CLI access with lab-provided credentials.
- Listing IAM users from the terminal.
- Retrieving a managed policy document through a CLI challenge.

> **Scope:** Access keys are used to follow the lab instructions. This is not a recommendation to store long-lived IAM user credentials on production EC2 instances.

---

## Lab Environment

| Component | Details |
| :--- | :--- |
| Command environment | Preconfigured Red Hat Linux EC2 instance |
| SSH username | `ec2-user` |
| SSH authentication | Lab-provided private key |
| CLI version | AWS CLI v2 |
| IAM user | `awsstudent` |
| IAM policy | `lab_policy` |
| Default Region | `us-east-1` |
| Output format | JSON |

The EC2 instance, IAM user, access key, and policy were provided by the lab.

## 1. Connect Through SSH

For a Linux or macOS terminal, restrict the private key's permissions before connecting.

Replace `INSTANCE_PUBLIC_IP` with the lab instance's actual public IPv4 address.

```bash
chmod 400 labsuser.pem
ssh -i labsuser.pem ec2-user@INSTANCE_PUBLIC_IP
```

Windows users can use PuTTY with the supplied `.ppk` key.

The SSH key authenticates access to the operating system. It is separate from the AWS credentials used by the CLI.

## 2. Install AWS CLI v2

Install the unzip utility and download the Linux x86-64 installer:

```bash
sudo yum install -y unzip

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
  -o "awscliv2.zip"

unzip awscliv2.zip
sudo ./aws/install
```

> **Note:** The installer above is for an x86-64 environment. Select the appropriate installer when using a different architecture.

### Verify installation

```bash
aws --version
aws help
```

Press `q` to exit the help pager.

**Python distinction:** The activity checks `python3 --version`, but the AWS CLI v2 bundled installer does not require a separately installed Python interpreter. The older Python requirements in the activity instructions should not be treated as CLI v2 prerequisites.

## 3. Inspect the IAM Configuration

Review `awsstudent` in the IAM console and inspect `lab_policy`.

The policy defines the AWS operations available to the lab user. Installing the CLI does not grant additional permissions or bypass IAM restrictions.

Also, review the user's existing access key.

The secret access key cannot be retrieved again from IAM after creation. The lab makes the previously captured value available through its credentials panel.

## 4. Configure CLI Access

Run the following command:

```bash
aws configure
```

Enter the lab-provided credentials and select:

| Prompt | Value |
| :--- | :--- |
| AWS Access Key ID | Lab-provided value |
| AWS Secret Access Key | Lab-provided value |
| Default Region | `us-east-1` |
| Default output format | `json` |

No working credentials are included in this repository.

On Linux, the default configuration locations are:

- `~/.aws/credentials` — saved credentials.
- `~/.aws/config` — settings such as Region and output format.

These files are local configuration, not files to commit to Git.

## 5. Query IAM from the Terminal

Use the following command to inspect IAM users:

```bash
aws iam list-users
```

The expected response is a JSON document containing the users visible under the active identity's permissions.

Compare the returned information with the IAM console.

---

## Challenge: Export the Managed Policy

**Objective:** Retrieve the JSON document for `lab_policy` using the CLI rather than the console.

The workflow below is a reference solution. It should not be described as completed until the exported file has been checked.

### Step 1 — Find the policy ARN

List customer managed policies:

```bash
aws iam list-policies \
  --scope Local \
  --query "Policies[?PolicyName=='lab_policy'].{Name:PolicyName,Arn:Arn}" \
  --output json
```

Copy the correct ARN and assign it to a variable:

```bash
POLICY_ARN='REPLACE_WITH_LAB_POLICY_ARN'
```

### Step 2 — Find the default version

```bash
aws iam get-policy \
  --policy-arn "$POLICY_ARN" \
  --query 'Policy.DefaultVersionId' \
  --output text
```

Use the returned version rather than assuming it is `v1`:

```bash
VERSION_ID='REPLACE_WITH_DEFAULT_VERSION'
```

### Step 3 — Export the policy document

```bash
aws iam get-policy-version \
  --policy-arn "$POLICY_ARN" \
  --version-id "$VERSION_ID" \
  --query 'PolicyVersion.Document' \
  --output json > lab_policy.json
```

### Step 4 — Inspect the file

```bash
cat lab_policy.json
```

Confirm that the file contains a policy document with fields such as `Version` and `Statement`, rather than policy metadata.

**Why multiple commands?** `get-policy` returns metadata, including the default version ID. `get-policy-version` returns the actual policy document.

This workflow applies to a managed policy. Inline policies use different retrieval commands.

## Verification Checklist

| Check | Expected Evidence |
| :--- | :--- |
| SSH connection | Terminal session on the lab instance |
| CLI installation | `aws --version` reports AWS CLI v2 |
| CLI help | `aws help` opens successfully |
| Account access | IAM command returns an authorized response |
| IAM inspection | User details can be compared with the console |
| Challenge, if completed | `lab_policy.json` contains the intended policy document |

A working installation and valid authentication do not imply unrestricted account access. Each operation still requires authorization.

## Security Considerations

- Never commit access keys, private SSH keys, or the `.aws` directory.
- Avoid exposing credentials in screenshots or terminal recordings.
- Use only the credentials supplied for the authorized lab.
- Temporary credentials also require a session token; an access key and secret alone are not sufficient for those sessions.
- For production EC2 workloads, prefer an appropriate instance role and temporary credentials instead of saved IAM user access keys.
- Review exported policies before publishing them; they can reveal account identifiers and internal resource names.


## Key Takeaways

- CLI installation, authentication, and authorization are separate steps.
- SSH credentials and AWS API credentials serve different purposes.
- AWS CLI v2 can be installed using its bundled Linux installer.
- CLI operations remain subject to IAM policies.
- Managed-policy metadata and policy documents require different commands.
- Retrieve the default policy version rather than assuming a version number.
- Keep credentials outside version control.
