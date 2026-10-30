# Automated Security Group Remediation
### AWS Config · AWS Lambda · IAM · CloudWatch

Monitor security group configuration changes, detect deviations from a defined baseline, and automatically restore the required inbound rules.

**AWS Academy Cloud Architecting — Lab 7.1**

> **Lab scope:** AWS Config and a preconfigured Lambda function enforce a fixed HTTP/HTTPS baseline. This is a controlled exercise, not a universal production security policy.

---

## Overview

This lab moves beyond detecting configuration changes to automatically correcting them.

I configured AWS Config to record security group changes and connected a custom rule to a supplied Lambda function. The function inspected inbound permissions, removed unexpected rules, restored the required configuration, and reported evaluation results.

### What I worked on

- Reviewed IAM roles used by AWS Config and Lambda.
- Enabled continuous recording of security groups.
- Introduced configuration drift in a test security group.
- Created a custom Lambda-backed Config rule.
- Verified the resulting security group changes.
- Examined CloudWatch logs for remediation evidence.

## Workflow

1. **Record** — AWS Config records security group configuration changes.
2. **Evaluate** — A custom Config rule invokes the Lambda function.
3. **Remediate** — The function compares permissions against its baseline and updates the security group.
4. **Verify** — Config evaluations, inbound rules, and execution logs show the outcome.

> The remediation logic is implemented inside the Lambda function. Creating a Config rule alone does not automatically repair resources.

---

## Lab Environment

| Component | Resource |
| :--- | :--- |
| VPC | `Lab VPC` |
| Test security group | `LabSG1` |
| Config service role | `AwsConfigRole` |
| Lambda execution role | `AwsConfigLambdaSGRole` |
| Lambda function | `awsconfig_lambda_security_group` |
| Custom Config rule | `EC2SecurityGroup` |
| Recorded resource type | `AWS::EC2::SecurityGroup` |
| Recording frequency | Continuous |
| Rule parameter | `debug=true` |
| Configuration delivery | Amazon S3 |
| Execution logs | Amazon CloudWatch Logs |

**Preconfigured resources:** The lab supplied the VPCs, security groups, IAM roles, and Lambda function. I configured the monitoring integration and verified its behavior.

## 1. Review IAM Permissions

### Lambda execution role

Inspected `AwsConfigLambdaSGRole` to understand how the function could inspect resources, change inbound rules, submit evaluations, and write logs.

| Capability | Permissions |
| :--- | :--- |
| Inspect security groups | `ec2:DescribeSecurityGroups` |
| Add inbound permissions | `ec2:AuthorizeSecurityGroupIngress` |
| Remove inbound permissions | `ec2:RevokeSecurityGroupIngress` |
| Submit compliance results | `config:PutEvaluations` |
| Write execution logs | `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` |

### AWS Config service role

Reviewed the existing `S3Access` policy on `AwsConfigRole` and attached the AWS managed policy `AWS_ConfigRole`.

The two service roles have different responsibilities:

- **AWS Config role:** Supports resource recording and configuration delivery.
- **Lambda role:** Supports evaluation, security group updates, and execution logging.

**Important distinction:** AWS Config delivers configuration history and snapshots to S3. These are separate from the Lambda execution logs stored in CloudWatch.

## 2. Enable Configuration Recording

Configured AWS Config to continuously record `AWS::EC2::SecurityGroup` resources using `AwsConfigRole`.

After discovery, reviewed the resource inventory and confirmed that the lab security groups were available for evaluation.

The recording configuration applies to the selected resource type in the configured Region. It should not be described as monitoring every resource in the account.

## 3. Introduce Configuration Drift

Modified `LabSG1` to allow the following inbound traffic:

| Traffic | TCP Port | Source |
| :--- | :---: | :--- |
| HTTP | 80 | `0.0.0.0/0` |
| HTTPS | 443 | `0.0.0.0/0` |
| SMTPS | 465 | `0.0.0.0/0` |
| IMAPS | 993 | `0.0.0.0/0` |

The lab baseline required HTTP and HTTPS only. The SMTPS and IMAPS permissions therefore represented configuration drift.

> Ports 465 and 993 are not inherently malicious. They were unauthorized relative to the baseline defined for this exercise.

## 4. Configure the Custom Rule

Created the custom Config rule with these settings:

| Setting | Value |
| :--- | :--- |
| Rule name | `EC2SecurityGroup` |
| Rule type | Custom Lambda rule |
| Function | Supplied Lambda function ARN |
| Trigger | Configuration changes |
| Resource scope | `AWS::EC2::SecurityGroup` |
| Parameter | `debug=true` |

The rule was scoped to the resource type, not exclusively to `LabSG1`. Other security groups in scope could also be evaluated and modified.

### Initial evaluation

The test changes were made before the custom rule existed. The initial evaluation still identified the drift and invoked the function.

Reviewed the evaluation timestamp, compliance results, and the annotation **“Permissions were modified.”**

## 5. Verify Remediation

Reopened `LabSG1` and compared its inbound permissions with the earlier configuration.

| Inbound Permission | Before | After |
| :--- | :--- | :--- |
| HTTP — TCP 80 | IPv4 | IPv4 and IPv6 |
| HTTPS — TCP 443 | IPv4 | IPv4 and IPv6 |
| SMTPS — TCP 465 | Allowed | Removed |
| IMAPS — TCP 993 | Allowed | Removed |

The function restored its predefined baseline rather than simply deleting the two extra ports. It also added the required IPv6 permissions.

**Security implication:** Automated remediation can add access as well as remove it. The baseline itself must be reviewed before enforcement.

## 6. Inspect the Function and Logs

### Function review

Reviewed the supplied Python function and identified:

- `boto3` for AWS API operations.
- `REQUIRED_PERMISSIONS` defining the desired inbound configuration.
- Logic to inspect, compare, revoke, and authorize permissions.
- Compliance reporting to AWS Config.
- Additional logging controlled by `debug=true`.

The enforced baseline is defined in the function code. Editing the Config rule description does not change it.

### Log verification

Opened the function's CloudWatch log group and searched across log streams for `revoking for`.

Reviewed entries showing the removal of TCP ports **465** and **993**, along with changes affecting other security groups in scope.

Verification combined three sources:

- **AWS Config:** Evaluation results.
- **EC2/VPC:** Actual inbound permissions.
- **CloudWatch Logs:** Function execution details.

---

## Results

| Objective | Outcome |
| :--- | :--- |
| Review service permissions | Inspected separate Config and Lambda roles |
| Enable resource recording | Configured continuous security group recording |
| Connect evaluation logic | Created the custom Lambda-backed Config rule |
| Detect existing drift | Initial evaluation processed the modified permissions |
| Restore the baseline | Removed SMTPS/IMAPS and restored HTTP/HTTPS permissions |
| Audit remediation | Reviewed matching CloudWatch log entries |

**Validation boundary:** The initial remediation was verified. A separate repeat-change test would provide additional evidence that later changes are corrected in the same way.

## Production Considerations

- **Limit scope:** Avoid applying one fixed baseline to unrelated security groups.
- **Review source ranges:** Port restrictions alone do not establish least privilege.
- **Check IPv6:** Include IPv6 exposure in the baseline review.
- **Restrict permissions:** Reduce the function's ability to modify unrelated resources.
- **Handle repeated evaluations:** Changes made by remediation can trigger further evaluations.
- **Plan recovery:** Consider partial updates, concurrent changes, and loss of administrative access.
- **Expect a delay:** Configuration recording and evaluation are asynchronous, not preventive controls.
- **Control logging:** Review debug output, retention, and access permissions.

## Key Takeaways

- Configuration recording, compliance evaluation, and remediation are distinct functions.
- IAM roles define what each service can do.
- An initial evaluation can detect drift that predates a rule.
- Resource scope determines the potential impact of automated remediation.
- A compliant resource matches the configured policy; that does not prove the policy is secure.
- Resource state and execution logs should support the reported compliance result.
