---
name: django-storages-s3
description: >
  Apply this skill for ALL file storage work using AWS S3 or S3-compatible
  services (DigitalOcean Spaces, MinIO, Backblaze B2, Cloudflare R2).
  Covers: django-storages setup, public vs private buckets, CloudFront CDN,
  pre-signed URLs, direct browser uploads, file validation, multiple storage
  backends, IAM permissions, Celery-based file processing, and S3 + Django
  STORAGES config pattern (Django 4.2+ API). Use when any model has a FileField
  or ImageField, when handling uploads, serving downloads, or configuring media
  and static file backends.
---

# Django Storages + S3 Skill

> FIRST: Query Context7 `/websites/djangoproject_en_6_0` for current `STORAGES`
> dict API. Also check `django-storages` docs at https://django-storages.readthedocs.io
> because its settings evolve independently of Django.
> Context7 ID to resolve: search "django-storages boto3"

---

## 1. Decision Map — Which Storage Pattern?

```
┌─────────────────────────────────────────────────────────────────┐
│                  STORAGE PATTERN SELECTION                      │
├──────────────────────┬──────────────────────────────────────────┤
│ Static files         │ S3 + CloudFront CDN (served via CDN)     │
│ (CSS/JS/images)      │ whitenoise for simple/small deployments  │
├──────────────────────┼──────────────────────────────────────────┤
│ Public media         │ Public S3 bucket + CloudFront CDN        │
│ (avatars, thumbnails)│ AWS_QUERYSTRING_AUTH = False             │
│                      │ Cache-Control: max-age=86400             │
├──────────────────────┼──────────────────────────────────────────┤
│ Private media        │ Private S3 bucket (block all public)     │
│ (docs, invoices,     │ AWS_QUERYSTRING_AUTH = True              │
│  personal files)     │ Pre-signed URLs with short TTL (1 hour)  │
│                      │ NEVER serve from Django view (slow)      │
├──────────────────────┼──────────────────────────────────────────┤
│ Large file uploads   │ S3 pre-signed POST (direct browser→S3)   │
│ (video, datasets)    │ Bypass Django server entirely            │
│                      │ Celery task for post-upload processing   │
└──────────────────────┴──────────────────────────────────────────┘
```

---

## 2. Installation & Dependencies

```
# requirements/base.txt
django-storages[s3]     # Includes boto3
boto3                   # AWS SDK (pinned via pip-tools)
Pillow                  # ImageField support
python-magic            # File type validation by magic bytes
```

---

## 3. Settings — Full Production Configuration

```python
# config/settings/base.py
import environ
env = environ.Env()

# ── S3 Credentials ────────────────────────────────────────────────
# Prefer IAM role (EC2/ECS/Lambda) over access keys — no key rotation needed
# Only set access keys for local dev or non-AWS environments
AWS_ACCESS_KEY_ID = env("AWS_ACCESS_KEY_ID", default=None)
AWS_SECRET_ACCESS_KEY = env("AWS_SECRET_ACCESS_KEY", default=None)
AWS_S3_REGION_NAME = env("AWS_S3_REGION_NAME", default="us-east-1")

# ── Buckets ───────────────────────────────────────────────────────
AWS_STORAGE_BUCKET_NAME = env("AWS_STORAGE_BUCKET_NAME")       # Media (private)
AWS_PUBLIC_BUCKET_NAME = env("AWS_PUBLIC_BUCKET_NAME", default="")  # Public media
AWS_STATIC_BUCKET_NAME = env("AWS_STATIC_BUCKET_NAME", default="")  # Static files

# ── Security ──────────────────────────────────────────────────────
AWS_DEFAULT_ACL = None              # Inherit bucket policy — do NOT set public-read globally
AWS_S3_FILE_OVERWRITE = False       # Prevent overwrite — use unique paths
AWS_QUERYSTRING_AUTH = True         # Signed URLs — secure by default
AWS_QUERYSTRING_EXPIRE = 3600       # 1-hour signed URL TTL

# ── Encryption ────────────────────────────────────────────────────
AWS_S3_OBJECT_PARAMETERS = {
    "ServerSideEncryption": "AES256",  # S3-managed encryption at rest
}

# ── Transfer ──────────────────────────────────────────────────────
AWS_S3_SIGNATURE_VERSION = "s3v4"   # Required for all regions; v2 is legacy
AWS_S3_MAX_MEMORY_SIZE = 10 * 1024 * 1024  # 10MB before rolling to disk
AWS_S3_TRANSFER_CONFIG = {
    "multipart_threshold": 10 * 1024 * 1024,   # 10MB: use multipart above this
    "multipart_chunksize": 10 * 1024 * 1024,
    "max_concurrency": 10,
}

# ── CDN ───────────────────────────────────────────────────────────
AWS_S3_CUSTOM_DOMAIN = env("AWS_CLOUDFRONT_DOMAIN", default="")  # e.g. cdn.example.com
# CloudFront signed URLs (private CDN distribution):
AWS_CLOUDFRONT_KEY = env("AWS_CLOUDFRONT_KEY", default="").encode("ascii") or None
AWS_CLOUDFRONT_KEY_ID = env("AWS_CLOUDFRONT_KEY_ID", default=None)

# ── Django STORAGES dict (Django 4.2+ / Django 6 API) ─────────────
STORAGES = {
    # Default: private media files → S3
    "default": {
        "BACKEND": "apps.core.storage.PrivateS3Storage",
    },
    # Static files → S3 with manifest hashing
    "staticfiles": {
        "BACKEND": "storages.backends.s3boto3.S3ManifestStaticStorage",
        "OPTIONS": {
            "bucket_name": AWS_STATIC_BUCKET_NAME or AWS_STORAGE_BUCKET_NAME,
            "location": "static",
            "default_acl": "public-read",
            "querystring_auth": False,
            "object_parameters": {"CacheControl": "max-age=31536000, immutable"},
            "custom_domain": AWS_S3_CUSTOM_DOMAIN or None,
        },
    },
}
```

---

## 4. Custom Storage Backends

```python
# apps/core/storage.py
"""
Custom storage classes.
Rules:
- One class per access pattern (private, public, static)
- Storage classes are NEVER imported directly into models
- Use them as `storage=` kwarg on FileField/ImageField
- Configure via settings, not hardcoded bucket names
"""
from __future__ import annotations
from django.conf import settings
from storages.backends.s3boto3 import S3Boto3Storage


class PrivateS3Storage(S3Boto3Storage):
    """
    Default storage for sensitive user files.
    - Signed URLs only (no public access)
    - Files are private per bucket policy
    - URL TTL: 1 hour (configurable via AWS_QUERYSTRING_EXPIRE)
    """
    bucket_name: str = settings.AWS_STORAGE_BUCKET_NAME
    location: str = "media/private"
    default_acl = None          # Inherit bucket policy (which is block-all-public)
    file_overwrite: bool = False
    querystring_auth: bool = True
    querystring_expire: int = 3600
    object_parameters: dict = {
        "ServerSideEncryption": "AES256",
    }


class PublicS3Storage(S3Boto3Storage):
    """
    Storage for public assets: avatars, product images.
    Served via CloudFront CDN if configured, direct S3 otherwise.
    """
    bucket_name: str = settings.AWS_PUBLIC_BUCKET_NAME or settings.AWS_STORAGE_BUCKET_NAME
    location: str = "media/public"
    default_acl: str = "public-read"
    file_overwrite: bool = False
    querystring_auth: bool = False  # No signing needed — public bucket
    object_parameters: dict = {
        "CacheControl": "max-age=86400, public",  # 1-day CDN cache
    }


class DocumentS3Storage(S3Boto3Storage):
    """
    Storage for sensitive documents (invoices, contracts).
    Extra-short URL TTL for maximum security.
    """
    bucket_name: str = settings.AWS_STORAGE_BUCKET_NAME
    location: str = "media/documents"
    default_acl = None
    file_overwrite: bool = False
    querystring_auth: bool = True
    querystring_expire: int = 900  # 15-minute URLs for documents


# Singleton instances — use these on FileField storage= kwargs
private_storage = PrivateS3Storage()
public_storage = PublicS3Storage()
document_storage = DocumentS3Storage()
```

---

## 5. Model Integration — FileField & ImageField

```python
# apps/users/models.py
from __future__ import annotations
import uuid
from django.db import models
from apps.core.models import BaseModel
from apps.core.storage import public_storage, document_storage
from apps.core.validators import validate_file_type, validate_file_size


def user_avatar_path(instance: "UserProfile", filename: str) -> str:
    """
    Dynamic upload path generator.
    Rule: NEVER use a static path — always use the user/object ID.
    This prevents enumeration and collision.
    """
    from apps.core.storage import sanitize_filename
    ext = filename.rsplit(".", 1)[-1].lower()
    return f"avatars/{instance.user_id}/{uuid.uuid4()}.{ext}"


def user_document_path(instance: "UserDocument", filename: str) -> str:
    from apps.core.storage import sanitize_filename
    safe_name = sanitize_filename(filename)
    return f"documents/{instance.user_id}/{uuid.uuid4()}_{safe_name}"


class UserProfile(BaseModel):
    user = models.OneToOneField(
        "users.User", on_delete=models.CASCADE, related_name="profile"
    )
    avatar = models.ImageField(
        upload_to=user_avatar_path,
        storage=public_storage,    # Public: served via CDN
        blank=True,
        null=True,
        max_length=500,            # S3 paths can be long
    )

    @property
    def avatar_url(self) -> str | None:
        """Return the CDN or S3 URL for the avatar."""
        if not self.avatar:
            return None
        return self.avatar.url    # S3Boto3Storage.url() handles CDN or signed URL


class UserDocument(BaseModel):
    user = models.ForeignKey(
        "users.User", on_delete=models.PROTECT, related_name="documents"
    )
    file = models.FileField(
        upload_to=user_document_path,
        storage=document_storage,  # Private: 15-min signed URLs
        max_length=500,
    )
    original_filename = models.CharField(max_length=255)  # Store original name
    file_size = models.PositiveIntegerField()              # Store at upload time
    mime_type = models.CharField(max_length=100)          # Validated at upload time

    @property
    def download_url(self) -> str:
        """Returns a 15-minute pre-signed download URL."""
        return self.file.url
```

---

## 6. Serializers — File Upload Validation & Response

```python
# apps/users/serializers.py
from __future__ import annotations
from rest_framework import serializers
from apps.core.validators import validate_file_type, validate_file_size


class DocumentUploadSerializer(serializers.Serializer):
    """Input serializer for document upload."""
    file = serializers.FileField(max_length=500, allow_empty_file=False)
    document_type = serializers.ChoiceField(choices=["invoice", "contract", "id_proof"])

    def validate_file(self, file: object) -> object:
        # 1. Size check FIRST (cheap)
        validate_file_size(file, max_mb=20)
        # 2. Magic-byte type check (cannot be faked)
        validate_file_type(file, allowed_category="document")
        return file


class DocumentDetailSerializer(serializers.ModelSerializer):
    """Output serializer — returns signed download URL, not raw S3 path."""
    download_url = serializers.SerializerMethodField()

    class Meta:
        from apps.users.models import UserDocument
        model = UserDocument
        fields = ["id", "original_filename", "file_size", "mime_type",
                  "document_type", "download_url", "created_at"]
        read_only_fields = fields

    def get_download_url(self, obj: object) -> str:
        """Pre-signed URL generated fresh on every serialization."""
        return obj.download_url

    # ✅ NEVER expose obj.file.name (raw S3 key) to the client
    # ✅ NEVER expose bucket name or region
```

---

## 7. Service Layer — File Operations

```python
# apps/users/services.py
from __future__ import annotations
import logging
import magic
from typing import Any
from django.db import transaction
from django.contrib.auth import get_user_model

logger = logging.getLogger(__name__)
User = get_user_model()


def upload_user_document(
    *,
    user: Any,
    file: Any,
    document_type: str,
) -> Any:
    """
    Upload and persist a user document to S3.
    Returns the created UserDocument instance.
    """
    from apps.users.models import UserDocument

    # Read mime type BEFORE saving (file pointer at start)
    file.seek(0)
    mime = magic.from_buffer(file.read(1024), mime=True)
    file.seek(0)

    with transaction.atomic():
        doc = UserDocument.objects.create(
            user=user,
            file=file,               # django-storages handles the S3 upload
            original_filename=file.name,
            file_size=file.size,
            mime_type=mime,
            document_type=document_type,
        )
        logger.info(
            "Document uploaded",
            extra={"user_id": str(user.pk), "doc_id": str(doc.pk), "mime": mime},
        )
    return doc


def delete_user_document(*, document_id: str, user: Any) -> None:
    """Soft-delete document record AND delete the S3 object."""
    from apps.users.models import UserDocument
    from apps.core.exceptions import NotFoundError, PermissionDeniedError

    doc = UserDocument.objects.filter(pk=document_id).first()
    if not doc:
        raise NotFoundError("Document not found.")
    if doc.user_id != user.pk and not user.is_staff:
        raise PermissionDeniedError("Cannot delete this document.")

    with transaction.atomic():
        # Delete from S3
        _delete_s3_file(doc.file)
        # Soft-delete the record
        doc.soft_delete()
        logger.info("Document deleted", extra={"doc_id": document_id})


def _delete_s3_file(file_field: Any) -> None:
    """Delete file from S3. Logs error but doesn't raise — record deletion proceeds."""
    try:
        if file_field and file_field.name:
            file_field.delete(save=False)
    except Exception as exc:
        logger.error("Failed to delete S3 file: %s", exc, extra={"key": file_field.name})
```

---

## 8. Pre-Signed POST — Direct Browser Upload (Large Files)

```python
# apps/uploads/views.py
"""
Pre-signed POST pattern:
1. Client requests upload credentials from Django
2. Django generates pre-signed POST data from S3
3. Client POSTs DIRECTLY to S3 (bypasses Django server)
4. Client notifies Django when upload completes (callback)
5. Django Celery task validates and processes the file

Use for: videos, large datasets, high-frequency uploads
Benefit: no load on Django workers during file transfer
"""
from __future__ import annotations
import uuid
from typing import Any
import boto3
from botocore.config import Config
from django.conf import settings
from rest_framework.views import APIView
from rest_framework.request import Request
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated


class PresignedUploadView(APIView):
    """Generate pre-signed S3 POST credentials for direct browser upload."""
    permission_classes = [IsAuthenticated]

    ALLOWED_CONTENT_TYPES = {
        "video": ["video/mp4", "video/quicktime", "video/webm"],
        "image": ["image/jpeg", "image/png", "image/webp"],
        "document": ["application/pdf"],
    }
    MAX_FILE_SIZES = {
        "video": 500 * 1024 * 1024,    # 500 MB
        "image": 10 * 1024 * 1024,     # 10 MB
        "document": 20 * 1024 * 1024,  # 20 MB
    }

    def post(self, request: Request) -> Response:
        file_type = request.data.get("file_type", "")
        content_type = request.data.get("content_type", "")
        category = request.data.get("category", "document")

        allowed_types = self.ALLOWED_CONTENT_TYPES.get(category, [])
        if content_type not in allowed_types:
            return Response(
                {"error": {"code": "invalid_file_type", "message": f"Content type '{content_type}' not allowed."}},
                status=status.HTTP_400_BAD_REQUEST,
            )

        upload_key = f"uploads/{request.user.pk}/{uuid.uuid4()}/{file_type}"
        max_size = self.MAX_FILE_SIZES.get(category, 10 * 1024 * 1024)

        presigned = self._generate_presigned_post(
            key=upload_key,
            content_type=content_type,
            max_size=max_size,
        )

        return Response({
            "upload_url": presigned["url"],
            "upload_fields": presigned["fields"],
            "object_key": upload_key,
            "expires_in": 900,  # 15 minutes to complete the upload
        })

    def _generate_presigned_post(
        self, *, key: str, content_type: str, max_size: int
    ) -> dict[str, Any]:
        s3 = boto3.client(
            "s3",
            region_name=settings.AWS_S3_REGION_NAME,
            config=Config(signature_version="s3v4"),
        )
        return s3.generate_presigned_post(
            Bucket=settings.AWS_STORAGE_BUCKET_NAME,
            Key=key,
            Fields={
                "Content-Type": content_type,
                "x-amz-server-side-encryption": "AES256",
            },
            Conditions=[
                {"Content-Type": content_type},
                ["content-length-range", 1, max_size],
                {"x-amz-server-side-encryption": "AES256"},
            ],
            ExpiresIn=900,
        )


class UploadCompleteView(APIView):
    """
    Called by client after direct S3 upload completes.
    Triggers async processing task.
    """
    permission_classes = [IsAuthenticated]

    def post(self, request: Request) -> Response:
        object_key = request.data.get("object_key", "")
        if not object_key:
            return Response({"error": "object_key required"}, status=400)

        # Security: verify the key belongs to this user
        if not object_key.startswith(f"uploads/{request.user.pk}/"):
            return Response({"error": "Forbidden"}, status=403)

        from django.db import transaction
        from apps.uploads.tasks import process_uploaded_file
        from apps.uploads.models import PendingUpload

        with transaction.atomic():
            upload = PendingUpload.objects.create(
                user=request.user,
                object_key=object_key,
                status="pending",
            )
            transaction.on_commit(
                lambda: process_uploaded_file.delay(str(upload.pk))
            )

        return Response({"job_id": str(upload.pk)}, status=status.HTTP_202_ACCEPTED)
```

---

## 9. Celery Task — Post-Upload Processing

```python
# apps/uploads/tasks.py
from __future__ import annotations
import logging
from celery import shared_task
from celery.exceptions import SoftTimeLimitExceeded

logger = logging.getLogger(__name__)


@shared_task(
    bind=True,
    name="uploads.process_uploaded_file",
    max_retries=3,
    soft_time_limit=300,
    time_limit=360,
    acks_late=True,
    queue="uploads",
)
def process_uploaded_file(self, upload_id: str) -> dict:
    """
    Process a file after direct S3 upload:
    1. Verify file exists in S3
    2. Validate by downloading header bytes (magic bytes check)
    3. Generate thumbnail if image/video
    4. Move to permanent location
    5. Create final DB record
    """
    from apps.uploads.models import PendingUpload
    import boto3
    from django.conf import settings

    try:
        upload = PendingUpload.objects.get(pk=upload_id)
    except PendingUpload.DoesNotExist:
        logger.warning("PendingUpload not found: %s", upload_id)
        return {"status": "skipped"}

    try:
        s3 = boto3.client("s3", region_name=settings.AWS_S3_REGION_NAME)

        # 1. Verify object exists
        try:
            head = s3.head_object(
                Bucket=settings.AWS_STORAGE_BUCKET_NAME,
                Key=upload.object_key,
            )
        except s3.exceptions.ClientError:
            upload.status = "failed"
            upload.save(update_fields=["status", "updated_at"])
            return {"status": "failed", "reason": "object_not_found"}

        file_size = head["ContentLength"]

        # 2. Download first 1024 bytes for magic byte validation
        response = s3.get_object(
            Bucket=settings.AWS_STORAGE_BUCKET_NAME,
            Key=upload.object_key,
            Range="bytes=0-1023",
        )
        header_bytes = response["Body"].read()
        import magic
        detected_mime = magic.from_buffer(header_bytes, mime=True)

        # 3. Move to permanent location
        permanent_key = upload.object_key.replace("uploads/", "media/processed/")
        s3.copy_object(
            Bucket=settings.AWS_STORAGE_BUCKET_NAME,
            CopySource={"Bucket": settings.AWS_STORAGE_BUCKET_NAME, "Key": upload.object_key},
            Key=permanent_key,
            ServerSideEncryption="AES256",
        )
        s3.delete_object(Bucket=settings.AWS_STORAGE_BUCKET_NAME, Key=upload.object_key)

        # 4. Create final record
        from apps.users.services import create_document_from_upload
        create_document_from_upload(
            upload=upload,
            permanent_key=permanent_key,
            file_size=file_size,
            mime_type=detected_mime,
        )

        upload.status = "completed"
        upload.save(update_fields=["status", "updated_at"])
        return {"status": "completed"}

    except SoftTimeLimitExceeded:
        upload.status = "failed"
        upload.save(update_fields=["status"])
        raise
    except Exception as exc:
        logger.exception("Upload processing failed", extra={"upload_id": upload_id})
        raise self.retry(exc=exc)
```

---

## 10. IAM Policy — Least Privilege

```json
// Minimal IAM policy for Django application user
// NEVER use AmazonS3FullAccess — always least privilege
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DjangoAppS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:GetObjectVersion",
        "s3:HeadObject"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name/media/*",
        "arn:aws:s3:::your-bucket-name/uploads/*"
      ]
    },
    {
      "Sid": "StaticFilesAccess",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::your-static-bucket",
        "arn:aws:s3:::your-static-bucket/*"
      ]
    },
    {
      "Sid": "ListBucketForPresignedPost",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::your-bucket-name"
    }
  ]
}
```

---

## 11. S3-Compatible Backends (MinIO, DigitalOcean Spaces)

```python
# config/settings/base.py — MinIO (local/staging)
if env.bool("USE_MINIO", default=False):
    STORAGES["default"] = {
        "BACKEND": "storages.backends.s3boto3.S3Boto3Storage",
        "OPTIONS": {
            "endpoint_url": env("MINIO_ENDPOINT_URL", default="http://minio:9000"),
            "access_key": env("MINIO_ACCESS_KEY"),
            "secret_key": env("MINIO_SECRET_KEY"),
            "bucket_name": env("MINIO_BUCKET_NAME"),
            "region_name": "us-east-1",  # MinIO ignores this but boto3 requires it
            "default_acl": None,
            "file_overwrite": False,
            "querystring_auth": True,
            "signature_version": "s3v4",
        },
    }

# DigitalOcean Spaces
STORAGES["default"] = {
    "BACKEND": "storages.backends.s3boto3.S3Boto3Storage",
    "OPTIONS": {
        "endpoint_url": f"https://{env('DO_SPACES_REGION')}.digitaloceanspaces.com",
        "access_key": env("DO_SPACES_KEY"),
        "secret_key": env("DO_SPACES_SECRET"),
        "bucket_name": env("DO_SPACES_BUCKET"),
        "region_name": env("DO_SPACES_REGION"),
        "custom_domain": env("DO_SPACES_CDN_DOMAIN", default=""),
    },
}
```

---

## 12. Test Configuration — Local Storage in Tests

```python
# config/settings/testing.py
# Override S3 with local filesystem in tests — no real S3 calls
STORAGES = {
    "default": {
        "BACKEND": "django.core.files.storage.InMemoryStorage",
    },
    "staticfiles": {
        "BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage",
    },
}
```

---

## 13. Security Checklist

```
Bucket Policy:
✅ Block All Public Access enabled on private buckets
✅ Bucket policy denies s3:GetObject without signed URL
✅ Versioning enabled on buckets with critical data
✅ MFA Delete enabled for production buckets
✅ S3 Access Logging enabled

Application:
✅ All FileFields use storage= with explicit backend class
✅ Upload paths include user ID and UUID (not just filename)
✅ AWS_S3_FILE_OVERWRITE = False
✅ File type validated by magic bytes, not extension
✅ File size validated before S3 upload
✅ Signed URL TTL is short (15 min for docs, 1h for media)
✅ Raw S3 keys never exposed to clients in API responses
✅ Pre-signed POST conditions restrict content-type and size
✅ IAM user has minimum required S3 permissions only

Celery Processing:
✅ Uploaded files validated AGAIN in Celery (belt-and-suspenders)
✅ Files moved from temp upload prefix to permanent on success
✅ Temp files deleted from S3 on processing failure
```
