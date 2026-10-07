# Week 2 — IAM & Security Foundations

## Policies

### `policies/S3UploaderOnly-minh.json`

Attached directly to `s3-test-user`, an IAM user with no console access that only has programmatic access through the `s3test` CLI profile.

**What it allows:** `s3:PutObject` and `s3:GetObject` on objects inside `my-training-bucket-minh` (`arn:aws:s3:::my-training-bucket-minh/*`). That is all it allows.

**What it does not allow:** listing buckets, listing the objects in the bucket, deleting or overwriting bucket settings, or touching any other bucket. Anything not explicitly allowed is an implicit deny.

**Why I scoped it this way:** the user only needs to upload and download files in one training bucket, so it gets only those two actions on only that bucket's objects. The `/*` at the end of the ARN matters: object-level actions like `PutObject` and `GetObject` apply to object ARNs, not to the bucket ARN. Without the `/*`, the policy would match the bucket itself and every upload would be denied. Bucket-level actions such as `s3:ListBucket` would need the bucket ARN without `/*` as a separate resource, and I left that out on purpose because this user doesn't need to list anything.

I checked the policy with the IAM policy simulator (`aws iam simulate-custom-policy`):

| Action | Resource | Decision |
|---|---|---|
| `s3:PutObject` | `my-training-bucket-minh/file.txt` | allowed |
| `s3:GetObject` | `my-training-bucket-minh/file.txt` | allowed |
| `s3:PutObject` | `my-training-bucket-minh` (no `/*`) | implicitDeny |
| `s3:DeleteObject` | `my-training-bucket-minh/file.txt` | implicitDeny |
| `s3:ListAllMyBuckets` | `*` | implicitDeny |
| `s3:PutObject` | `some-other-bucket/file.txt` | implicitDeny |

Running `aws s3 ls --profile s3test` returns `AccessDenied` because `ls` calls `s3:ListAllMyBuckets`, which the policy never grants. That is least privilege working correctly, not a bug. I tested with the separate `s3test` profile instead of my admin profile, because the admin profile would never show a deny.

## Access Analyzer

I created an account-level IAM Access Analyzer (`ConsoleAnalyzer-minh`) with the default settings. It shows 0 active findings, which means nothing in the account is shared with an outside principal. It keeps auditing the account for the rest of the program.

## Policy evaluation notes

- By default every request is denied (implicit deny).
- An explicit `Allow` in an identity-based or resource-based policy overrides the implicit deny.
- An explicit `Deny` anywhere always wins over any `Allow`.
