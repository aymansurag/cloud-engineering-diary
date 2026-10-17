Lesson 7 — Amazon S3 Replication
Topic: S3 Replication (SRR and CRR)
AWS CLI profile used in examples: aws-learning
Purpose: Study notes and safe command templates for checking, configuring, and troubleshooting replication.
Safety: Replace every placeholder such as SOURCE_BUCKET, DESTINATION_BUCKET, SOURCE_REGION, account IDs, role ARNs, and KMS key ARNs before running a command. put-bucket-replication changes the source bucket's replication configuration. Inspect and preserve any existing rules before updating it.

1. What replication does
Amazon S3 Replication automatically copies eligible objects from a source bucket to a destination bucket.
- Same-Region Replication (SRR): source and destination are in the same AWS Region.
- Cross-Region Replication (CRR): source and destination are in different AWS Regions.
- Replication is configured from the source bucket to the destination; it is not automatically bidirectional.
- Prefix filters apply to the beginning of an object key. For example, reports/ matches reports/monthly.txt, but not images/photo.jpg.
2. Requirements to remember
1. Versioning must be Enabled on both source and destination buckets.
2. The replication configuration on the source bucket must point to a valid destination bucket.
3. Amazon S3 must be able to assume the configured replication IAM role.
4. The role needs the required source-read and destination-replication permissions. Cross-account setups may also need a destination bucket policy.
5. SSE-KMS encrypted objects require additional replication configuration and KMS permissions; see Section 7.
3. Check Versioning on both buckets
Check the source bucket:
aws s3api get-bucket-versioning \
  --bucket SOURCE_BUCKET \
  --profile aws-learning \
  --region SOURCE_REGION
Check the destination bucket:
aws s3api get-bucket-versioning \
  --bucket DESTINATION_BUCKET \
  --profile aws-learning \
  --region DESTINATION_REGION
Look for "Status": "Enabled" in both responses. If Versioning is absent or suspended, replication requirements are not met. Do not change bucket settings unless you are authorized and have verified the target bucket.
4. Inspect the existing replication configuration
aws s3api get-bucket-replication \
  --bucket SOURCE_BUCKET \
  --profile aws-learning \
  --region SOURCE_REGION
Review the replication role ARN, every rule's ID and Status, the filter, and destination bucket ARN. If AWS reports that no replication configuration exists, live replication has not been configured on that source bucket.
Important: Treat a replication configuration as a complete configuration. Before applying an update, include every existing rule that must remain in the JSON file.
5. Example replication configuration
This is a template, not a ready-to-run configuration. It creates one enabled rule for object keys beginning with reports/. It leaves delete-marker replication disabled.
Save as replication.json:
{
  "Role": "arn:aws:iam::SOURCE_ACCOUNT_ID:role/S3ReplicationRole",
  "Rules": [
    {
      "ID": "ReplicateReports",
      "Priority": 1,
      "Status": "Enabled",
      "Filter": {
        "Prefix": "reports/"
      },
      "DeleteMarkerReplication": {
        "Status": "Disabled"
      },
      "Destination": {
        "Bucket": "arn:aws:s3:::DESTINATION_BUCKET"
      }
    }
  ]
}
Apply the configuration only after replacing placeholders and confirming the role, destination, permissions, and existing rules:
aws s3api put-bucket-replication \
  --bucket SOURCE_BUCKET \
  --replication-configuration file://replication.json \
  --profile aws-learning \
  --region SOURCE_REGION
Then verify what S3 saved:
aws s3api get-bucket-replication \
  --bucket SOURCE_BUCKET \
  --profile aws-learning \
  --region SOURCE_REGION
6. Check an object's replication status
Use head-object on the source bucket:
aws s3api head-object \
  --bucket SOURCE_BUCKET \
  --key 'reports/monthly.txt' \
  --profile aws-learning \
  --region SOURCE_REGION \
  --query '{VersionId:VersionId,ReplicationStatus:ReplicationStatus,ServerSideEncryption:ServerSideEncryption,SSEKMSKeyId:SSEKMSKeyId}' \
  --output json
Common ReplicationStatus values:
- PENDING — replication is in progress.
- COMPLETED — replication succeeded.
- FAILED — replication failed; investigate permissions, filters, destination access, encryption, and the replication configuration.
- REPLICA — the object is a replica; this value is commonly returned when inspecting the object in the destination bucket.
The field may be absent when replication status is not applicable or not available for that object. An enabled rule does not guarantee every object has replicated successfully.
7. SSE-KMS replication
By default, S3 does not replicate SSE-KMS or DSSE-KMS encrypted objects unless the replication configuration explicitly enables this behavior.
For SSE-KMS replication, check all of the following:
1. The replication rule includes SSE-KMS encrypted objects, using SourceSelectionCriteria with SseKmsEncryptedObjects set to Enabled.
2. The destination specifies the KMS key to use for the replica with Destination.EncryptionConfiguration.ReplicaKmsKeyID.
3. The replication role is authorized to use the source key for decryption (kms:Decrypt) and the destination key for encryption (kms:Encrypt).
4. The source and destination KMS key policies (and any applicable restrictions) permit the required operations.
Configuration fragment to incorporate into the relevant rule in replication.json when needed:
"SourceSelectionCriteria": {
  "SseKmsEncryptedObjects": {
    "Status": "Enabled"
  }
},
"Destination": {
  "Bucket": "arn:aws:s3:::DESTINATION_BUCKET",
  "EncryptionConfiguration": {
    "ReplicaKmsKeyID": "arn:aws:kms:DESTINATION_REGION:DESTINATION_ACCOUNT_ID:key/DESTINATION_KEY_ID"
  }
}
This fragment illustrates the required fields; integrate it into the rule carefully rather than pasting it as a second Destination block. Use a KMS key ARN appropriate for the destination Region.
8. Delete behavior and versioning
- A normal delete in a versioned source bucket normally adds a delete marker; it does not permanently erase all data versions.
- For modern replication rules that have a Filter, delete-marker replication is off by default. It can be enabled for eligible non-tag-based rules by setting DeleteMarkerReplication.Status to Enabled.
- Deleting a specific source version by Version ID permanently deletes that version in the source, but that deletion is not replicated as a deletion to the destination. The destination version normally remains.
- Delete-marker replication causes the marker to be copied; it does not permanently delete older data versions.
9. New versus existing objects
- Live replication applies to eligible objects created after the replication configuration is in place.
- Objects that already existed before the rule was configured are not automatically copied just because live replication is enabled. Use S3 Batch Replication for eligible existing objects.
- If a rule's prefix is reports/, only matching keys are selected by that filter.
10. Cross-account ownership, resilience, and cost
Cross-account object ownership
When replication crosses AWS accounts, review object ownership on the destination. If the destination bucket uses Bucket owner enforced Object Ownership, the destination bucket owner owns the replica and ACLs are disabled; access is managed with IAM and bucket policies. Also make sure the cross-account destination bucket policy grants the required replication access.
Replication is not a complete backup by itself
Replication can support disaster recovery and regional resilience, especially with CRR, but it is not automatically an independent backup. Incorrect source content may also be replicated. Deletion behavior depends on the rule's delete-marker settings, and deleting a specific source version is not replicated as a deletion. Design retention and recovery controls for the use case.
Costs and timing
CRR can incur destination storage, replication request, and inter-Region data-transfer costs. Compare total costs before selecting Regions. Replication is asynchronous, so an enabled rule does not mean every replica appears immediately.
11. Troubleshooting checklist
When replication does not work, check in this order:
1. Is Versioning enabled on both buckets?
2. Does get-bucket-replication show the intended enabled rule, correct filter, destination ARN, and role ARN?
3. Can Amazon S3 assume the replication role, and does the role have the required source and destination S3 permissions?
4. Is the object eligible for the rule's prefix/tag filters and timing?
5. If the object uses SSE-KMS, is encrypted-object replication enabled, is a destination KMS key configured, and can the role use both KMS keys?
6. What does head-object report for ReplicationStatus?
7. Is the object old (created before live replication was configured)? Consider S3 Batch Replication.
8. Remember that replication can take time; an enabled rule is not proof that a specific object's status is COMPLETED.