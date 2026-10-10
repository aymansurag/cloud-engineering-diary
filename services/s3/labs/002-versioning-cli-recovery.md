# S3 Lab 002 — Object Versioning and CLI Recovery

## 1. Goal

Learn how S3 Versioning preserves previous object versions, how to retrieve a specific version, and how to recover an object after a normal delete using AWS CLI.

## 2. Environment

- AWS service: Amazon S3
- AWS Region: `us-east-1`
- Authentication: IAM Identity Center
- CLI profile: `aws-learning`
- Versioning: Enabled
- Test object key: `practice/version-test.txt`

## 3. Verify AWS identity and bucket versioning

I authenticated through IAM Identity Center:

```bash
aws sso login --profile aws-learning
```

I verified which AWS account and identity the CLI was using:

```bash
aws sts get-caller-identity --profile aws-learning
```

I checked the bucket's versioning configuration:

```bash
aws s3api get-bucket-versioning \
  --bucket aymen-s3-learning-2026-133775960371 \
  --profile aws-learning
```

The bucket returned `Status: Enabled`.

**Why this matters:** Before making changes, I should verify the AWS identity, account, and relevant bucket configuration.

## 4. Create and upload Version 1

I created a local test file:

```bash
printf "Version 1: original backup\n" > version-test.txt
```

I uploaded it to S3:

```bash
aws s3 cp version-test.txt \
  s3://aymen-s3-learning-2026-133775960371/practice/version-test.txt \
  --profile aws-learning
```

I inspected the object metadata:

```bash
aws s3api head-object \
  --bucket aymen-s3-learning-2026-133775960371 \
  --key practice/version-test.txt \
  --profile aws-learning
```

The response included a `VersionId`.

**Why this matters:** With bucket versioning enabled, S3 assigns a version identifier to each stored version of an object.

## 5. Upload Version 2

I changed the local file's contents:

```bash
printf "Version 2: verified backup\n" > version-test.txt
```

I uploaded it to the same S3 key:

```bash
aws s3 cp version-test.txt \
  s3://aymen-s3-learning-2026-133775960371/practice/version-test.txt \
  --profile aws-learning
```

I ran `head-object` again and observed a different `VersionId`.

I then listed the version history:

```bash
aws s3api list-object-versions \
  --bucket aymen-s3-learning-2026-133775960371 \
  --prefix practice/version-test.txt \
  --profile aws-learning
```

The response contained two versions of the same key. Version 2 was the latest, while Version 1 remained available.

**What I learned:** Uploading new content to the same key in a versioning-enabled bucket creates a new version instead of destroying the previous version.

## 6. Retrieve an older version

I used `get-object` with Version 1's version ID to download the older contents without changing the current S3 version:

```bash
aws s3api get-object \
  --bucket aymen-s3-learning-2026-133775960371 \
  --key practice/version-test.txt \
  --version-id hYTqXDPE2Us9wQe9MwV1aXq4ocTIavOw \
  --profile aws-learning \
  version1-retrieved.txt
```

I checked the downloaded file:

```bash
cat version1-retrieved.txt
```

The file contained:

```text
Version 1: original backup
```

**What I learned:** Retrieving an older version is different from making it the current version. This command downloaded an older copy to my computer without modifying S3.

## 7. Investigate an accidental deletion

I performed a normal delete on the practice object:

```bash
aws s3 rm \
  s3://aymen-s3-learning-2026-133775960371/practice/version-test.txt \
  --profile aws-learning
```

I listed the object versions:

```bash
aws s3api list-object-versions \
  --bucket aymen-s3-learning-2026-133775960371 \
  --prefix practice/version-test.txt \
  --profile aws-learning
```

The response showed:

- A delete marker with `IsLatest: true`
- Version 2 still stored
- Version 1 still stored

The object no longer appeared in the ordinary object listing because the delete marker became the current entry.

## 8. Recover the object

I identified the exact delete marker's version ID and removed only that marker:

```bash
aws s3api delete-object \
  --bucket aymen-s3-learning-2026-133775960371 \
  --key practice/version-test.txt \
  --version-id OhtzyQvULTiVScW0I0TSwwd_JQFQgbsh \
  --profile aws-learning
```

I then verified that the object was visible again:

```bash
aws s3 ls \
  s3://aymen-s3-learning-2026-133775960371/practice/ \
  --profile aws-learning
```

I confirmed the current version using `head-object`.

**Important:** I removed the delete marker, not either data version. The previous latest data version became current again.

## 9. Troubleshooting lessons

### Object not visible

I should check the exact bucket name and object key, then inspect version history for a delete marker.

### Need an older version

I can use `list-object-versions` to find the version ID and `get-object --version-id` to retrieve that specific version.

### AccessDenied

I should first verify my identity with `aws sts get-caller-identity`. Then I should investigate effective IAM permissions, bucket policies, and other applicable access restrictions.

### Important limitation

Versioning helps recover from normal deletes and overwrites, but it does not guarantee protection against permanent deletion of specific versions.

## 10. Summary

This lab taught me to:

- Verify my AWS identity and bucket versioning configuration.
- Upload two versions of the same S3 object.
- Inspect version IDs and identify the latest version.
- Download an older version without changing the current object.
- Investigate a delete marker.
- Recover an object without re-uploading the local file.

**Next step:** Practise S3 permissions and diagnose a controlled `AccessDenied` error.

> Security reminder: Never commit AWS credentials, access keys, MFA secrets, or other authentication secrets to GitHub.