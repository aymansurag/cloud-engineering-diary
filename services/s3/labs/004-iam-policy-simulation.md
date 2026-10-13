# S3 Lab 004 — IAM Policies and Permission Simulation

## 1. Goal

Understand how IAM identity policies and permissions boundaries affect access to S3 resources. Learn how to inspect an IAM role, read a managed policy document, and simulate authorization decisions without changing real AWS permissions.

## 2. Environment

- AWS service: IAM and Amazon S3
- AWS account: `133775960371`
- CLI profile: `aws-learning`
- Region: `us-east-1`
- Identity Center user: `cloudadmin`
- Permission set: `AdministratorAccess`
- Practice bucket: `aymen-s3-learning-2026-133775960371`
- Practice object: `practice/version-test.txt`

## 3. Verify the AWS identity

I checked which AWS account and identity my CLI session was using:

```bash
aws sts get-caller-identity --profile aws-learning
```

The result confirmed account `133775960371` and an assumed-role session associated with `cloudadmin`.

**What I learned:** Authentication confirms my identity, while authorization determines which operations I am allowed to perform.

## 4. Inspect managed policies attached to the role

I listed the managed policies attached directly to my IAM Identity Center-created role:

```bash
aws iam list-attached-role-policies \
  --role-name AWSReservedSSO_AdministratorAccess_72912ccdd73682ea \
  --profile aws-learning
```

The output showed the AWS-managed policy:

`arn:aws:iam::aws:policy/AdministratorAccess`

**What I learned:** A role can have managed policies attached to it. This command lists those attached managed policies; it does not show every possible source of permissions or restrictions.

## 5. Inspect the AdministratorAccess policy

I retrieved the managed policy's metadata:

```bash
aws iam get-policy \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
  --profile aws-learning
```

The output identified the default policy version as `v1`.

I then retrieved its permission document:

```bash
aws iam get-policy-version \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
  --version-id v1 \
  --profile aws-learning
```

The document contained:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

Interpretation:

- `Effect: Allow` grants permission.
- `Action: "*"` covers all actions included by this statement.
- `Resource: "*"` applies across resources rather than limiting the statement to one resource.

**What I learned:** AdministratorAccess grants broad permissions, but an applicable explicit Deny or other restriction can still prevent a particular request.

## 6. Inspect the IAM role

I retrieved the role configuration:

```bash
aws iam get-role \
  --role-name AWSReservedSSO_AdministratorAccess_72912ccdd73682ea \
  --profile aws-learning
```

The response did not contain a `PermissionsBoundary` field, so no permissions boundary was attached to this role at the time of inspection.

I also checked for inline policies:

```bash
aws iam list-role-policies \
  --role-name AWSReservedSSO_AdministratorAccess_72912ccdd73682ea \
  --profile aws-learning
```

The result was:

```json
{
  "PolicyNames": []
}
```

**What I learned:** Managed policies and inline policies are different. Listing attached managed policies does not reveal inline policies, so I inspected both.

## 7. Check the S3 object's access baseline

I confirmed that the practice object was accessible through a metadata request:

```bash
aws s3api head-object \
  --bucket aymen-s3-learning-2026-133775960371 \
  --key practice/version-test.txt \
  --profile aws-learning
```

The command succeeded and returned metadata including the object's `VersionId`, content length, content type, and server-side encryption.

I checked the bucket policy:

```bash
aws s3api get-bucket-policy \
  --bucket aymen-s3-learning-2026-133775960371 \
  --profile aws-learning
```

AWS returned `NoSuchBucketPolicy`, indicating that no bucket policy was attached at that time.

**What I learned:** Checking the actual resource and its configuration helps establish a baseline before investigating a permission problem. The absence of a bucket policy does not mean that no other authorization restrictions can apply.

## 8. Simulate an identity-policy decision

I used the IAM policy simulator to test whether the role's identity-based permissions allowed reading the practice object:

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::133775960371:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_AdministratorAccess_72912ccdd73682ea \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::aymen-s3-learning-2026-133775960371/practice/version-test.txt \
  --profile aws-learning
```

The simulation reported the read action as allowed.

**What I learned:** The simulator can evaluate authorization decisions without performing the requested S3 operation. Its result does not guarantee that every real request will succeed, because not every resource-based policy or live restriction is necessarily represented in a simulation.

## 9. Simulate a hypothetical permissions boundary

I then supplied a hypothetical permissions boundary directly to the simulator. It allowed `s3:GetObject` on the practice object but did not allow `s3:DeleteObject`.

The simulation evaluated these two actions:

- `s3:GetObject` — allowed
- `s3:DeleteObject` — denied by the simulated boundary

This boundary was supplied only for the simulation. **I did not attach it to my real IAM role or change my actual AWS permissions.**

**What I learned:** A permissions boundary sets the maximum permissions that identity-based policies can grant. It does not grant permissions on its own.

## 10. Simulate a resource ARN mismatch

I simulated access to:

`arn:aws:s3:::aymen-s3-learning-2026-133775960371/practice/version-test.txt`

The hypothetical boundary allowed `s3:GetObject` only on:

`arn:aws:s3:::aymen-s3-learning-2026-133775960371/practice/another-object.txt`

The simulator returned `implicitDeny`.

**What I learned:** The requested resource must match the resource covered by an applicable Allow. An Allow for a different object does not automatically grant access to the requested object.

## 11. Explicit Deny versus implicit Deny

### Explicit Deny

A matching policy statement explicitly contains:

```json
{
  "Effect": "Deny",
  "Action": "s3:DeleteObject",
  "Resource": "*"
}
```

An applicable explicit Deny overrides an Allow.

### Implicit Deny

No applicable policy grants the required permission. For example, a simulated permissions boundary might allow reading but not deletion.

**Memory rule:**

- Explicit Deny: the policy specifically forbids the action.
- Implicit Deny: the action is not allowed by the applicable permission rules.

## 12. Troubleshooting process

When investigating an S3 `AccessDenied` error, I should:

1. Confirm the AWS identity and account.
2. Identify the exact failed action, such as `s3:GetObject`.
3. Verify the exact bucket name and object key.
4. Inspect relevant identity-based and resource-based policies.
5. Check for permissions boundaries and other applicable restrictions.
6. Use the IAM policy simulator when appropriate.
7. Change only the rule responsible for the failure, then retest the original request.

I should not assume that every `AccessDenied` is caused by a missing IAM Allow, and I should not grant broader permissions without understanding the cause.

## 13. Outcome

In this lab, I inspected the policies and configuration of my IAM Identity Center-created role, examined the AdministratorAccess document, checked the role for inline policies and a permissions boundary, and used the IAM policy simulator to explore permitted and denied S3 actions.

I also learned that a policy's action and resource must match the request, and that a successful simulation is not a guarantee that every live request will succeed.

**Next step:** Continue practising IAM permission evaluation and S3 access troubleshooting, then consolidate the completed lessons into a small S3 project.