# S3 Lab 005 — Encryption and Data Protection

## 1. Goal

Understand Amazon S3 server-side encryption, compare SSE-S3 with SSE-KMS, inspect bucket and object encryption settings, and verify that encrypted objects can be retrieved.

## 2. Environment

- AWS account: `133775960371`
- CLI profile: `aws-learning`
- Region: `us-east-1`
- Bucket: `aymen-s3-learning-2026-133775960371`
- Practice prefix: `practice/`
- Local test file: `encryption-demo-lesson5.txt`

## 3. Inspect the bucket's default encryption

I ran:

```bash
aws s3api get-bucket-encryption \
  --bucket aymen-s3-learning-2026-133775960371 \
  --profile aws-learning
```

The output showed:

```json
{
  "ApplyServerSideEncryptionByDefault": {
    "SSEAlgorithm": "AES256"
  }
}
```

The response also reported `BucketKeyEnabled: true` and that SSE-C was blocked for new uploads.

### What I learned

- `get-bucket-encryption` retrieves the bucket's default encryption configuration.
- `AES256` indicates SSE-S3.
- Bucket default encryption applies to new uploads; the metadata of an existing object should be checked separately.
- Blocking SSE-C means the bucket rejects uploads that request customer-provided encryption keys.

## 4. Inspect an existing object's encryption

I ran:

```bash
aws s3api head-object \
  --bucket aymen-s3-learning-2026-133775960371 \
  --key practice/version-test.txt \
  --profile aws-learning
```

The response included:

```json
{
  "ServerSideEncryption": "AES256",
  "ContentType": "text/plain",
  "ContentLength": 27
}
```

### What I learned

- `head-object` retrieves object metadata without returning the object's contents.
- `ServerSideEncryption: AES256` confirms that S3 reports SSE-S3 for this object version.
- Bucket configuration describes the default for new uploads, while object metadata reports the encryption associated with the specific object version.

## 5. Create and upload a test object with SSE-S3

I created a local test file:

```bash
printf "Lesson 5: default encryption test\n" > encryption-demo-lesson5.txt
```

I checked its contents:

```bash
cat encryption-demo-lesson5.txt
```

I uploaded it using the bucket's default encryption:

```bash
aws s3 cp encryption-demo-lesson5.txt \
  s3://aymen-s3-learning-2026-133775960371/practice/encryption-demo-lesson5.txt \
  --profile aws-learning
```

I also uploaded the same content while explicitly requesting SSE-S3:

```bash
aws s3 cp encryption-demo-lesson5.txt \
  s3://aymen-s3-learning-2026-133775960371/practice/encryption-explicit-lesson5.txt \
  --sse AES256 \
  --profile aws-learning
```

I inspected the second object's metadata with `head-object` and observed `ServerSideEncryption: AES256`.

### What I learned

SSE-S3 uses encryption keys managed by Amazon S3. S3 automatically encrypts new objects at rest, and `--sse AES256` explicitly requests SSE-S3 during an upload.

## 6. Inspect AWS KMS aliases

I listed KMS aliases in `us-east-1`:

```bash
aws kms list-aliases \
  --profile aws-learning \
  --region us-east-1
```

The results included:

`alias/aws/s3`

I inspected that alias:

```bash
aws kms describe-key \
  --key-id alias/aws/s3 \
  --region us-east-1 \
  --profile aws-learning
```

The output confirmed that the key was enabled and AWS-managed. It returned a key ARN and the `KeyManager: AWS` value.

### What I learned

- AWS KMS manages keys that can be used for SSE-KMS.
- A key alias is a friendly name for identifying a KMS key.
- `describe-key` retrieves key metadata; it does not return the key material.
- The AWS-managed S3 KMS key is different from SSE-S3 encryption.

## 7. Upload an object using SSE-KMS

I uploaded a new object using SSE-KMS:

```bash
aws s3 cp encryption-demo-lesson5.txt \
  s3://aymen-s3-learning-2026-133775960371/practice/encryption-kms-lesson5.txt \
  --sse aws:kms \
  --profile aws-learning \
  --region us-east-1
```

I then downloaded the object and inspected the local content.

### What I learned

- `--sse aws:kms` requests SSE-KMS.
- SSE-KMS uses AWS KMS keys for server-side encryption.
- Normal downloads are decrypted automatically by S3 when the requester has the required permissions.
- Depending on the applicable policies and key configuration, SSE-KMS access can require both `s3:GetObject` and KMS decryption permission.

## 8. Specify a KMS key explicitly

I uploaded another object while specifying the S3 KMS key alias:

```bash
aws s3 cp encryption-demo-lesson5.txt \
  s3://aymen-s3-learning-2026-133775960371/practice/encryption-kms-explicit-key-lesson5.txt \
  --sse aws:kms \
  --sse-kms-key-id alias/aws/s3 \
  --profile aws-learning \
  --region us-east-1
```

I checked the resulting object metadata:

```bash
aws s3api head-object \
  --bucket aymen-s3-learning-2026-133775960371 \
  --key practice/encryption-kms-explicit-key-lesson5.txt \
  --profile aws-learning
```

The output included:

```json
{
  "ServerSideEncryption": "aws:kms",
  "SSEKMSKeyId": "arn:aws:kms:us-east-1:133775960371:key/d6a26c12-b631-4948-b506-1456e0945a6b",
  "BucketKeyEnabled": true
}
```

### What I learned

- `--sse-kms-key-id` specifies the KMS key for the upload.
- `ServerSideEncryption: aws:kms` confirms that the object uses SSE-KMS.
- `SSEKMSKeyId` identifies the key associated with the object.
- `BucketKeyEnabled: true` indicates that S3 Bucket Keys are enabled for this object. Bucket Keys can reduce requests to KMS and help reduce KMS request costs.

## 9. Compare SSE-S3 and SSE-KMS

| Feature | SSE-S3 | SSE-KMS |
|---|---|---|
| Encryption at rest | Yes | Yes |
| Key management | Managed by S3 | Managed through KMS |
| Explicit upload option | `--sse AES256` | `--sse aws:kms` |
| Specify KMS key | Not applicable | `--sse-kms-key-id` |
| Additional KMS permissions | No | May be required for the requested operation |
| Key-related auditing and controls | S3-managed | Additional KMS controls and auditing |

## 10. Troubleshooting and safety

When investigating S3 encryption, I should:

1. Check the bucket's default encryption with `get-bucket-encryption`.
2. Check the actual object's encryption with `head-object`.
3. Identify the encryption algorithm shown in `ServerSideEncryption`.
4. For SSE-KMS, inspect the associated key and relevant permissions when needed.
5. Verify that the intended object was uploaded and that retrieval succeeds.
6. Avoid changing the bucket's default encryption or KMS permissions unnecessarily.

## 11. Outcome

I inspected my S3 bucket's default encryption, confirmed SSE-S3 on an existing object, uploaded test objects using both SSE-S3 and SSE-KMS, inspected the relevant KMS key, and verified the SSE-KMS metadata of an object uploaded with an explicitly specified key.

This exercise taught me how to distinguish bucket defaults from object-level encryption metadata and how to verify encryption settings through the AWS CLI.

**Next step:** Continue learning S3 data protection and storage management, then combine the completed labs into a small documented S3 project.