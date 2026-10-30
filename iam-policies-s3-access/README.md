# IAM Policies and Roles for S3 Access
### Identity Policies · Role Trust · Bucket Policies

A hands-on AWS security exercise exploring how different identities receive access to S3 resources through group policies, assumed roles, and bucket policies.

## Overview

This lab uses one IAM user, two IAM roles, and three pre-created S3 buckets to compare effective permissions.

I tested actions with each identity, examined access-denied results, and used bucket policies to identify the appropriate role for an upload task.

### What I worked on

- Examined an IAM group policy.
- Compared user permissions with assumed-role permissions.
- Tested bucket-level and object-level S3 actions.
- Reviewed resource-based access through bucket policies.
- Switched roles to complete a restricted upload.

> **Scope:** These results describe the lab environment. Complete policy documents are not included, so the summaries below are not a full permissions audit.

---

## Lab Environment

| Component | Resource |
| :--- | :--- |
| Region | `us-east-1` — N. Virginia |
| IAM user | `devuser` |
| IAM group | `DeveloperGroup` |
| Group policy | `DeveloperGroupPolicy` |
| Primary role | `BucketsAccessRole` |
| Challenge role | `OtherBucketAccessRole` |
| Primary role policies | `ListAllBucketsPolicy`, `GrantBucket1Access` |
| S3 resources | Three pre-created buckets |
| Test objects | `Image2.jpg` in `bucket1`; `Image1.jpg` in `bucket2` |

All identities and buckets are in the same account.

**Naming:** `bucket1`, `bucket2`, and `bucket3` are shorthand for the lab's actual bucket names.

## Access Model

### IAM user

`devuser` receives permissions through membership in `DeveloperGroup`.

The group policy provides selected IAM read actions and S3 bucket-level permissions, including listing buckets and creating a bucket. It does not provide the EC2 or S3 object-level permissions tested in this exercise.

### Primary role

`BucketsAccessRole` provides a different permission set through its role policies and applicable bucket policies.

| Permission source | Access described in the lab |
| :--- | :--- |
| `ListAllBucketsPolicy` | List bucket names |
| `GrantBucket1Access` | Selected read and list access to `bucket1` |
| `bucket2` policy | Get objects, put objects, and list the bucket |

The role's trust policy permits `devuser` to assume it, subject to applicable conditions and restrictions.

### Challenge role

The `bucket3` policy identifies `OtherBucketAccessRole` as an authorized upload principal.

The lab also provides the role-assumption access needed for `devuser` to use that role.

> **Permission detail to verify:** `BucketsAccessRole` could read the policy on `bucket3`. The policy summaries supplied here do not identify the statement granting `s3:GetBucketPolicy`; check the full policies before attributing that access to a specific source.

---

## 1. Test the User's Permissions

Performed the following actions as `devuser`, without an assumed role.

| Action | Result |
| :--- | :--- |
| List EC2 instances or attempt an instance launch | Denied |
| View the S3 bucket list | Allowed, with some console details unavailable |
| Open the IAM dashboard | `iam:GetAccountSummary` denied |
| Create a bucket in `us-east-1` | Allowed |
| Upload an object to the new bucket | Denied |
| Download `Image2.jpg` from `bucket1` | Denied |
| Download `Image1.jpg` from `bucket2` | Denied |

### What these results show

- Listing bucket names does not grant access to their objects.
- Creating a bucket does not automatically authorize uploads.
- Selected IAM read permissions do not provide access to every IAM console page.
- A bucket policy granting access to a role does not automatically grant the same access to the original user.

## 2. Test the Primary Role

Assumed `BucketsAccessRole` and repeated selected operations.

| Action | Result | Permission source |
| :--- | :--- | :--- |
| Download `Image2.jpg` from `bucket1` | Allowed | Role identity policy |
| Open the IAM groups page | Denied | Required IAM permissions absent |
| Upload `Image2.jpg` to `bucket2` | Allowed | Bucket policy |

After returning to `devuser`, the IAM groups page was accessible again.

### Why the results changed

Requests made using assumed-role credentials use the role session's effective permissions.

The user's group permissions are not added to the role's permissions. Returning to the user identity causes subsequent requests to use the user's permissions again.

## 3. Complete the Bucket3 Challenge

**Objective:** Upload the file downloaded from `bucket1` to `bucket3`.

| Active identity | Attempt | Result |
| :--- | :--- | :--- |
| `devuser` | Upload to `bucket3` | Denied |
| `devuser` | Read the bucket policy | Denied |
| `BucketsAccessRole` | Read the bucket policy | Allowed |
| `OtherBucketAccessRole` | Upload to `bucket3` | Allowed |

### Workflow

1. Downloaded `Image2.jpg` using `BucketsAccessRole`.
2. Read the policy on `bucket3`.
3. Identified `OtherBucketAccessRole` as the authorized upload principal.
4. Switched to that role using the access available to `devuser`.
5. Uploaded the file successfully.

No public-access changes or broader bucket permissions were needed.

**Key distinction:** Permission to read a bucket policy does not grant permission to upload objects.

---

## How the Permissions Work

### Identity-based and resource-based permissions

In this lab:

- Access to `bucket1` came from an identity-based role policy.
- Access to `bucket2` came from a resource-based bucket policy.

For same-account access, a resource-based policy can grant access without a matching identity-policy allow. Effective access still depends on explicit denies and other applicable restrictions.

Do not reduce policy evaluation to an unconditional union of all permissions. Permissions boundaries, session policies, and AWS Organizations controls may also affect the result.

### Bucket actions and object actions

| Action | Resource scope |
| :--- | :--- |
| `s3:ListAllMyBuckets` | `*` |
| `s3:CreateBucket` | Bucket ARN |
| `s3:ListBucket` | Bucket ARN |
| `s3:GetBucketPolicy` | Bucket ARN |
| `s3:GetObject` | Object ARN |
| `s3:PutObject` | Object ARN |

**Bucket example:** `arn:aws:s3:::example-bucket`

**All objects in that bucket:** `arn:aws:s3:::example-bucket/*`

A bucket-level permission does not automatically include object-level access.

### Trust and resource access are separate

A role trust policy governs who may assume the role.

The role's effective permissions govern what the resulting session may do.

When a trust policy directly grants access to a specific IAM user in the same account, a separate identity-based allow for `sts:AssumeRole` is not always required. Applicable restrictions still matter.

### Console switching and role chaining

These are different workflows:

- **Console role switching:** Uses the original sign-in identity to authorize switching roles.
- **Role chaining:** Uses one role's temporary credentials to assume another role.

Returning to `devuser` first can clarify the console workflow, but it is not a universal requirement for every role switch.

## Audit Considerations

S3 object reads and writes are CloudTrail data events.

Do not assume that `GetObject` and `PutObject` operations appear automatically in CloudTrail Event history. Configure the required data-event logging if object-access records are needed as evidence.

**Validation boundary:** The evidence described in this exercise is based on action results. CloudTrail data-event logging is not claimed as configured.

---

## Results

| Objective | Outcome |
| :--- | :--- |
| Compare user and role access | Different permission sets demonstrated |
| Distinguish bucket and object permissions | Bucket creation succeeded while upload failed |
| Test identity-based access | Download from `bucket1` succeeded using the primary role |
| Test bucket-policy access | Upload to `bucket2` succeeded using the primary role |
| Inspect restricted access | Read the `bucket3` policy using the permitted identity |
| Complete the challenge | Uploaded using `OtherBucketAccessRole` |

## Key Takeaways

- Review identity policies, bucket policies, and role trust together.
- Bucket listing, bucket administration, and object access are distinct permissions.
- Assumed-role permissions are not combined with the original user's group permissions.
- Reading an access policy is different from being authorized to use the resource.
- Use an authorized identity rather than broadening access unnecessarily.
- Scope permissions to the required actions and resources.
- Configure object-level audit logging deliberately.
