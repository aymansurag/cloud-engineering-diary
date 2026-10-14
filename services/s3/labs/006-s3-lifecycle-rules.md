# S3 Lab 006 — Lifecycle Rules and Storage Management

## 1. Goal

Learn how Amazon S3 Lifecycle rules manage objects automatically over time. Practise configuring storage-class transitions, filtering objects by prefix and size, inspecting object versions, and understanding expiration in a versioned bucket.

## 2. Environment

* AWS service: Amazon S3
* AWS account: `133775960371`
* CLI profile: `aws-learning`
* Region: `us-east-1`
* Bucket: `aymen-s3-learning-2026-133775960371`
* Lifecycle practice prefix: `practice/lifecycle-demo/`
* Destination storage class: `STANDARD_IA`

## 3. Inspect the existing lifecycle configuration

I checked whether the bucket already had lifecycle rules:

```bash
aws s3api get-bucket-lifecycle-configuration \
  --bucket aymen-s3-learning-2026-133775960371 \
  --profile aws-learning
```

AWS returned `NoSuchLifecycleConfiguration`.

This indicated that the bucket did not have a lifecycle configuration at the time of inspection.

I also checked the bucket's Region:

```bash
aws s3api get-bucket-location \
  --bucket aymen-s3-learning-2026-133775960371 \
  --profile aws-learning
```

The result was `null`, which represents `us-east-1` for this API.

### What I learned

* `get-bucket-lifecycle-configuration` retrieves the lifecycle configuration.
* `get-bucket-location` checks the bucket's Region constraint.
* Inspecting the current configuration before making changes helps avoid accidentally overwriting existing rules.

## 4. Create practice objects

I created a small text file:

```bash
printf "S3 lifecycle practice object\n" > lifecycle-demo.txt
```

I inspected its contents:

```bash
cat lifecycle-demo.txt
```

I uploaded it under the dedicated lifecycle prefix:

```bash
aws s3 cp lifecycle-demo.txt \
  s3://aymen-s3-learning-2026-133775960371/practice/lifecycle-demo/lifecycle-demo.txt \
  --profile aws-learning
```

I then created a larger test file:

```bash
dd if=/dev/zero of=lifecycle-demo-large.bin bs=1024 count=129
```

This created a file containing 129 KiB, or 132,096 bytes.

I checked its local size:

```bash
ls -lh lifecycle-demo-large.bin
```

Then I uploaded it:

```bash
aws s3 cp lifecycle-demo-large.bin \
  s3://aymen-s3-learning-2026-133775960371/practice/lifecycle-demo/lifecycle-demo-large.bin \
  --profile aws-learning
```

### What I learned

* `aws s3 cp` can upload a local file to an S3 object key.
* A prefix allows lifecycle rules to target a particular group of objects.
* The object-size filter lets a rule target objects above a specified size.

## 5. Inspect object metadata

I used `head-object` to inspect the larger test object:

```bash
aws s3api head-object \
  --bucket aymen-s3-learning-2026-133775960371 \
  --key practice/lifecycle-demo/lifecycle-demo-large.bin \
  --profile aws-learning
```

### What I learned

The `head-object` command retrieves metadata without downloading the object's contents. The `ContentLength` field reports the object's size in bytes.

This helps verify whether an object meets a lifecycle rule's size filter.

## 6. Create the first lifecycle rule

I created a local lifecycle configuration file named `lifecycle-rule.json`.

The first rule was:

```json
{
  "ID": "LifecycleDemo-TransitionLargeObjectsToStandardIA",
  "Filter": {
    "And": {
      "Prefix": "practice/lifecycle-demo/",
      "ObjectSizeGreaterThan": 131071
    }
  },
  "Status": "Enabled",
  "Transitions": [
    {
      "Days": 30,
      "StorageClass": "STANDARD_IA"
    }
  ]
}
```

I reviewed the file:

```bash
cat lifecycle-rule.json
```

### What the rule means

* `ID` identifies the rule.
* `Prefix` limits the rule to objects under `practice/lifecycle-demo/`.
* `ObjectSizeGreaterThan` selects objects larger than 131,071 bytes.
* `Status: Enabled` activates the rule.
* `Days: 30` sets the transition eligibility period.
* `StorageClass: STANDARD_IA` specifies the destination storage class.

The large test object is 132,096 bytes, so it satisfies the size filter. The small text file does not.

S3 Lifecycle has a default minimum object-size behavior for transitions. The explicit size filter in this rule also excludes objects at or below 131,071 bytes.

## 7. Apply and verify the lifecycle configuration

I applied the configuration:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket aymen-s3-learning-2026-133775960371 \
  --lifecycle-configuration file://lifecycle-rule.json \
  --profile aws-learning
```

I verified the saved configuration:

```bash
aws s3api get-bucket-lifecycle-configuration \
  --bucket aymen-s3-learning-2026-133775960371 \
  --profile aws-learning
```

The response contained the rule ID `LifecycleDemo-TransitionLargeObjectsToStandardIA`, its enabled status, the 30-day transition period, and the `STANDARD_IA` destination.

### What I learned

`put-bucket-lifecycle-configuration` creates or replaces the bucket's lifecycle configuration. When updating it, the JSON must include every existing rule I want to retain.

## 8. Inspect objects and versions

I listed the objects under my lifecycle prefix:

```bash
aws s3api list-objects-v2 \
  --bucket aymen-s3-learning-2026-133775960371 \
  --prefix practice/lifecycle-demo/ \
  --query 'Contents[].{Key:Key,Size:Size,StorageClass:StorageClass,LastModified:LastModified}' \
  --output table \
  --profile aws-learning
```

I also listed object versions:

```bash
aws s3api list-object-versions \
  --bucket aymen-s3-learning-2026-133775960371 \
  --prefix practice/lifecycle-demo/ \
  --profile aws-learning
```

### What I learned

* `list-objects-v2` lists current objects and their metadata.
* `list-object-versions` lists object versions and delete markers.
* Versioning makes it important to distinguish a current version from a noncurrent version.

## 9. Practise S3 Versioning

I created a local file for the versioning exercise:

```bash
printf "Lifecycle version 1\n" > lifecycle-version-demo.txt
```

I uploaded it:

```bash
aws s3 cp lifecycle-version-demo.txt \
  s3://aymen-s3-learning-2026-133775960371/practice/lifecycle-demo/lifecycle-version-demo.txt \
  --profile aws-learning
```

I changed its contents:

```bash
printf "Lifecycle version 2\n" > lifecycle-version-demo.txt
```

Then I uploaded it to the same S3 key:

```bash
aws s3 cp lifecycle-version-demo.txt \
  s3://aymen-s3-learning-2026-133775960371/practice/lifecycle-demo/lifecycle-version-demo.txt \
  --profile aws-learning
```

I inspected the versions:

```bash
aws s3api list-object-versions \
  --bucket aymen-s3-learning-2026-133775960371 \
  --prefix practice/lifecycle-demo/lifecycle-version-demo.txt \
  --query 'Versions[].{Key:Key,VersionId:VersionId,IsLatest:IsLatest,Size:Size}' \
  --output table \
  --profile aws-learning
```

With S3 Versioning enabled, uploading new content to the same key creates a new version while retaining the previous version.

## 10. Create a larger versioned object

To practise lifecycle behavior for older versions, I created another large file:

```bash
dd if=/dev/zero of=lifecycle-version-large.bin bs=1024 count=129
```

I uploaded the first version under a dedicated prefix:

```bash
aws s3 cp lifecycle-version-large.bin \
  s3://aymen-s3-learning-2026-133775960371/practice/lifecycle-demo/versioned/lifecycle-version-large.bin \
  --profile aws-learning
```

I appended a marker to the local file:

```bash
printf "Version 2 marker\n" >> lifecycle-version-large.bin
```

Then I uploaded the second version to the same key:

```bash
aws s3 cp lifecycle-version-large.bin \
  s3://aymen-s3-learning-2026-133775960371/practice/lifecycle-demo/versioned/lifecycle-version-large.bin \
  --profile aws-learning
```

I inspected its versions:

```bash
aws s3api list-object-versions \
  --bucket aymen-s3-learning-2026-133775960371 \
  --prefix practice/lifecycle-demo/versioned/lifecycle-version-large.bin \
  --query 'Versions[].{Key:Key,VersionId:VersionId,IsLatest:IsLatest,Size:Size}' \
  --output table \
  --profile aws-learning
```

This exercise demonstrates how a newly uploaded version becomes current and the previous version becomes noncurrent.

## 11. Add a rule for noncurrent versions

I updated `lifecycle-rule.json` to include both lifecycle rules:

```json
{
  "Rules": [
    {
      "ID": "LifecycleDemo-TransitionLargeObjectsToStandardIA",
      "Filter": {
        "And": {
          "Prefix": "practice/lifecycle-demo/",
          "ObjectSizeGreaterThan": 131071
        }
      },
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        }
      ]
    },
    {
      "ID": "LifecycleDemo-TransitionNoncurrentVersionsToStandardIA",
      "Filter": {
        "And": {
          "Prefix": "practice/lifecycle-demo/versioned/",
          "ObjectSizeGreaterThan": 131071
        }
      },
      "Status": "Enabled",
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 30,
          "StorageClass": "STANDARD_IA"
        }
      ]
    }
  ]
}
```

I reviewed the updated JSON:

```bash
cat lifecycle-rule.json
```

I then applied the complete configuration:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket aymen-s3-learning-2026-133775960371 \
  --lifecycle-configuration file://lifecycle-rule.json \
  --profile aws-learning
```

Finally, I verified it:

```bash
aws s3api get-bucket-lifecycle-configuration \
  --bucket aymen-s3-learning-2026-133775960371 \
  --profile aws-learning
```

The final configuration contains two enabled transition rules:

1. `LifecycleDemo-TransitionLargeObjectsToStandardIA` transitions eligible current versions after 30 days.
2. `LifecycleDemo-TransitionNoncurrentVersionsToStandardIA` transitions eligible noncurrent versions after 30 days of being noncurrent.

### What I learned

* `Transitions` applies to eligible current object versions.
* `NoncurrentVersionTransitions` applies to eligible noncurrent versions.
* `Days` and `NoncurrentDays` refer to different starting points.
* Replacing a lifecycle configuration requires preserving all rules that should remain active.

## 12. Understand expiration and Versioning

I learned the difference between these two lifecycle actions:

### `Expiration`

In a versioning-enabled bucket, expiration of the current version normally adds a delete marker. Previous object data can remain stored as noncurrent versions.

### `NoncurrentVersionExpiration`

This action permanently deletes eligible noncurrent versions.

I did not add either expiration action to my bucket's lifecycle configuration during this lab.

## 13. Storage class costs and timing

S3 lifecycle transitions happen asynchronously. An object does not necessarily move to its target storage class immediately when its eligibility period ends.

S3 Standard-IA is designed for infrequently accessed data. It has retrieval charges and a 30-day minimum storage duration.

The lifecycle rules created in this lab are enabled, but the 30-day transitions do not happen immediately. I will avoid applying broad expiration rules without first reviewing the configuration and the consequences for versioned objects.

## 14. Outcome

In this lab, I inspected the bucket's lifecycle configuration, prepared test objects, created and applied a lifecycle rule for current versions, and verified the stored configuration.

I then practised S3 Versioning by uploading new versions to the same object key. I updated the lifecycle configuration to include a separate rule for eligible noncurrent versions and learned why lifecycle transitions, current-version expiration, and noncurrent-version expiration must be treated differently.

**Final safety note:** The active lifecycle configuration contains transition rules only. I did not configure permanent deletion. The rules are restricted to the `practice/lifecycle-demo/` prefix and require objects to exceed the specified size threshold.

**Next step:** Document and review the commands for this lab, then continue practising S3 storage management before moving to a larger project.
