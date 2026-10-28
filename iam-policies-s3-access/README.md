# IAM Policies and Roles for S3 Access

A hands-on exercise exploring how identity-based policies, role trust policies and S3 bucket policies affect access.

The exercise uses one IAM user, two IAM roles and three pre-created S3 buckets. Permissions are tested by attempting actions with different identities and comparing the results.

> **Scope:** The results below describe the lab environment. The complete policy JSON is not included here, so policy summaries should not be treated as a complete permissions audit.

## Environment

| Item | Details |
|---|---|
| Region | `us-east-1` (N. Virginia) |
| IAM user | `devuser`, member of `DeveloperGroup` |
| Group policy | `DeveloperGroupPolicy` |
| Primary IAM role | `BucketsAccessRole`, with a trust policy allowing `devuser` in the same account to assume it |
| Challenge IAM role | `OtherBucketAccessRole`, assumed by `devuser` for the final upload |
| Primary role policies | `ListAllBucketsPolicy` and `GrantBucket1Access` |
| Buckets | Three pre-created buckets, referred to here as `bucket1`, `bucket2` and `bucket3`. Their actual names contain these strings. |
| Test objects | `Image2.jpg` in `bucket1`; `Image1.jpg` in `bucket2` |

All identities and buckets used in this exercise are in the same AWS account. Bucket names in this document are shorthand, not literal deployment names.

## Access model

| Identity or resource | Policy | Type | Permissions described in the lab |
|---|---|---|---|
| `DeveloperGroup` | `DeveloperGroupPolicy` | Identity-based group policy | Selected IAM read actions and S3 bucket-level actions, including listing buckets and creating a bucket. No EC2 actions or S3 object-level actions are described. |
| `BucketsAccessRole` | `ListAllBucketsPolicy` | Identity-based role policy | `s3:ListAllMyBuckets` on `*` |
| `BucketsAccessRole` | `GrantBucket1Access` | Identity-based role policy | Selected get and list actions for `bucket1` and its objects. Does not grant `s3:PutObject`. |
| `bucket2` | Statement `S3Write` | Resource-based bucket policy | `s3:GetObject` and `s3:PutObject` for `BucketsAccessRole` |
| `bucket2` | Statement `ListBucket` | Resource-based bucket policy | `s3:ListBucket` for `BucketsAccessRole` |
| `bucket3` | Bucket policy | Resource-based bucket policy | Identifies `OtherBucketAccessRole` as an authorized upload principal |
| `BucketsAccessRole` | Trust policy | Resource-based role trust policy | Allows the named same-account user, `devuser`, to assume the role, subject to applicable conditions and restrictions |
| `OtherBucketAccessRole` | Trust and permission configuration | Role configuration | Allows `devuser` to assume this role in the challenge; the complete policy documents are not reproduced here |

**Policy detail to verify:** The reported results show that `BucketsAccessRole` can read the bucket policy on `bucket3`. This requires an effective permission for `s3:GetBucketPolicy` on that bucket. The policy summaries above do not identify which statement grants it; check the full policy documents before attributing that permission to a specific policy.

```text
devuser
  |
  +-- member of DeveloperGroup
  |     `-- DeveloperGroupPolicy
  |
  +-- assumes BucketsAccessRole
  |     +-- ListAllBucketsPolicy
  |     +-- GrantBucket1Access
  |     `-- bucket2 policy grants get, put and list access
  |
  `-- assumes OtherBucketAccessRole
        `-- bucket3 policy grants upload access
```

Requests without an applicable allow are implicitly denied. Explicit denies and other applicable policy restrictions can prevent access even when an allow exists.

## Results

The following table records the outcomes reported for the lab.

| # | Active identity | Action | Result | Explanation |
|---|---|---|---|---|
| 1 | `devuser` | List EC2 instances or attempt an instance launch | Not authorized | No applicable EC2 permissions are described in the user's policies. |
| 2 | `devuser` | Open the S3 bucket list | Three pre-created buckets visible; some console details show insufficient permissions | Listing bucket names does not grant access to their contents or every permission required by the console. |
| 3 | `devuser` | Open the IAM dashboard | `iam:GetAccountSummary` denied | The required action is not allowed, despite other IAM read permissions. |
| 4 | `devuser` | Create a new bucket in `us-east-1` | Allowed | The group policy grants `s3:CreateBucket` for the request. |
| 5 | `devuser` | Upload an object to the new bucket | Access Denied | No applicable `s3:PutObject` permission is described. |
| 6 | `devuser` | Download `Image2.jpg` from `bucket1` or `Image1.jpg` from `bucket2` | Access Denied | The group policy does not grant object reads, and the described `bucket2` grants target the role rather than the user. |
| 7 | `devuser` | Assume `BucketsAccessRole` | Allowed | The role's trust configuration permits the same-account user to assume it. |
| 8 | `BucketsAccessRole` session | Download `Image2.jpg` from `bucket1` | Allowed | `GrantBucket1Access` grants the required object-read permission. |
| 9 | `BucketsAccessRole` session | Open the IAM groups page | Denied | The role session lacks the required IAM permissions, including `iam:ListGroups`. |
| 10 | `devuser`, after returning from the role session | Open the IAM groups page | Allowed | Requests use the user's permissions again. |
| 11 | `BucketsAccessRole` session | Upload `Image2.jpg` to `bucket2` | Allowed | The bucket policy grants `s3:PutObject` to the role; the listed role policies do not grant access to `bucket2`. |

`Role` in these results means an assumed-role session, not the original IAM user.

## Challenge: uploading to bucket3

The objective is to upload `Image2.jpg`, previously downloaded from `bucket1`, to `bucket3`.

| Step | Active identity | Reported result |
|---|---|---|
| Attempt upload to `bucket3` | `devuser` | Fails |
| Attempt to read the `bucket3` policy | `devuser` | Denied |
| Read the `bucket3` policy | `BucketsAccessRole` session | Allowed |
| Upload to `bucket3` | `OtherBucketAccessRole` session | Succeeds |

The bucket policy identifies `OtherBucketAccessRole` as an authorized upload principal. Reading that policy does not itself grant upload permission to `BucketsAccessRole`.

The successful lab workflow is:

1. Download `Image2.jpg` from `bucket1` using `BucketsAccessRole`.
2. Read the policy on `bucket3` and identify the authorized upload role.
3. Switch to `OtherBucketAccessRole`, using the role-assumption access available to `devuser`.
4. Upload the downloaded file to `bucket3`.

There is no need to loosen the bucket policy or make the bucket public.

### Console switching versus role chaining

In the AWS Management Console, switching from one role to another uses the original sign-in identity to authorize the switch. Returning to `devuser` first can make the workflow clearer, but it is not a universal requirement.

Using temporary credentials from one role to call `sts:AssumeRole` for another role is **role chaining**. That is a different authorization path and requires the appropriate permissions and trust configuration.

## How the permissions combine

### 1. Same-account S3 access can come from either policy type

In this lab, access to `bucket1` comes from an identity-based role policy, while access to `bucket2` comes from a bucket policy.

For same-account S3 access, a bucket policy can grant access without a matching allow in the role's identity-based policies. However, this is not an unconditional “union of all permissions”: explicit denies, permissions boundaries, session policies and applicable AWS Organizations controls must also be considered. The precise evaluation depends on how the principal is specified.

### 2. Role permissions are not added to the user's permissions

Requests made with assumed-role credentials use the role session's effective permissions, not the combined permissions of the role and the original user.

The user does not permanently lose their permissions. Returning to the original identity causes subsequent requests to use that identity's permissions again.

### 3. Bucket-level and object-level permissions are separate

Creating a bucket does not automatically grant permission to upload objects.

Examples:

| Action | Resource scope |
|---|---|
| `s3:ListAllMyBuckets` | `*` |
| `s3:CreateBucket` | Bucket ARN |
| `s3:ListBucket` | Bucket ARN |
| `s3:GetBucketPolicy` | Bucket ARN |
| `s3:GetObject` | Object ARN |
| `s3:PutObject` | Object ARN |

A bucket ARN has the form `arn:aws:s3:::example-bucket`. An object scope covering all keys in that bucket has the form `arn:aws:s3:::example-bucket/*`.

### 4. Trust policies govern role assumption

A role's trust policy identifies principals permitted to assume the role and any conditions they must satisfy. Successful role assumption produces temporary credentials through AWS STS.

When a trust policy directly grants access to a specific IAM user in the same account, a separate identity-based allow for `sts:AssumeRole` is not always required. Applicable restrictions must still be satisfied.

The target role's S3 permissions are separate from its trust policy: being allowed to assume a role does not, by itself, grant access to every bucket.

### 5. Reading a policy is different from accessing objects

`s3:GetBucketPolicy` allows reading a bucket's policy. It does not grant `s3:GetObject` or `s3:PutObject`.

This explains why an identity can inspect the access rules without being able to upload.

### 6. CloudTrail coverage must be configured

Do not assume that every upload and download appears automatically in CloudTrail Event history.

S3 object operations such as `GetObject` and `PutObject` are data events. They require explicit data-event logging configuration and are not shown in the standard Event history.

For this exercise, configure the required logging before testing if CloudTrail records are intended to serve as evidence.

## Takeaways

- Apply least privilege to actions, resources and policy conditions.
- For IAM users in a lab, use groups to manage shared identity-based permissions rather than duplicating policies on individual users.
- Use temporary role sessions for access needed only for a particular task.
- Review identity-based policies, bucket policies, trust policies and applicable restrictions together.
- Distinguish account-wide bucket listing, bucket-level access and object-level access.
- A bucket policy can identify an authorized role, but the user must also be permitted to assume that role.
- An access-denied result is not a reason to make a bucket public or broaden its policy unnecessarily.
- Configure S3 data-event logging when upload and download activity must be audited.
- Use the AWS Service Authorization Reference to check action names and supported resource types.
