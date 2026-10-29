# Encrypting Data at Rest with AWS KMS

A hands-on AWS security lab covering customer managed encryption keys, Amazon S3 server-side encryption, encrypted EBS root volumes, and CloudTrail auditing.

Completed as part of AWS Academy Cloud Architecting — Lab 5.1.

## Overview

The lab starts with an S3 bucket and an EC2 instance using an unencrypted root EBS volume.

A customer managed AWS KMS key is then used to:

- Encrypt an S3 object with SSE-KMS.
- Compare anonymous and authenticated object access.
- Replace the existing EC2 root volume with an encrypted volume.
- Examine related API activity in CloudTrail.
- Test how disabling and re-enabling the key affects dependent resources.

> **Scope:** This is a controlled lab. Public-access settings were temporarily changed to demonstrate the behavior of an SSE-KMS object. Those changes are not recommended as a production access pattern.

## Environment

| Resource | Details |
|---|---|
| AWS KMS key | Symmetric customer managed key |
| Key alias | `alias/MyKMSKey` |
| Lab role | `voclabs`, selected as both key administrator and key user |
| S3 bucket | Pre-created bucket whose name contains `imagebucket` |
| Test object | `clock.png` |
| EC2 instance | `LabInstance` |
| Original storage | Unencrypted root EBS volume |
| Replacement storage | EBS volume encrypted using `MyKMSKey` |
| Auditing | AWS CloudTrail Event history |

Resource names are lab labels. Actual bucket names, resource IDs and Availability Zones depend on the provisioned environment.



The KMS key protects data keys used by the integrated services. It is not used to send the entire image or EBS volume through KMS for encryption.

## Implementation

### 1. Create a customer managed KMS key

Created a symmetric encryption key with the alias `MyKMSKey`.

The lab role, `voclabs`, was selected for:

- Key administration.
- Cryptographic key usage.

These are separate permission responsibilities. A production design should review whether the same principal needs both.

### 2. Upload an S3 object using SSE-KMS

Uploaded `clock.png` to the lab bucket and explicitly selected:

```text
Encryption type: SSE-KMS
KMS key: MyKMSKey
```

Verified the object's server-side encryption settings after the upload.

**Important distinction:** New S3 uploads are already encrypted by default. This task specifically selected a customer managed KMS key instead of relying on the bucket's existing default encryption configuration.

Selecting SSE-KMS for this upload does not, by itself, change the bucket's default encryption settings or re-encrypt other objects.

### Envelope encryption

The simplified SSE-KMS workflow is:

1. S3 requests a data key from KMS.
2. KMS returns a plaintext data key and an encrypted copy protected by the KMS key.
3. S3 encrypts the object using the plaintext data key.
4. S3 stores the encrypted object and encrypted data key.
5. S3 removes the plaintext data key from memory after use.

For an authorized read, S3 obtains the data key needed to decrypt the object.

S3 Bucket Keys can change the frequency of KMS requests, so a production setup should not assume one KMS API call for every object operation.

### 3. Test anonymous object access

Attempted to open the object's direct S3 URL in a browser.

| Configuration | Lab result |
|---|---|
| Initial public-access restrictions | `AccessDenied` |
| Public-access restrictions relaxed and public-read object ACL applied | Signature-related error; image still not displayed |

The lab temporarily disabled the bucket's Block Public Access settings and enabled ACLs to demonstrate the second case.

SSE-KMS objects do not support anonymous access. Requests must be authenticated using AWS Signature Version 4 and satisfy the applicable S3 and KMS authorization requirements.

**A public-read ACL does not grant KMS decryption permission.**

> Encryption is an additional control, not a substitute for keeping data private and configuring access permissions correctly.

### 4. Test signed access

Selected the object in the S3 console and used **Open**.

The console-generated signed URL successfully displayed `clock.png` while the key was enabled and the signing identity had the required permissions.

This access depends on:

- Permission to read the S3 object.
- Permission to use the KMS key for decryption.
- A valid signed request.
- An available, enabled KMS key.

A signed URL is not the same as making an object public. Treat presigned URLs as sensitive temporary access links and do not commit them to the repository.

### 5. Review KMS activity in CloudTrail

Filtered CloudTrail Event history by:

```text
Event source: kms.amazonaws.com
```

Reviewed the relevant lab events:

| Event | What was examined |
|---|---|
| `GenerateDataKey` | Data-key generation associated with the encrypted upload |
| `Decrypt` | Key usage associated with reading the encrypted object |

Useful fields include:

- `eventTime`
- `eventName`
- `userIdentity`
- `requestParameters`
- `resources`
- `errorCode` and `errorMessage`, when present

The encryption context can help relate a KMS request to an S3 resource. Its exact contents depend on the service configuration, including whether S3 Bucket Keys are enabled.

**Logging distinction:** CloudTrail Event history provides the last 90 days of management events for the selected Region. Seeing a KMS event is not the same as having S3 object-level data-event logging enabled.

Configure the relevant S3 data events separately when object reads and writes must be audited directly.

### 6. Replace the unencrypted EC2 root volume

The existing volume was not encrypted in place. A new encrypted volume was created from a snapshot and attached as the replacement root volume.



The workflow was:

1. Stop `LabInstance`.
2. Record the original root device name and Availability Zone.
3. Create a snapshot of the original root volume.
4. Wait for the snapshot to complete.
5. Create an encrypted volume from that snapshot using `MyKMSKey`.
6. Place the replacement volume in the same Availability Zone as the instance.
7. Label the original and replacement volumes.
8. Detach the original root volume.
9. Attach the encrypted replacement using the expected root device name.
10. Verify the encryption status and KMS key reference.

In this lab, the root attachment device name was:

```text
/dev/xvda
```

Do not assume that this device name is correct for every instance.

**Security consideration:** The original volume and snapshot remain unencrypted copies. Creating an encrypted replacement does not encrypt or remove them.

### 7. Disable the key and test dependent resources

Disabled `MyKMSKey` while `LabInstance` was stopped.

| Test | Lab result |
|---|---|
| Start the instance with the encrypted root volume | Instance returned from `Pending` to `Stopped` |
| Open `clock.png` using the previously successful signed-access method | Request failed with `KMS.DisabledException` |

Reviewed the related CloudTrail events:

| Event | Significance |
|---|---|
| `DisableKey` | Records the key-state change |
| `StartInstances` | Records the request to start the instance |
| `CreateGrant` | The lab failure included a KMS error indicating that the key was disabled |

An accepted `StartInstances` request does not prove that the instance successfully reached the `Running` state.

`CreateGrant` is a grant-management operation, not itself a decryption operation. Its failure can prevent the encrypted-volume startup workflow from completing.

### 8. Re-enable the key

Re-enabled `MyKMSKey` and started `LabInstance` again.

The instance successfully reached the `Running` state.

Re-enabling a key restores its availability for permitted operations; it does not bypass S3 permissions, IAM policies or the key policy.

## Verification Summary

| Check | Expected lab outcome |
|---|---|
| KMS key created | Symmetric customer managed key with alias `MyKMSKey` |
| S3 object encryption | `clock.png` shows SSE-KMS with the selected key |
| Anonymous access with initial restrictions | Denied |
| Anonymous access after public-read changes | Still cannot retrieve the SSE-KMS object |
| Authorized signed access with key enabled | Image opens |
| CloudTrail review | Relevant KMS events identified |
| Replacement EBS root volume | Encrypted and associated with the selected key |
| Start instance with key disabled | Startup fails |
| Signed object read with key disabled | Fails in the lab |
| Start instance after key is re-enabled | Instance reaches `Running` |

## Security and Operational Considerations

- Keep S3 Block Public Access enabled outside the specific lab test.
- Avoid enabling ACLs solely to share objects; use an appropriate private-access mechanism.
- Scope S3 and KMS permissions to the required principals and resources.
- Separate key administration and key usage where appropriate.
- Treat key availability as a dependency of encrypted workloads.
- Do not assume disabling a key instantly stops an already-running instance. Services may already hold usable data keys.
- Plan root-volume replacement as a maintenance operation with a rollback path.
- Retain original storage only as long as required for recovery, then handle unencrypted copies according to the retention policy.
- Preserve audit records beyond Event history when longer retention is required.
- Do not publish temporary credentials or complete presigned URLs.

## Key Takeaways

- Encryption at rest and access authorization are complementary controls.
- SSE-KMS adds a KMS authorization and availability dependency to S3 object access.
- Envelope encryption separates data encryption from protection of the data key.
- Existing unencrypted EBS volumes require a replacement workflow rather than in-place encryption.
- CloudTrail helps connect key-state changes and cryptographic activity to workload failures.
- Disabling a shared key can affect multiple dependent resources.
- Reversible key disablement is different from permanent key deletion.
