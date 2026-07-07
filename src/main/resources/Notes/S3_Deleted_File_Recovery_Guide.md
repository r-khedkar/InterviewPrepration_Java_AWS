# How to Recover Deleted Files from AWS S3 Bucket

## ⚠️ Critical Distinction: Recovery Depends on Versioning

**THE SHORT ANSWER:**
- ✅ **With Versioning Enabled:** YES, you can recover 100% of deleted files (if deleted AFTER versioning was enabled)
- ❌ **Without Versioning:** NO, deleted files are permanently lost
- ⚠️ **If versioning was enabled AFTER file upload:** You can't recover files that don't have version history

---

## **SECTION 1: RECOVERY WHEN VERSIONING IS ENABLED**

### How S3 Versioning Works

When versioning is enabled and you delete an object, S3 doesn't actually remove it. Instead:

```
Action: User runs "delete file.txt"
Result: S3 creates a "delete marker" (a placeholder)
        Original file versions remain intact
        File appears deleted when you list bucket normally
        But all versions are still recoverable
```

### Method 1: Using AWS Console (Easiest)

**Step 1: Open S3 Console**
```
1. Go to AWS S3 Console
2. Select your bucket
3. Find the deleted file
4. Click "Show versions" (top right of object list)
```

**Step 2: View Delete Marker**
```
You'll see:
- Delete marker (latest version) - this is what deleted the file
- Previous versions of your file (older versions)
```

**Step 3: Restore the File**

**Option A: Remove the Delete Marker**
```
1. Select the delete marker (not the file versions)
2. Click "Delete"
3. Confirm deletion of the marker
4. File instantly comes back!
```

**Option B: Download Previous Version**
```
1. Select the version you want to restore
2. Click "Download"
3. Upload it back if needed
```

### Method 2: Using AWS CLI (Better for Automation)

#### Step 1: List All Versions of Deleted File

```bash
# Simple listing
aws s3api list-object-versions \
  --bucket my-bucket \
  --prefix important-file.csv

# Better formatted output
aws s3api list-object-versions \
  --bucket my-bucket \
  --prefix important-file.csv \
  --query "{ DeleteMarkers: DeleteMarkers[*].{VersionId: VersionId, IsLatest: IsLatest, Modified: LastModified}, Versions: Versions[*].{VersionId: VersionId, IsLatest: IsLatest, Modified: LastModified, Size: Size} }" \
  --output table
```

**Output Example:**
```
DeleteMarkers:
VersionId:  del-marker-abc123
IsLatest:   True
Modified:   2026-02-12T15:30:00Z

Versions:
VersionId:  ver-xyz789      (This is what you want)
IsLatest:   False
Modified:   2026-02-12T10:00:00Z
Size:       1048576

VersionId:  ver-def456
IsLatest:   False
Modified:   2026-02-11T08:00:00Z
Size:       1048000
```

#### Step 2: Remove Delete Marker (Fastest Recovery)

```bash
# This instantly "undeletes" the file
aws s3api delete-object \
  --bucket my-bucket \
  --key important-file.csv \
  --version-id 'del-marker-abc123'

# Verify file is back
aws s3 ls s3://my-bucket/ | grep important-file.csv
# Should now show the file!
```

#### Step 3: Restore Specific Previous Version (If Needed)

If you want a specific older version (not the most recent):

```bash
# Download specific version
aws s3api get-object \
  --bucket my-bucket \
  --key important-file.csv \
  --version-id 'ver-def456' \
  ./recovered-file.csv

# Make it the current version (copy back)
aws s3api copy-object \
  --bucket my-bucket \
  --key important-file.csv \
  --copy-source "my-bucket/important-file.csv?versionId=ver-def456"
```

### Method 3: Batch Recovery (Entire Directory Deleted)

If someone deleted an entire folder, you need to remove multiple delete markers:

#### Using Shell Script

```bash
#!/bin/bash

BUCKET="my-bucket"
PREFIX="deleted-folder/"
BATCH_SIZE=1000

# Step 1: List all delete markers in the folder
echo "Finding delete markers for $PREFIX..."
aws s3api list-object-versions \
  --bucket "$BUCKET" \
  --prefix "$PREFIX" \
  --query "DeleteMarkers[?IsLatest==\`true\`].{Key: Key, VersionId: VersionId}" \
  --output json > /tmp/delete-markers.json

TOTAL=$(jq 'length' /tmp/delete-markers.json)
echo "Found $TOTAL delete markers to remove"

# Step 2: Process in batches (S3 API limit is 1000 per request)
BUCKET="$BUCKET" BATCH_SIZE="$BATCH_SIZE" python3 << 'PYEOF'
import json
import os
import subprocess

with open('/tmp/delete-markers.json') as f:
    markers = json.load(f)

bucket = os.environ["BUCKET"]
batch_size = int(os.environ["BATCH_SIZE"])

for i in range(0, len(markers), batch_size):
    batch = markers[i:i + batch_size]
    
    delete_request = {
        "Objects": [
            {"Key": m["Key"], "VersionId": m["VersionId"]} 
            for m in batch
        ],
        "Quiet": True
    }
    
    with open('/tmp/batch-delete.json', 'w') as f:
        json.dump(delete_request, f)
    
    result = subprocess.run([
        'aws', 's3api', 'delete-objects',
        '--bucket', bucket,
        '--delete', 'file:///tmp/batch-delete.json'
    ], capture_output=True, text=True)
    
    if result.returncode == 0:
        print(f"✓ Batch {i//batch_size + 1}: Removed {len(batch)} delete markers")
    else:
        print(f"✗ Error: {result.stderr}")

print(f"\n✅ Restored all files in {PREFIX}")
PYEOF
```

#### Or Using Python Script (More Reliable)

```python
import boto3
from botocore.exceptions import ClientError

def restore_deleted_folder(bucket_name, prefix):
    """Restore all deleted files in a folder"""
    s3_client = boto3.client('s3')
    
    try:
        # Step 1: List all delete markers
        paginator = s3_client.get_paginator('list_object_versions')
        pages = paginator.paginate(Bucket=bucket_name, Prefix=prefix)
        
        delete_markers = []
        for page in pages:
            if 'DeleteMarkers' in page:
                for marker in page['DeleteMarkers']:
                    if marker.get('IsLatest', False):
                        delete_markers.append({
                            'Key': marker['Key'],
                            'VersionId': marker['VersionId']
                        })
        
        print(f"Found {len(delete_markers)} delete markers")
        
        # Step 2: Delete markers in batches of 1000
        batch_size = 1000
        for i in range(0, len(delete_markers), batch_size):
            batch = delete_markers[i:i + batch_size]
            
            response = s3_client.delete_objects(
                Bucket=bucket_name,
                Delete={'Objects': batch}
            )
            
            deleted = len(response.get('Deleted', []))
            print(f"✓ Batch {i//batch_size + 1}: Deleted {deleted} delete markers")
            
            if 'Errors' in response:
                for error in response['Errors']:
                    print(f"✗ Error: {error['Key']} - {error['Message']}")
        
        print(f"✅ Restoration complete! {len(delete_markers)} files recovered")
        
    except ClientError as e:
        print(f"❌ Error: {e}")

# Usage
restore_deleted_folder('my-bucket', 'deleted-folder/')
```

---

## **SECTION 2: RECOVERY WHEN VERSIONING IS DISABLED**

### ❌ Can You Recover? 

**Short Answer: NO**

When versioning is disabled:
- Deleted files are **permanently gone**
- No delete markers created
- No previous versions stored
- AWS support cannot recover (unless they have a backup)

### Your Only Options

1. **Contact AWS Support** (Low probability of success)
   - Explain the situation
   - Note deletion timestamp
   - AWS might have automated backups (not guaranteed)
   - Costs may apply

2. **Check for Backups**
   - Did you backup to another bucket?
   - Did you use S3 Cross-Region Replication?
   - Any third-party backup solutions?

3. **Data Recovery Services**
   - DriveSavers, Iron Mountain can attempt recovery
   - Expensive ($1,000+)
   - Success not guaranteed

---

## **SECTION 3: ENABLING VERSIONING (DO THIS NOW!)**

### Enable Versioning on Existing Bucket

#### Using Console

```
1. Go to S3 console
2. Select bucket
3. Click "Properties" tab
4. Find "Versioning" section
5. Click "Edit"
6. Select "Enable"
7. Save changes
```

#### Using AWS CLI

```bash
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled
```

#### Using Terraform/Infrastructure as Code

```hcl
resource "aws_s3_bucket_versioning" "example" {
  bucket = aws_s3_bucket.example.id
  
  versioning_configuration {
    status = "Enabled"
  }
}
```

#### Using Java/Spring Boot

```java
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;

public class S3VersioningManager {
    
    public static void enableVersioning(String bucketName) {
        S3Client s3 = S3Client.builder().build();
        
        try {
            PutBucketVersioningRequest request = PutBucketVersioningRequest.builder()
                .bucket(bucketName)
                .versioningConfiguration(VersioningConfiguration.builder()
                    .status("Enabled")
                    .build())
                .build();
            
            s3.putBucketVersioning(request);
            System.out.println("✓ Versioning enabled for: " + bucketName);
            
        } finally {
            s3.close();
        }
    }
    
    public static void main(String[] args) {
        enableVersioning("my-bucket");
    }
}
```

### Important: Cost Impact of Versioning

**Versioning increases storage costs because:**
- Old versions take up storage space
- You're charged for every version stored

**Example:**
```
Without versioning:
  10 GB of files × $0.023/GB = $0.23/month

With versioning (100 versions stored):
  10 GB × 100 versions × $0.023/GB = $23/month

That's 100x more storage cost!
```

### Solution: Implement Lifecycle Policy

Automatically delete old versions to save money:

#### Using Console

```
1. Select bucket
2. Go to "Management" tab
3. Click "Create lifecycle rule"
4. Name: "Delete old versions"
5. Apply to: "All objects in bucket"
6. Under "Previous versions":
   - Enable "Permanently delete previous versions"
   - Set to: "30 days" (or your preference)
7. Save
```

#### Using CLI

```bash
cat > lifecycle-policy.json << 'EOF'
{
  "Rules": [
    {
      "Id": "DeleteOldVersions",
      "Status": "Enabled",
      "NoncurrentVersionExpirationInDays": 30
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle-policy.json
```

#### Using Terraform

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "example" {
  bucket = aws_s3_bucket.example.id

  rule {
    id     = "DeleteOldVersions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 30
    }
  }
}
```

---

## **SECTION 4: ADVANCED PROTECTION - MFA Delete**

### What is MFA Delete?

MFA Delete requires multi-factor authentication (MFA) before anyone can delete objects. Even root account can't delete without MFA token.

### Enable MFA Delete

**Note:** Only possible via CLI, not console

```bash
# Step 1: Get your MFA device serial
aws iam list-mfa-devices

# Output: arn:aws:iam::123456789:mfa/user-mfa

# Step 2: Generate MFA token code (from your MFA device)
# Let's say code is: 123456

# Step 3: Enable MFA Delete
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::123456789:mfa/user-mfa 123456"
```

### With MFA Delete Enabled

Now deletions require:
```bash
# Delete needs MFA token
aws s3api delete-object \
  --bucket my-bucket \
  --key important-file.txt \
  --mfa "arn:aws:iam::123456789:mfa/user-mfa 654321"
```

---

## **SECTION 5: TROUBLESHOOTING RECOVERY ISSUES**

### Issue 1: "You don't have permission"

```
Error: User: arn:aws:iam::xxx is not authorized 
to perform s3:ListBucketVersions
```

**Fix:** Add IAM permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucketVersions",
        "s3:GetObjectVersion",
        "s3:DeleteObjectVersion"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

### Issue 2: "Versioning shows 0 matches"

**Cause:** Versioning was enabled AFTER the file was uploaded
- Files uploaded before versioning has no version history
- Only future versions are tracked

**Solution:** Can't recover. Enable versioning first next time.

### Issue 3: "Too many versions, listing times out"

```bash
# Use pagination to avoid timeout
aws s3api list-object-versions \
  --bucket my-bucket \
  --prefix important-folder/ \
  --max-items 100 \
  --starting-token <next-token>
```

### Issue 4: "Accidental deletion of delete marker"

If you accidentally deleted a delete marker:

```bash
# See if the delete marker still exists
aws s3api list-object-versions \
  --bucket my-bucket \
  --key accidental-file.txt

# If not found, the delete marker is gone
# The last actual version of the file is now "current"
# File is restored!
```

---

## **SECTION 6: BEST PRACTICES CHECKLIST**

### Immediate Actions (Do Today)

- [ ] Enable S3 versioning on all buckets with important data
- [ ] Set up lifecycle policies to delete old versions (if cost is concern)
- [ ] Enable MFA Delete for critical buckets
- [ ] Test recovery procedure on non-production bucket

### Ongoing Practices

- [ ] Review bucket versioning settings monthly
- [ ] Monitor storage costs from versioning
- [ ] Set up bucket access logging (CloudTrail)
- [ ] Use bucket policies to restrict delete operations
- [ ] Implement cross-region replication for DR
- [ ] Regular backups to separate bucket (additional safety net)

### Prevention of Accidental Deletes

**1. Use IAM Policies to Restrict Delete**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringNotLike": {
          "aws:username": "admin-*"
        }
      }
    }
  ]
}
```

**2. Require Versioning Check**

```bash
# Before any delete operation, verify versioning is on
VERSIONING=$(aws s3api get-bucket-versioning \
  --bucket my-bucket \
  --query 'Status' \
  --output text)

if [ "$VERSIONING" != "Enabled" ]; then
  echo "❌ ABORT: Versioning not enabled!"
  exit 1
fi

# Only then proceed with operations
aws s3 rm s3://my-bucket/file.txt
```

**3. Use S3 Object Lock (For Compliance)**

```bash
# Create bucket with Object Lock
aws s3api create-bucket \
  --bucket my-compliance-bucket \
  --region us-east-1 \
  --object-lock-enabled-for-bucket

# Files automatically protected from deletion
# Even root account can't delete before retention expires
```

---

## **SECTION 7: RECOVERY WORKFLOW DECISION TREE**

```
┌─ Is versioning enabled?
│
├─ YES ─┬─ Was file deleted AFTER versioning was enabled?
│       │
│       ├─ YES → ✅ Can recover 100%
│       │         Steps:
│       │         1. List object versions
│       │         2. Find delete marker or previous version
│       │         3. Remove delete marker OR copy version back
│       │
│       └─ NO → ❌ Cannot recover
│               (No version history before versioning enabled)
│
└─ NO ──┬─ Less than 24 hours ago?
        │
        ├─ YES → 📞 Contact AWS Support
        │         (They might have automated backups)
        │
        └─ NO ──→ ❌ Permanently lost
                   (Data recovery services are expensive)
```

---

## **SECTION 8: QUICK REFERENCE - Most Common Commands**

```bash
# ===== RECOVERY =====

# 1. List all versions of a file
aws s3api list-object-versions --bucket my-bucket --prefix file.txt

# 2. Remove delete marker (fastest recovery)
aws s3api delete-object --bucket my-bucket --key file.txt --version-id 'marker-id'

# 3. Download specific version
aws s3api get-object --bucket my-bucket --key file.txt --version-id 'version-id' recovered-file.txt

# 4. Restore specific version as current
aws s3api copy-object --bucket my-bucket --key file.txt --copy-source "my-bucket/file.txt?versionId=version-id"

# ===== SETUP & PREVENTION =====

# 5. Enable versioning
aws s3api put-bucket-versioning --bucket my-bucket --versioning-configuration Status=Enabled

# 6. Check versioning status
aws s3api get-bucket-versioning --bucket my-bucket

# 7. Set lifecycle to delete old versions
aws s3api put-bucket-lifecycle-configuration --bucket my-bucket --lifecycle-configuration file://policy.json

# 8. Check storage costs
aws s3api list-object-versions --bucket my-bucket --query 'sum(Versions[*].Size)' --output text
```

---

## **SUMMARY**

| Scenario | Recovery Possible? | Time to Recover | Cost |
|----------|------------------|-----------------|------|
| Versioning enabled, recent deletion | ✅ Yes (100%) | 1-5 minutes | $0 |
| Versioning enabled, old deletion | ✅ Yes (100%) | 1-5 minutes | $0 |
| Versioning disabled, recent | ⚠️ Maybe (AWS support) | Hours/Days | $0-$ |
| Versioning disabled, old | ❌ No | - | - |
| MFA Delete enabled | ✅ Yes but need MFA token | 5-10 minutes | $0 |

**Bottom Line:** Enable versioning on all important buckets TODAY. It's the only reliable way to recover from accidents.

---

**Remember:** 
- Versioning is your insurance policy
- Set and forget with lifecycle policies
- Test recovery procedure before you need it
- Automate everything!
