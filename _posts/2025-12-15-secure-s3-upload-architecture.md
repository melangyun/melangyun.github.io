---
title: Secure File Upload Architecture with S3 Presigned URLs
date: 2025-12-15 10:00:00 +0900
categories: [Dev, AWS]
tags: [s3, presigned-url, security, architecture, cloudfront]
---

## Background

I recently looked into the file upload architecture used in our company's notice system. I found it really well-designed from a security perspective, so I decided to write this post to organize what I learned.

This architecture solves several common problems:

- **Server load**: Uploading large files through API servers is expensive
- **Security**: Direct S3 bucket access can be abused
- **Cleanup**: Orphaned files waste storage
- **Access control**: Different admin roles need different upload limits

Let me walk through how it works.

---

## Architecture Overview

![Architecture](/assets/img/posts/2025-12-15-secure-s3-upload/architecture.png)

---

## Step-by-Step Flow

### Step 1: Request Presigned URL

The client requests a presigned URL from the **Storage API**.

```http
POST /api/storage/presigned-url
Content-Type: application/json
Authorization: Bearer <token>

{
  "fileName": "photo.jpg",
  "contentType": "image/jpeg",
  "fileSize": 1048576
}
```

The Storage API validates the request and returns:

```json
{
  "uploadId": "uuid-1234-5678",
  "presignedUrl": "https://temp-bucket.s3.amazonaws.com/...",
  "expiresAt": "2025-12-15T11:00:00Z"
}
```

#### Security checks at this stage:

| Check | Description |
|-------|-------------|
| **File type restriction** | Only allow specific MIME types (e.g., `image/*`, `application/pdf`) |
| **File size limit** | Enforce maximum file size per user role |
| **Rate limiting** | Prevent abuse by limiting presigned URL requests |
| **User-based quotas** | Different limits based on user role or subscription tier |

> The presigned URL includes conditions that S3 enforces during upload. If the client tries to upload a different file type or size, S3 will reject it.
{: .prompt-info }

---

### Step 2: Direct Upload to S3

The client uploads the file **directly to S3** using the presigned URL.

```javascript
const uploadFile = async (presignedUrl, file) => {
  const response = await fetch(presignedUrl, {
    method: 'PUT',
    body: file,
    headers: {
      'Content-Type': file.type,
    },
  });

  if (!response.ok) {
    throw new Error('Upload failed');
  }
};
```

#### Benefits of direct upload:

- **No server bottleneck**: Files go directly to S3, not through your API
- **Better performance**: Leverage S3's global infrastructure
- **Cost savings**: Reduced bandwidth and compute costs on your servers

---

### Step 3: Submit Notice with Temporary URLs

After uploading, the admin client submits the notice content with the temporary S3 URLs.

```http
POST /api/notices
Content-Type: application/json
Authorization: Bearer <admin_token>

{
  "title": "System Maintenance Notice",
  "content": "We will be performing scheduled maintenance...",
  "attachments": [
    {
      "uploadId": "uuid-1234-5678",
      "tempUrl": "https://temp-bucket.s3.amazonaws.com/admin123/uuid-1234-5678/maintenance-schedule.png"
    }
  ]
}
```

---

### Step 4: Validate and Copy Objects

The **Notice API** receives the request and asks **Storage API** to copy files from the temp bucket to the permanent notice bucket.

```http
POST /api/storage/copy
Content-Type: application/json

{
  "uploadId": "uuid-1234-5678",
  "adminId": "admin123",
  "destination": "notices/notice-9999/"
}
```

#### Critical validation at this stage:

| Validation | Why it matters |
|------------|----------------|
| **Upload ID verification** | Ensures the file was uploaded by this admin, not stolen from another admin's temp storage |
| **File existence check** | Confirms the file actually exists in temp bucket |
| **Content-Type re-validation** | Double-check the actual file type matches what was claimed |
| **Magic bytes verification** | Verify the file content matches its claimed type |

> **Why is Upload ID verification critical?**
> Without it, a malicious admin could submit another admin's temp URL in their notice, effectively hijacking someone else's uploaded file.
{: .prompt-warning }

---

### Step 5: Object Copy and URL Transformation

The Storage API performs the S3 object copy and returns CloudFront URLs.

```kotlin
// Storage API internal logic
fun copyObject(uploadId: String, adminId: String, destination: String): String {
    // 1. Verify ownership
    val uploadRecord = uploadRepository.findByUploadIdAndAdminId(uploadId, adminId)
        ?: throw ForbiddenException("Upload not found or not owned by admin")

    // 2. Verify file exists
    s3Client.headObject(
        HeadObjectRequest.builder()
            .bucket(TEMP_BUCKET)
            .key(uploadRecord.tempKey)
            .build()
    )

    // 3. Validate magic bytes
    val fileHead = s3Client.getObjectAsBytes(
        GetObjectRequest.builder()
            .bucket(TEMP_BUCKET)
            .key(uploadRecord.tempKey)
            .range("bytes=0-11")
            .build()
    )

    if (!validateMagicBytes(fileHead.asByteArray(), uploadRecord.contentType)) {
        throw BadRequestException("File content does not match declared type")
    }

    // 4. Copy to permanent bucket
    val destinationKey = "$destination/${uploadRecord.fileName}"
    s3Client.copyObject(
        CopyObjectRequest.builder()
            .sourceBucket(TEMP_BUCKET)
            .sourceKey(uploadRecord.tempKey)
            .destinationBucket(NOTICE_BUCKET)
            .destinationKey(destinationKey)
            .build()
    )

    // 5. Return CloudFront URL
    return "https://cdn.example.com/$destinationKey"
}
```

---

## Why Use a Temp Bucket?

The temp bucket with S3 lifecycle rules is crucial for **automatic cleanup**.

```json
{
  "Rules": [
    {
      "ID": "DeleteTempFiles",
      "Status": "Enabled",
      "Expiration": {
        "Days": 1
      }
    }
  ]
}
```

### Scenarios handled by lifecycle rules:

| Scenario | Without lifecycle | With lifecycle |
|----------|------------------|----------------|
| Admin uploads but never submits notice | Files accumulate forever | Auto-deleted after 1 day |
| Admin abandons draft | Orphaned files | Auto-cleaned |
| Failed/interrupted uploads | Partial files remain | Auto-cleaned |

> This "upload first, commit later" pattern with automatic cleanup is a clean way to handle the inherent uncertainty of admin-driven uploads.
{: .prompt-tip }

---

## Why Use CloudFront?

I also learned about CloudFront while studying this architecture. Here's what I found useful.

### Keeping S3 Private with OAC

Instead of making S3 bucket public, we can use **Origin Access Control (OAC)** to allow only CloudFront to access S3.

```
❌ Without OAC:
https://my-bucket.s3.amazonaws.com/image.jpg
→ S3 URL exposed
→ Users can bypass CloudFront
→ Higher cost, less secure

✅ With OAC:
https://cdn.example.com/image.jpg
→ S3 bucket stays private
→ Only CloudFront can access S3
```

### OAI vs OAC

Both achieve the same goal: **keep S3 private and allow only CloudFront to access it**. But they work differently.

#### OAI (Origin Access Identity) - Legacy

CloudFront creates a "special virtual user" to access S3.

```json
// S3 bucket policy with OAI
{
  "Principal": {
    "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity E1234567890"
  }
}
```

Problems:
- CloudFront-specific ID, separate from IAM
- No SSE-KMS (S3 encryption) support
- Regional limitations

#### OAC (Origin Access Control) - Recommended

CloudFront uses standard IAM signing (SigV4) to access S3.

```json
// S3 bucket policy with OAC
{
  "Principal": {
    "Service": "cloudfront.amazonaws.com"
  },
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::123456789:distribution/E1234567890"
    }
  }
}
```

Benefits:
- IAM-based, consistent with other AWS services
- SSE-KMS supported
- All regions supported

#### Summary

| | OAI | OAC |
|---|---|---|
| **How it works** | CloudFront-specific special ID | IAM signing (SigV4) |
| **S3 encryption (SSE-KMS)** | Not supported | Supported |
| **AWS recommendation** | Legacy | Recommended |

> If you're setting up a new CloudFront distribution, use OAC. OAI is only for legacy systems.
{: .prompt-info }

### Caching Strategy

CloudFront caches files at **edge locations** worldwide for faster delivery.

```
1st request: CloudFront → fetches from S3 → caches at edge
2nd request: Served directly from edge (no S3 call)
```

#### Common caching settings:

| Setting | Description |
|---------|-------------|
| **TTL (Time To Live)** | How long to keep cache. e.g., 86400s = 24 hours |
| **Cache-Control header** | Set on S3 object. e.g., `max-age=31536000` (1 year) |
| **Invalidation** | Force delete cache. Used when file is updated |

#### Recommended strategy for notice images:

Notice images rarely change after upload, so:
- Set long TTL (e.g., 1 year)
- Include hash in filename (e.g., `image-abc123.jpg`)
- Upload as new file when updating

This way, you don't need to worry about cache invalidation.

---

## Magic Bytes Validation

**Magic bytes** (also called file signatures) are the first few bytes of a file that identify its true format.

### Common magic bytes:

| Format | Magic Bytes (Hex) | ASCII |
|--------|------------------|-------|
| JPEG | `FF D8 FF` | ÿØÿ |
| PNG | `89 50 4E 47` | ‰PNG |
| GIF | `47 49 46 38` | GIF8 |
| PDF | `25 50 44 46` | %PDF |
| ZIP | `50 4B 03 04` | PK.. |

### Why is this important?

```
Attack scenario:
1. Attacker requests presigned URL for "image.jpg"
2. Uploads malware.exe renamed to image.jpg
3. Sets Content-Type: image/jpeg
4. Presigned URL validation passes ✓
5. File is served to other users...

With magic bytes validation:
→ Server reads first few bytes
→ Finds "MZ" (Windows executable signature)
→ Rejects the file ✗
```

### Implementation example:

```kotlin
val MAGIC_BYTES = mapOf(
    "image/jpeg" to byteArrayOf(0xFF.toByte(), 0xD8.toByte(), 0xFF.toByte()),
    "image/png" to byteArrayOf(0x89.toByte(), 0x50, 0x4E, 0x47),
    "image/gif" to byteArrayOf(0x47, 0x49, 0x46, 0x38),
    "application/pdf" to byteArrayOf(0x25, 0x50, 0x44, 0x46),
)

fun validateMagicBytes(buffer: ByteArray, declaredType: String): Boolean {
    val expected = MAGIC_BYTES[declaredType]
        ?: return true  // Unknown type, skip validation

    return expected.withIndex().all { (i, byte) -> buffer[i] == byte }
}
```

---

## Additional Security Considerations

These are optional depending on your requirements. In our case, only admins upload notices, so the traffic is very low. Some of these might be over-engineering for admin-only systems.

### Malware Scanning

For high-security applications, integrate virus scanning before copying to the permanent bucket.

Options:
- **AWS GuardDuty for S3** - Native AWS solution
- **ClamAV on Lambda** - Open source, triggered by S3 events
- **Third-party services** - Trend Micro, Sophos, etc.

### Transaction Consistency

What if the notice creation fails after files are copied? You could implement:

- **Delayed cleanup**: Add lifecycle rules to notice bucket + cleanup job for unconfirmed files
- **Saga pattern**: If notice creation fails, delete copied files as compensation

In practice, temp bucket lifecycle already handles most orphan files.

### Rate Limiting

Limit presigned URL requests per admin role. Our system already has this implemented at the API gateway level.

### Audit Logging

Track file operations for security analysis (upload events, admin ID, file metadata, timestamps, IP addresses).

---

## Summary

| Component | Purpose |
|-----------|---------|
| **Presigned URL** | Secure, time-limited direct upload to S3 |
| **Temp bucket + Lifecycle** | Automatic cleanup of orphaned files |
| **Upload ID** | Ownership verification to prevent file hijacking |
| **Magic bytes validation** | Detect disguised malicious files |
| **Object copy** | Move confirmed files to permanent notice storage |
| **CloudFront** | Fast, cached delivery of files |

This architecture provides:

- **Scalability**: Direct-to-S3 uploads bypass your servers
- **Security**: Multiple validation layers prevent abuse
- **Cost efficiency**: Auto-cleanup prevents storage waste
- **Flexibility**: Role-based limits and quotas

---

## Conclusion

While studying this architecture, I learned a lot about S3 security best practices:

- **Presigned URLs** allow direct uploads without exposing credentials
- **Temp bucket + Lifecycle** is a simple but effective cleanup strategy
- **Upload ID verification** prevents file hijacking between users
- **Magic bytes validation** catches disguised malicious files

The separation of concerns is also clean:
- **Storage API** handles all S3 operations
- **Notice API** handles business logic
- **S3 lifecycle** handles cleanup
- **CloudFront** handles delivery

I hope this post helps someone who wants to learn about secure file upload patterns.

---

## References

- [AWS - Uploading objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)
- [AWS - S3 Lifecycle Configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [AWS - Origin Access Control (OAC)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- [AWS - CloudFront Caching](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ConfiguringCaching.html)
- [OWASP - Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
- [List of file signatures - Wikipedia](https://en.wikipedia.org/wiki/List_of_file_signatures)
