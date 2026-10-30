# Security Monitoring with CloudTrail, CloudWatch and EventBridge

A hands-on AWS monitoring lab covering API auditing, event-driven security notifications, failed console sign-in detection, and log analysis.

Completed as part of AWS Academy Cloud Architecting — Lab 6.1.

## Overview

This lab builds two notification workflows:

1. An EventBridge rule routes selected EC2 API events to an SNS topic.
2. A CloudWatch alarm sends a notification when a custom metric records at least three failed console sign-ins during a five-minute evaluation period.

CloudTrail provides the audit events, while CloudWatch Logs Insights supports investigation of the recorded login failures.

> **Scope:** This is a controlled lab, not a complete security-monitoring deployment. The event pattern covers selected API operations, and the failed-login metric counts matching events across the monitored log stream rather than identifying every possible authentication threat.

## Environment

| Resource | Details |
|---|---|
| EC2 instance | Preconfigured `LabInstance` |
| Security group | Associated group whose name contains `LabSecurityGroup` |
| Existing CloudTrail trail | `LabCloudTrail` |
| CloudWatch log group | `CloudTrailLogGroup` |
| CloudTrail delivery role | `LabCloudTrailRole` |
| SNS topic | `MySNSTopic`, Standard |
| SNS subscription | Email subscription, confirmed by the recipient |
| EventBridge rule | `MonitorSecurityGroups`, on the default event bus |
| Metric filter | `ConsoleLoginErrors` |
| Metric namespace | `CloudTrailMetrics` |
| Metric name | `ConsoleLoginFailureCount` |
| CloudWatch alarm | `FailedLogins` |
| Test identity | Preconfigured IAM user `test` |


## Implementation

### 1. Inspect CloudTrail events and trail configuration

Reviewed a CloudFormation `CreateStack` event in CloudTrail Event history.

Examined fields including:

- `userIdentity`
- `eventTime`
- `eventSource`
- `eventName`
- `awsRegion`
- `requestParameters`

Then reviewed the workflow for creating `MyLabCloudTrail` with:

- S3 log storage.
- CloudWatch Logs delivery.
- Management events.
- Read and write API activity.
- The existing `LabCloudTrailRole`.

**Lab restriction:** The creation workflow was cancelled because the lab identity did not have the required permissions to complete it.

The remaining tasks used the preconfigured `LabCloudTrail`, which already delivered events to `CloudTrailLogGroup`.

This lab therefore demonstrates reviewing the trail-creation workflow and using an existing trail—not successfully creating an additional trail.

### 2. Create an SNS notification channel

Created a Standard topic named `MySNSTopic`.

Added an email subscription and confirmed it through the subscription-confirmation message.

An unconfirmed subscription does not provide a working email notification path.

> The lab instructions select “Everyone” for publishing and subscribing. This is an intentionally permissive lab configuration, not a production recommendation. Outside the lab, scope topic access to the required principals and services.

### 3. Configure EventBridge monitoring

Created `MonitorSecurityGroups` on the default event bus with the following event pattern:

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["ec2.amazonaws.com"],
    "eventName": [
      "AuthorizeSecurityGroupIngress",
      "ModifyNetworkInterfaceAttribute"
    ]
  }
}
```

Configured `MySNSTopic` as the target and an input transformer to format the notification.

#### What this pattern actually monitors

| API operation | Meaning |
|---|---|
| `AuthorizeSecurityGroupIngress` | Adds inbound permissions to a security group |
| `ModifyNetworkInterfaceAttribute` | Changes a network-interface attribute; this can include security-group associations, but is not limited to them |

This pattern does **not** monitor every security-group change. For example, it does not include inbound-rule revocation, outbound-rule changes or all rule-modification operations.

It also does not check whether a change is malicious or whether it opens a particular port. It alerts on matching API activity.

#### Input-transformer limitation

The lab transformer uses:

```json
{
  "name": "$.detail.requestParameters.groupId",
  "source": "$.detail.eventName",
  "time": "$.time",
  "value": "$.detail"
}
```

The `name` variable is a security-group ID, not a display name.

Also, `requestParameters.groupId` is not guaranteed to exist for every matched operation. In particular, network-interface modification events have a different request structure.

For a broader deployment, use event-specific formatting or a generic message that does not assume every event contains the same resource fields.

### 4. Test the security notification

Added an inbound SSH rule to the lab security group:

```text
Protocol: TCP
Port: 22
Source: 0.0.0.0/0
```

The validation workflow was:

1. Find the corresponding `AuthorizeSecurityGroupIngress` event in CloudTrail.
2. Confirm that its permission details refer to port 22.
3. Verify that the event matches the EventBridge pattern.
4. Check the subscribed email inbox for the SNS notification.

This tests the notification path for adding an inbound rule. It does not demonstrate coverage of every security-group operation.

**Security note:** Opening SSH to `0.0.0.0/0` was a test action. Remove the rule after testing unless the lab requires it to remain.

### 5. Create a failed-login metric filter

Created a filter on `CloudTrailLogGroup`:

```text
{ ($.eventName = ConsoleLogin) && ($.errorMessage = "Failed authentication") }
```

Configured the metric transformation:

| Setting | Value |
|---|---|
| Filter name | `ConsoleLoginErrors` |
| Namespace | `CloudTrailMetrics` |
| Metric name | `ConsoleLoginFailureCount` |
| Metric value | `1` |
| Dimensions | None |

Each matching event contributes `1` to the metric.

The filter applies to newly ingested matching log events; it does not retroactively create metric data from all historical records.

### 6. Configure the CloudWatch alarm

Configured `FailedLogins` with:

| Setting | Value |
|---|---|
| Metric | `CloudTrailMetrics / ConsoleLoginFailureCount` |
| Statistic | Sum |
| Period | 5 minutes / 300 seconds |
| Threshold | Greater than or equal to 3 |
| Evaluation | 1 datapoint out of 1 |
| Alarm action | Publish to `MySNSTopic` |

This detects three or more matching failures within a five-minute evaluation period.

Because the metric has no dimensions, it aggregates matching events across the monitored logs. It does **not** require all failures to come from the same user or source IP.

This is a period-based alarm, not an exact rolling five-minute counter.

### 7. Generate controlled login failures

Used the preconfigured `test` user to attempt console sign-in with an incorrect password at least three times.

After testing, re-established the lab's administrative console session.

Validation included:

- Checking the custom metric.
- Inspecting the alarm state and alarm history.
- Checking the subscribed inbox for the alarm notification.

Log delivery and alarm evaluation are asynchronous, so the notification may not appear immediately.

An SNS alarm notification is associated with an alarm state transition; it is not a separate email for every failed sign-in.

### 8. Investigate failures with Logs Insights

Selected `CloudTrailLogGroup` and ran:

```sql
filter eventSource = "signin.amazonaws.com"
  and eventName = "ConsoleLogin"
  and responseElements.ConsoleLogin = "Failure"
| stats count(*) as Total_Count
  by sourceIPAddress as Source_IP,
     errorMessage as Reason,
     awsRegion as AWS_Region,
     userIdentity.arn as IAM_Arn
```

The query groups failed sign-ins by:

- Source IP address.
- Recorded failure reason.
- AWS Region.
- IAM ARN, when available in the event.

Set the query time range to include the test attempts.

The query and metric filter use different matching fields. Don't assume their counts are identical, especially if the query time range differs from the alarm period.

## Verification Checklist

| Component | Evidence to check |
|---|---|
| Existing trail | `LabCloudTrail` is logging and delivering to the expected destinations |
| SNS subscription | Subscription is confirmed |
| EventBridge rule | Correct event pattern and SNS target |
| Security-group test | Port-22 authorization event visible in CloudTrail |
| Event notification | Email contains the matching API activity |
| Metric filter | Correct log group, pattern, and metric transformation |
| Alarm | Sum ≥ 3 over a 300-second period |
| Login test | Matching failure events reach CloudWatch Logs |
| Alarm notification | Alarm history and email show the transition |
| Investigation | Logs Insights returns the relevant failure groups |

## Security and Operational Considerations

- Restrict SNS publishing and subscription permissions rather than retaining the lab's broad access.
- Ensure EventBridge and CloudWatch have the permissions required to publish notifications.
- Review regional coverage; a rule on one regional event bus is not automatically account-wide monitoring.
- Confirm where console sign-in events are recorded and whether the trail delivers them to the monitored log group.
- Extend the event pattern if broader security-group change coverage is required.
- Consider distinguishing failed API attempts from successful configuration changes.
- Configure log retention and S3 lifecycle policies deliberately.
- Decide how the alarm should treat missing data.
- Use supported CloudTrail log encryption settings appropriate to the environment. Disabling the SSE-KMS option does not mean that S3 objects are stored without server-side encryption.
- Redact email addresses, account IDs, and sensitive event details from published evidence.
- Remove temporary test access after validation.

## Key Takeaways

- CloudTrail provides audit records; EventBridge routes selected events; CloudWatch evaluates metrics and alarms.
- A preconfigured resource should be distinguished from one created during the lab.
- An event pattern only covers the operations and conditions it explicitly matches.
- Notification delivery requires both correct permissions and a confirmed subscription.
- Metric filters convert matching log events into numerical signals.
- Alarm statistics, periods, and dimensions determine what a threshold actually means.
- Logs Insights supports investigation beyond the information in an alert.
