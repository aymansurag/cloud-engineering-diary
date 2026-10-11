# S3 Lab 003 — AccessDenied Troubleshooting

## 1. Goal

Understand why an AWS CLI request can return `AccessDenied` even when authentication succeeds, and learn how to diagnose and safely resolve an S3 permission problem.

## 2. Environment

* AWS service: Amazon S3
* Region: `us-east-1`
* AWS account: `133775960371`
* CLI profile: `aws-learning`
* Authentication: IAM Identity Center
* Bucket: `aymen-s3-learning-2026-133775960371`
* Practice object: `practice/version-test.txt`

## 3. Verify identity and baseline access

I logged in through IAM Identity Center:

```bash
aws sso login --profile aws-learning
```

I verified the AWS identity:

```bash
aws sts get-caller-identity --profile aws-learning
```

I confirmed that I could list the practice prefix:

```bash
aws s3 ls s3://aymen-s3-learning-2026-133775960371/practice/ --profile aws-learning
```

The object was visible before I introduced the test restriction.

## 4. Check for an existing bucket policy

I checked whether the bucket already had a bucket policy:

```bash
aws s3api get-bucket-policy --bucket aymen-s3-learning-2026-133775960371 --profile aws-learning
```

AWS returned `NoSuchBucketPolicy`, confirming that no bucket policy was attached at that point.

This check mattered because I did not want to overwrite an existing policy while creating the controlled test.

## 5. Create a controlled explicit Deny

I created a temporary policy that denied `s3:GetObject` for only this object:

`arn:aws:s3:::aymen-s3-learning-2026-133775960371/practice/version-test.txt`

The policy used:

* `Effect: Deny`
* `Action: s3:GetObject`
* A resource ARN restricted to the practice object

I applied the policy using:

```bash
aws s3api put-bucket-policy \
  --bucket aymen-s3-learning-2026-133775960371 \
  --policy file://lesson3-deny-policy.json \
  --profile aws-learning
```

I initially received `MalformedPolicy` because the JSON file had a blank line before its opening `{`. I inspected the file with `cat -A`, removed the leading blank line, and successfully applied the policy.

**Lesson learned:** When AWS reports a JSON formatting error, inspect the file before assuming the problem is related to permissions.

## 6. Reproduce the failure

I attempted to download the practice object:

```bash
aws s3 cp s3://aymen-s3-learning-2026-133775960371/practice/version-test.txt denied-test.txt --profile aws-learning
```

AWS returned:

```text
403 Forbidden
```

The error occurred during the `HeadObject` operation used by the high-level copy command to inspect the source object.

I also confirmed that `head-object` was forbidden while the policy was applied.

## 7. Compare listing and reading permissions

I ran:

```bash
aws s3 ls s3://aymen-s3-learning-2026-133775960371/practice/ --profile aws-learning
```

The listing still worked, even though reading the object's contents was denied.

This demonstrated an important distinction:

* `s3:ListBucket` governs listing objects in a bucket.
* `s3:GetObject` governs reading an object's contents and is required for `HeadObject` requests.

The policy denied `s3:GetObject` for one object; it did not deny the listing operation.

## 8. Identify the root cause

The root cause was the explicit `Deny` in the bucket policy.

AWS evaluates applicable policies, and an explicit Deny overrides an Allow. Therefore, the broad permissions associated with my AdministratorAccess permission set did not override this applicable bucket-policy Deny.

This was an authorization problem, not an authentication failure.

## 9. Restore normal access

Before the exercise, I had confirmed that the bucket had no existing bucket policy. I therefore removed the temporary policy:

```bash
aws s3api delete-bucket-policy \
  --bucket aymen-s3-learning-2026-133775960371 \
  --profile aws-learning
```

I retested the object's metadata:

```bash
aws s3api head-object \
  --bucket aymen-s3-learning-2026-133775960371 \
  --key practice/version-test.txt \
  --profile aws-learning
```

Then I retried the download:

```bash
aws s3 cp s3://aymen-s3-learning-2026-133775960371/practice/version-test.txt denied-test.txt --profile aws-learning
```

Both operations succeeded after the temporary policy was removed.

## 10. Troubleshooting process for work

If an S3 request returns `AccessDenied`, I should:

1. Verify my AWS identity and account using `aws sts get-caller-identity`.
2. Confirm that the bucket name and object key are correct.
3. Identify the action that failed, such as listing, reading, uploading, or deleting.
4. Inspect the relevant IAM and bucket policies for an explicit Deny or missing Allow.
5. Check other applicable restrictions, such as organization or VPC endpoint policies.
6. Make only a justified permission change, then repeat the failed operation to verify the fix.

I should not grant broader permissions until I understand the cause.

## 11. Key takeaways

* Authentication answers, “Who am I?”
* Authorization answers, “Am I allowed to perform this action?”
* Listing and reading an object require different S3 permissions.
* An explicit Deny overrides an applicable Allow.
* A `403 Forbidden` response is evidence of a failed request, but the exact cause must be investigated.
* Checking for an existing bucket policy before changing it helps avoid overwriting existing access rules.
* After a temporary policy change, verify that the test restriction has been removed and the original operation works again.

## 12. Outcome

I successfully created a controlled S3 permission failure, observed the difference between listing and reading an object, identified the explicit Deny as the cause, removed the temporary policy, and verified that metadata inspection and download worked again.

**Next lesson:** Practice S3 permissions and investigate other causes of `AccessDenied` without relying on an intentionally created bucket policy.
