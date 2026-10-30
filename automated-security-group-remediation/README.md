# Automated Security Group Remediation with AWS Config and Lambda

A hands-on AWS security lab covering configuration monitoring, custom compliance evaluation, and automated remediation of EC2 security group inbound rules.

Completed as part of AWS Academy Cloud Architecting — Lab 7.1.

## Overview

This lab uses AWS Config to record security group configuration changes and invoke a preconfigured Lambda function through a custom rule.

The function compares inbound permissions against a defined baseline and modifies security groups to restore the expected configuration.

CloudWatch Logs provides evidence of the remediation actions.

> **Scope:** This is a controlled lab using a fixed HTTP/HTTPS baseline. It is not a production-ready policy for every security group. Applying the same baseline broadly can remove legitimate access or introduce unwanted permissions.

## Environment

| Resource | Details |
|---|---|
| Custom VPC | `Lab VPC` |
| Test security group | `LabSG1` |
| Additional resources | Default security groups included in the rule's scope |
| AWS Config recording | Continuous recording of `AWS::EC2::SecurityGroup` |
| Config service role | Preconfigured `AwsConfigRole`, updated during the lab |
| Lambda execution role | Preconfigured `AwsConfigLambdaSGRole` |
| Lambda function | Preconfigured `awsconfig_lambda_security_group` |
| Custom Config rule | `EC2SecurityGroup` |
| Rule parameter | `debug=true` |
| Configuration delivery | S3 delivery channel |
| Verification | AWS Config evaluations, security group rules and CloudWatch Logs |

The Lambda function and IAM roles were supplied by the lab. The work focused on inspecting permissions, configuring AWS Config, connecting the custom rule to the function, and verifying remediation.

#
```

The remediation is implemented by the Lambda function's code. This lab does not configure a separate AWS Config remediation action or Systems Manager Automation runbook.

## Implementation

### 1. Review service roles and permissions

Inspected two roles with different responsibilities.

#### Lambda execution role

`AwsConfigLambdaSGRole` includes permissions for:

| Permission | Purpose |
|---|---|
| `ec2:DescribeSecurityGroups` | Inspect current security group settings |
| `ec2:AuthorizeSecurityGroupIngress` | Add required inbound permissions |
| `ec2:RevokeSecurityGroupIngress` | Remove unwanted inbound permissions |
| `config:PutEvaluations` | Submit compliance evaluations |
| `logs:CreateLogGroup` | Create a log group where required |
| `logs:CreateLogStream` | Create execution log streams |
| `logs:PutLogEvents` | Write execution details |

The supplied policy uses broad resource scopes. This is a lab configuration, not a least-privilege production policy.

#### AWS Config role

Reviewed the existing `S3Access` policy on `AwsConfigRole` and attached the AWS managed policy:

```text
AWS_ConfigRole
```

This role supports AWS Config resource discovery, configuration recording and delivery to the configured destination.

**Delivery distinction:** AWS Config writes configuration history and snapshots to its S3 delivery destination. These are not CloudWatch log files.

The Config service role and Lambda execution role are separate from the permission allowing AWS Config to invoke the Lambda function.

### 2. Enable configuration recording

Configured AWS Config with:

| Setting | Value |
|---|---|
| Recording strategy | Specific resource types |
| Resource type | `AWS::EC2::SecurityGroup` |
| Recording frequency | Continuous |
| IAM role | `AwsConfigRole` |
| Delivery channel | S3 destination configured during setup |

Reviewed the resource inventory after discovery.

The selected recording scope is security groups in the configured Region. The presence of related resources in an inventory view should not be treated as proof that every resource type is continuously recorded.

### 3. Introduce configuration drift

Modified `LabSG1` to permit the following inbound IPv4 traffic:

| Traffic | TCP port | Source |
|---|---|---|
| HTTP | 80 | `0.0.0.0/0` |
| HTTPS | 443 | `0.0.0.0/0` |
| SMTPS | 465 | `0.0.0.0/0` |
| IMAPS | 993 | `0.0.0.0/0` |

Ports 465 and 993 were outside the lab's desired baseline.

This simulated configuration drift. The presence of those ports is not inherently evidence of a real security incident; the violation is relative to the policy defined for this exercise.

### 4. Create the custom Config rule

Created a custom Lambda rule with:

| Setting | Value |
|---|---|
| Name | `EC2SecurityGroup` |
| Description | Restrict inbound ports to HTTP and HTTPS |
| Lambda function | ARN of the supplied function |
| Trigger | Configuration changes |
| Resource scope | `AWS::EC2::SecurityGroup` |
| Parameter | `debug=true` |

The rule was scoped by resource type, not restricted to `LabSG1`.

This means other security groups in scope, including default security groups, could also be evaluated and modified by the function.

### 5. Observe the initial evaluation

Waited for the initial evaluation to complete and inspected:

- The last successful evaluation timestamp.
- Resources in scope.
- Compliance results.
- Evaluation annotations.

The lab reported resources as `Compliant` with an annotation such as:

```text
Permissions were modified
```

The unwanted rules were added before the custom rule existed. The initial evaluation still detected the drift and invoked the function.

A `Compliant` result here reflects the function's reported evaluation. It should be checked alongside the actual resource configuration and execution logs.

### 6. Verify the repaired security group

Reopened `LabSG1` and compared its inbound rules with the earlier configuration.

| Configuration | Before remediation | After remediation |
|---|---|---|
| HTTP, TCP 80 | IPv4 | IPv4 and IPv6 |
| HTTPS, TCP 443 | IPv4 | IPv4 and IPv6 |
| SMTPS, TCP 465 | Allowed | Removed |
| IMAPS, TCP 993 | Allowed | Removed |

The function did more than remove the two extra ports: it restored its predefined HTTP/HTTPS permissions, including IPv6 rules.

> Allowing HTTP and HTTPS does not automatically make a configuration secure. Source ranges, workload requirements, and IPv6 exposure still require review.

### 7. Inspect the Lambda logic

Reviewed the supplied Python function.

Key elements included:

- `boto3` for AWS API operations.
- `REQUIRED_PERMISSIONS` defining the desired inbound permissions.
- Security group inspection.
- Comparison of current and required permissions.
- Revocation and authorization operations.
- Compliance reporting.
- Additional logging controlled by the `debug` parameter.

The desired configuration is defined in the function code. Changing the Config rule's description alone does not change the enforced baseline.

### 8. Audit remediation through CloudWatch Logs

Opened the Lambda function's CloudWatch log group and searched across log streams for:

```text
revoking for
```

Reviewed entries showing the permissions being revoked, including those for:

- TCP 465.
- TCP 993.

Other log entries related to additional security groups in the rule's scope.

Combined three sources of evidence:

1. AWS Config evaluation results.
2. Actual security group inbound rules.
3. Lambda execution logs.

A log entry describing an attempted action should be checked against the resulting resource state before concluding that remediation succeeded.

## Verification Summary

| Check | Evidence |
|---|---|
| Config role updated | `AWS_ConfigRole` attached to `AwsConfigRole` |
| Recording enabled | Continuous security group recording |
| Drift introduced | Ports 80, 443, 465, and 993 initially permitted |
| Custom rule configured | `EC2SecurityGroup` references the supplied Lambda |
| Initial evaluation completed | Timestamp and compliance results available |
| Unexpected permissions removed | Ports 465 and 993 no longer present |
| Baseline restored | HTTP/HTTPS permissions match the function's baseline |
| Logs reviewed | Revocation details visible in Lambda logs |
| Scope reviewed | Other security groups also included in evaluation |

The initial remediation was the validation performed here. A repeat-change test can additionally confirm that later configuration changes trigger the same behavior.

## Security and Operational Considerations

### Limit the remediation scope

Applying one fixed baseline to all security groups can disrupt unrelated workloads.

For production, define explicit eligibility criteria, such as approved resources or tags, and validate the function's handling of out-of-scope resources.

### Review the baseline itself

A successfully enforced baseline is not necessarily a secure baseline.

Review:

- Allowed ports.
- Source CIDRs.
- IPv4 and IPv6 exposure.
- Required administration paths.
- Application-specific dependencies.
- Default security group behavior.

### Separate evaluation from remediation where appropriate

This lab combines both in one function.

A production design may separate compliance detection from remediation to support approvals, exceptions, controlled retries, and clearer operational ownership.

### Design for repeated invocations

Remediation creates additional configuration changes, which can lead to further evaluations.

The function should safely handle a resource that already matches the baseline and avoid unnecessary updates.

### Use least-privilege permissions

Scope mutation permissions where supported and restrict which resources the function may modify.

Review both service trust relationships and Lambda invocation permissions.

### Plan for failures and recovery

Consider:

- Partial updates.
- Concurrent administrator changes.
- API errors and throttling.
- Lost administrative access.
- A tested rollback or emergency-access procedure.

Configuration recording and evaluation are asynchronous. Automated remediation reduces exposure time but does not prevent the original change from taking effect.

### Manage debug logging

`debug=true` helps explain lab behavior. In production, review log contents, retention, and access before enabling verbose output.

## Key Takeaways

- AWS Config recording, compliance evaluation, and remediation are distinct responsibilities.
- IAM service roles determine what each service can do on behalf of the account.
- A custom Lambda rule can include code that modifies resources, but that behavior must be explicitly implemented.
- Initial evaluation can identify drift that predates the rule.
- Rule scope determines the potential impact of automated changes.
- Restoring a baseline can add permissions as well as remove them.
- Compliance results should be verified against resource state and execution evidence.
