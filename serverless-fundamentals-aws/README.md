# Serverless Fundamentals on AWS
### AWS Lambda · Amazon DynamoDB · Amazon SQS

Reference documentation for hands-on serverless labs exploring function execution, database operations, and message-based application design.

> **Scope:** Lambda and DynamoDB were explored in the lab environment. SQS queue creation was blocked by sandbox permissions, so the queue configuration and integration workflow below are reference material—not a verified end-to-end deployment.

## Overview

These labs cover three responsibilities commonly found in serverless applications:

- **Compute:** Run application code with AWS Lambda.
- **Storage:** Store and retrieve items with Amazon DynamoDB.
- **Messaging:** Decouple producers and consumers with Amazon SQS.

Each service is documented separately. Their inclusion in this repository does not imply that a complete integrated application was deployed.

### What I worked on

- Configured Lambda using the sandbox's existing execution role.
- Used execution logs to investigate function behavior.
- Explored DynamoDB configuration and item operations.
- Documented an SQS-to-Lambda processing pattern.
- Identified IAM restrictions that prevented further integration testing.

**Configuration values:** Entries such as `<runtime>` are placeholders. Replace them with verified lab settings before presenting them as deployment details.

---

## Implementation Status

| Area | Status |
| :--- | :--- |
| Lambda execution role | Used the pre-provisioned `LabRole` |
| Lambda configuration | Lab configuration documented below |
| DynamoDB | Table settings and operations documented below |
| Custom IAM role creation | Restricted by the sandbox |
| SQS queue creation | Blocked in the sandbox |
| API Gateway API creation | Blocked in the sandbox |
| SQS → Lambda → DynamoDB integration | Reference design; not verified end to end |

## 1. AWS Lambda

Lambda provides the compute layer for running application code without managing an application server.

### Function configuration

| Setting | Lab Value |
| :--- | :--- |
| Function name | `<function-name>` |
| Runtime | `<runtime>` |
| Handler | `<handler>` |
| Memory | `<memory-mb>` MB |
| Timeout | `<timeout-seconds>` seconds |
| Execution role | `LabRole` |
| Trigger or invocation method | `<trigger>` |
| Expected log group | `/aws/lambda/<function-name>` |

### Execution and debugging

CloudWatch Logs is the primary place to inspect function output and execution errors when logging permissions and configuration are in place.

Useful evidence includes:

- Invocation timestamps.
- Application log messages.
- Error messages and stack traces.
- Execution duration.
- Returned test results.

A successful invocation should be checked against the intended application outcome. Successful execution alone does not prove that the expected database changes occurred.

### Execution-role boundary

The sandbox supplied `LabRole`; creating a custom role was not permitted.

This lab therefore does not claim that a dedicated least-privilege execution role was created.

---

## 2. Amazon DynamoDB

DynamoDB provides the data-storage layer.

### Table configuration

| Setting | Lab Value |
| :--- | :--- |
| Table name | `<table-name>` |
| Partition key | `<partition-key>` |
| Partition-key type | `<type>` |
| Sort key | `<sort-key or none>` |
| Sort-key type | `<type or not applicable>` |
| Capacity mode | `<on-demand or provisioned>` |

### Operations explored

| Operation | Purpose |
| :--- | :--- |
| `PutItem` | Create an item or replace an existing item with the same primary key |
| `GetItem` | Retrieve one item using its complete primary key |
| `Query` | Retrieve items using a partition-key value and an optional sort-key condition |
| `Scan` | Read items across the table rather than selecting them through a key condition |

### Access-pattern considerations

- Use `GetItem` when the complete primary key is known.
- Use `Query` when the key structure supports the required access pattern.
- Avoid treating `Scan` as the default lookup method.
- Check pagination before assuming that one response contains all matching items.

### Reference IAM permissions

For a function that only needs to write, retrieve, and query items, a starting permission set is:

| Action | Resource Scope |
| :--- | :--- |
| `dynamodb:PutItem` | Target table ARN |
| `dynamodb:GetItem` | Target table ARN |
| `dynamodb:Query` | Target table ARN; relevant index ARN if an index is queried |

**Table ARN format:** `arn:aws:dynamodb:<region>:<account-id>:table/<table-name>`

This is a reference permission set, not a policy claimed as attached during the lab.

Permissions should reflect the actual function behavior. For example, code using `Scan` would require an additional permission.

---

## 3. Amazon SQS — Reference Configuration

SQS provides a queue between a producer and a consumer.

**Implementation boundary:** Queue creation was blocked in the sandbox. The settings below are design references, not deployed resource values.

| Setting | Reference Value |
| :--- | :--- |
| Queue type | `<standard or FIFO>` |
| Visibility timeout | `<seconds>` |
| Message retention | `<period>` |
| Dead-letter queue | `<configured or not>` |
| Redrive threshold | `<maxReceiveCount, if configured>` |

### Lambda consumer behavior

In an SQS integration:

- An event source mapping polls the queue and invokes the function with message batches.
- Successfully processed batches can be removed from the queue.
- Failed processing can cause messages to become available again after the visibility timeout.
- A configured dead-letter queue can isolate messages that exceed the redrive threshold.
- Consumers should handle duplicate delivery safely.

Batch failure handling must be reviewed carefully. Without partial batch failure reporting, a failed batch can cause successfully processed messages in that batch to be retried as well.

### Visibility timeout

The reference guidance is to allow sufficient time for processing and retries.

For a Lambda consumer, use at least six times the function timeout as a starting point, with additional allowance for any configured batching window.

The correct setting still depends on the final integration configuration.

### Consumer permissions

The function's execution role typically needs:

| Permission | Purpose |
| :--- | :--- |
| `sqs:ReceiveMessage` | Receive messages |
| `sqs:DeleteMessage` | Remove processed messages |
| `sqs:GetQueueAttributes` | Read queue attributes |

Additional permissions may be required by the chosen encryption and resource-policy configuration.

These permissions were not validated through a working queue integration in this sandbox.

## Reference Integration Pattern

### Processing flow

1. A producer sends a message to SQS.
2. The Lambda event source mapping polls the queue.
3. Lambda processes the message.
4. The function writes the result to DynamoDB.
5. Execution details are sent to CloudWatch Logs when configured.
6. Failed messages follow the configured retry and dead-letter handling.

### Responsibilities

| Component | Responsibility |
| :--- | :--- |
| Producer | Submit work |
| SQS | Buffer messages and separate producers from consumers |
| Lambda | Validate and process messages |
| DynamoDB | Persist processing results |
| CloudWatch Logs | Support execution debugging |
| Dead-letter queue | Retain repeatedly unsuccessful messages, when configured |

> This workflow is an architectural reference. It was not deployed end-to-end in the restricted environment.

---

## Verification Checklist

### Lambda and DynamoDB

- Confirm the deployed function settings.
- Run a representative test event.
- Inspect the returned result and execution logs.
- Confirm the intended DynamoDB item changes.
- Test the required read operations.
- Record actual results rather than relying only on configuration screenshots.

### Future SQS integration

These checks remain pending until queue creation is available:

- Create the queue and any dead-letter queue.
- Configure the event source mapping.
- Verify consumer permissions.
- Send a test message.
- Confirm the expected database result.
- Test retries, duplicate handling, and failure isolation.


## Key Takeaways

- Separate compute, storage, and messaging responsibilities.
- Match database operations to the table's access patterns.
- Verify application outcomes alongside function execution.
- Treat IAM permissions as part of the integration design.
- Design queue consumers for retries and duplicate delivery.
- Distinguish deployed resources from reference architecture.
- Document environment restrictions without overstating implementation.
