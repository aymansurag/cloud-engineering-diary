# S3 Lab 001 — Bucket and Versioning Recovery

## Goal

Create an S3 bucket and understand object versioning and recovery
from accidental deletion.

## Configuration

- Region: us-east-1
- ACLs: Disabled
- Block Public Access: Enabled
- Versioning: Enabled
- Encryption: AWS default

## What I learned

### Bucket
A bucket is a container for S3 objects.

### Object
An object is data stored in S3, such as a file, image, video, or backup.

### Key
A key identifies an object inside a bucket.

Example:

s3://bucket-name/server-backup.txt

Bucket:
bucket-name

Key:
server-backup.txt

## Recovery test

I uploaded `server-backup.txt`, then deleted it.

Because versioning was enabled, S3 created a delete marker while
the previous object version remained stored.

I enabled "Show versions", found the delete marker, deleted the
delete marker, and the original object became visible again.

## Important lesson

Versioning helps recover previous versions and accidental deletes,
but it does not completely prevent permanent deletion of object
versions.