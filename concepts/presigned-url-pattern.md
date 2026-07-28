This is **Phase 8 of System Design**: handling large media assets efficiently without choking your web servers or primary database.

When users upload heavy media (like 4K videos, high-res images, or large PDFs), handling that data stream incorrectly is one of the fastest ways to bring down an entire system.

Here is a clear breakdown of why the traditional approach fails, and how the **Presigned URL Pattern** solves it.

---

## 1. The Anti-Pattern: Uploading Through API Servers

In a naive architecture, the file travels from the user's browser, through your API server, and into your database or attached disk:

```
[ User Browser ] ──(Pushes 1 GB Video)──> [ API Server ] ──(Writes to DB/Disk)──> [ Database ]

```

### Why This Breaks Down

* **Network & Memory Bottleneck:** A 1 GB file stream consumes network bandwidth and RAM on your API server for the entire duration of the upload (which could take several minutes on slow connections).
* **Thread Exhaustion:** While the server spends minutes receiving and holding that open connection, that server worker thread cannot process any other requests. A few hundred simultaneous uploads will completely paralyze your API fleet.
* **Database Bloat:** Relational databases (like PostgreSQL or MySQL) are optimized for small, structured records (rows of numbers and text). Storing raw binary files (**BLOBs**) inside a database bloats backups, destroys query performance, and scales terribly.

---

## 2. The Solution: Direct-to-Object Storage (Presigned URLs)

Instead of passing heavy media through your API servers, you bypass them completely for the actual file transfer. You stream the file directly to **Object Storage** (like AWS S3 or Google Cloud Storage), which is designed specifically to store petabytes of binary data reliably and cheaply.

To do this securely without exposing public write access to your storage bucket, you use a **Presigned Upload URL**.

```
  ┌────────────────────────────────────────────────────────┐
  │ 1. Metadata: "video.mp4, 500MB"                        │
  │ 3. Return Presigned URL (e.g., s3.amazonaws.com/...)   │
  └───────────────────────────┬────────────────────────────┘
                              │
                              ▼
[ Client Browser ] ───────────────────────> [ File API Service ] ──> [ Database ]
        │                                                                │
        │ 2. Request Presigned URL                                       │
        │                                                                │
        │                                                                ▼
        │ 4. Stream 500MB File Directly                     ┌──────────────────────────┐
        └──────────────────────────────────────────────────>│ Object Storage (S3/GCS)  │
                                                            └──────────────────────────┘

```

---

## 3. Step-by-Step Execution Flow

### Step 1: Request Permission & Metadata

The client app (web or mobile) does **not** send the file yet. It sends a small JSON payload with metadata to the **File API Service**:

```json
{
  "filename": "vacation.mp4",
  "file_size": "524288000",
  "content_type": "video/mp4"
}

```

### Step 2: Register Record & Generate Presigned URL

The File API Service:

1. Writes an initial record in the primary database (e.g., status: `"upload_pending"`).
2. Calls the Object Storage API using its private cloud credentials to generate a **Presigned URL**.
* A presigned URL is a temporary, cryptographically signed link giving **write access to a specific path** in your bucket.
* It includes a short expiration time (e.g., valid for 15 minutes only).



### Step 3: Return the URL to Client

The API server responds to the client instantly with the signed upload link and moves on to serve other users:

```json
{
  "upload_url": "https://my-bucket.s3.amazonaws.com/uploads/123.mp4?AWSAccessKeyId=...&Signature=...&Expires=1700000000"
}

```

### Step 4: Direct Upload to Cloud Storage

The client issues an `HTTP PUT` request containing the raw video file **directly to the Presigned URL**.

* The 500 MB video streams directly over the internet into AWS S3 / Google Cloud Storage.
* Your API servers and primary database do **zero work** during this 500 MB transfer.

---

## 4. Why This Pattern Rules System Design

1. **Infinite Upload Scalability:** AWS S3 or Google Cloud Storage can easily handle millions of concurrent 1 GB uploads without slowing down.
2. **Cost-Effective:** Cloud storage costs fractions of a cent per gigabyte compared to primary database storage or high-performance compute disks.
3. **Enhanced Security:** You never expose your master storage credentials to the client. The client gets access to write *only* to a single isolated file path for a few minutes before the key expires.
