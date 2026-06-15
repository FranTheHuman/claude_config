# /minio:sdk — Generate Node.js/TypeScript SDK Integration Code

Generate complete, production-ready Node.js/TypeScript code for integrating
with the MinIO instance from the shalapp-api service.
Apply knowledge from `.claude/skills/minio-ops/SKILL.md`.

## Instructions

1. Read `.claude/skills/minio-ops/SKILL.md` section 5 (Node.js SDK Integration).
2. Always include `forcePathStyle: true` — this is non-negotiable for MinIO.
3. Use `http://minio:9000` as endpoint (Docker internal service name).
4. Generate complete TypeScript with imports, types, and error handling.
5. Always generate the `.env` variables needed alongside the code.

## Always generates

### Required .env variables

```bash
# MinIO / S3 configuration
S3_ENDPOINT=http://minio:9000          # Inside Docker Compose
# S3_ENDPOINT=http://127.0.0.1:9000   # For local development outside Docker
S3_ACCESS_KEY=your-service-account-key
S3_SECRET_KEY=your-service-account-secret
S3_BUCKET=shalapp
MINIO_PUBLIC_URL=http://127.0.0.1:9000  # For generating public URLs
```

### Base client (always included)

```typescript
// src/lib/storage.ts
import {
    S3Client,
    PutObjectCommand,
    GetObjectCommand,
    DeleteObjectCommand,
    HeadObjectCommand,
    ListObjectsV2Command,
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({
    endpoint: process.env.S3_ENDPOINT ?? 'http://minio:9000',
    region: 'us-east-1',      // MinIO ignores this; SDK requires it
    credentials: {
        accessKeyId: process.env.S3_ACCESS_KEY!,
        secretAccessKey: process.env.S3_SECRET_KEY!,
    },
    forcePathStyle: true,      // REQUIRED for MinIO — never remove
});

const BUCKET = process.env.S3_BUCKET ?? 'shalapp';
```

## Output Options (generated based on argument)

### `upload` — file upload with public URL return

```typescript
export async function uploadFile(
    key: string,
    body: Buffer | Uint8Array,
    contentType: string
): Promise<string> {
    await s3.send(new PutObjectCommand({
        Bucket: BUCKET,
        Key: key,
        Body: body,
        ContentType: contentType,
    }));
    // Returns public URL for logo/ and blog/ paths
    // Returns key for private paths (use getPresignedUrl for access)
    const isPublic = key.startsWith('logo/') || key.startsWith('blog/');
    if (isPublic) {
        return `${process.env.MINIO_PUBLIC_URL}/${BUCKET}/${key}`;
    }
    return key;
}
```

### `presigned` — generate pre-signed URL for private objects

```typescript
export async function getPresignedUrl(
    key: string,
    expiresInSeconds = 3600
): Promise<string> {
    return getSignedUrl(
        s3,
        new GetObjectCommand({ Bucket: BUCKET, Key: key }),
        { expiresIn: expiresInSeconds }
    );
}
```

### `delete` — delete an object

```typescript
export async function deleteFile(key: string): Promise<void> {
    await s3.send(new DeleteObjectCommand({ Bucket: BUCKET, Key: key }));
}
```

### `exists` — check if object exists

```typescript
export async function fileExists(key: string): Promise<boolean> {
    try {
        await s3.send(new HeadObjectCommand({ Bucket: BUCKET, Key: key }));
        return true;
    } catch {
        return false;
    }
}
```

### `multipart` — upload from Express multipart form (with multer)

```typescript
import multer from 'multer';
import { Request, Response } from 'express';

const upload = multer({ storage: multer.memoryStorage() });

export const uploadHandler = [
    upload.single('file'),
    async (req: Request, res: Response) => {
        if (!req.file) return res.status(400).json({ error: 'No file provided' });

        const key = `uploads/${Date.now()}-${req.file.originalname}`;
        const url = await uploadFile(key, req.file.buffer, req.file.mimetype);

        res.json({ url, key });
    },
];
```

## Arguments

Usage: `/minio:sdk [upload|presigned|delete|exists|multipart|all]`

Examples:
- `/minio:sdk` → generates full client with all operations
- `/minio:sdk upload` → only upload function
- `/minio:sdk presigned` → only pre-signed URL generation
- `/minio:sdk multipart` → multer + Express upload handler
- `/minio:sdk all` → complete storage.ts file with all operations + .env vars

## Critical reminders (included in every response)

```
⚠️  forcePathStyle: true is REQUIRED — without it the SDK prepends bucket
    name to the hostname and DNS fails (Error: getaddrinfo ENOTFOUND shalapp.minio)

⚠️  Inside Docker Compose: endpoint = http://minio:9000 (service name)
    Outside Docker (local dev): endpoint = http://127.0.0.1:9000

⚠️  MinIO ignores the region value but the SDK throws if it's missing
    Set to any string: 'us-east-1' works fine
```
