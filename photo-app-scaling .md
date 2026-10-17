# SnapShare: Scaling Plan

SnapShare is a photo-sharing app where users upload photos and scroll a feed of photos from people they follow.

## 1. Assumptions

- Registered users: 10,000,000
- Daily active users (DAU): 10% of registered users
- Each active user uploads **1 photo per day**
- Each active user views **50 feed pages per day**
- Average photo size: **2 MB**; each photo also gets a **50 KB thumbnail**
- 1 feed page view = 1 request to the backend
- Traffic is spread evenly over 24 hours for the average (1 day = 86,400 seconds)
- Peak traffic = **5×** the average
- Units are decimal (1 TB = 1,000,000 MB)
- Storage ignores replication and backups (real systems would multiply by about 3)

**Daily active users:** 10,000,000 × 10% = **1,000,000 DAU**

## 2. Estimates

### Uploads per second
- Uploads per day: 1,000,000 × 1 = 1,000,000
- Average: 1,000,000 / 86,400 ≈ **11.6 uploads/s**
- Peak (5×): 11.6 × 5 ≈ **58 uploads/s**

### Feed views per second
- Feed views per day: 1,000,000 × 50 = 50,000,000
- Average: 50,000,000 / 86,400 ≈ **579 views/s**
- Peak (5×): 579 × 5 ≈ **2,900 views/s**

### Photo storage per year
- Per photo: 2 MB + 0.05 MB = 2.05 MB
- Per day: 1,000,000 × 2.05 MB = 2,050,000 MB = **2.05 TB/day**
- Per year: 2.05 TB × 365 ≈ **748 TB/year (about 0.75 PB)**
  - Originals: 1,000,000 × 2 MB × 365 = 730 TB
  - Thumbnails: 1,000,000 × 0.05 MB × 365 = 18.25 TB
- Metadata (rows in the database, assuming about 1 KB per photo): about 1 GB/day, about 365 GB/year. This is tiny compared with the photo files.

## 3. Read-heavy or write-heavy?

**The system is read-heavy.** There are 50 million feed views per day against 1 million uploads, a read-to-write ratio of about **50:1**.

What this means for the design:
- Optimise the read path first: cache feed data and metadata, serve images from a CDN, and add database read replicas.
- Writes (about 58/s at peak) are modest, so a single primary database can handle them, and slow work such as thumbnails can be pushed to the background.
- Scale the read path horizontally by adding app servers, cache nodes and replicas; the write path can stay simple.
- Replicas can lag behind the primary, so a user may briefly not see a just-uploaded photo in some feeds. This is acceptable for a social feed.

## 4. Why photos should not live in the database

- At about 748 TB/year, photos would make the database enormous, slow to back up, slow to restore and expensive to scale.
- Database storage is costly and tuned for small structured rows, not large binary blobs.
- Every photo read would go through the database, competing with the queries that load feeds and users.
- Databases cannot serve files through a CDN, so images could not be cached near users.

**Where they go instead:** in **object storage** (such as Amazon S3 or Google Cloud Storage), which is cheap, scales to petabytes, is durable, and works with a CDN. The database stores only the **metadata and a URL/key** pointing to the file (photo ID, owner, caption, timestamp, object key).

## 5. Architecture diagram

```
                           +----------------+
                           |     Users      |
                           | (web / mobile) |
                           +--------+-------+
                                    |
              +---------------------+----------------------+
              | images (read)                              | API calls (feed, upload)
              v                                            v
     +-----------------+                          +------------------+
     |       CDN       |                          |  Load Balancer   |
     | (caches photos  |                          +--------+---------+
     |  and thumbnails)|                                   |
     +--------+--------+                    +--------------+--------------+
              | cache miss                  |              |              |
              v                             v              v              v
     +-----------------+              +---------+    +---------+    +---------+
     | Object Storage  |<-------------| App     |    | App     |    | App     |
     | (originals +    |  upload      | Server 1|    | Server 2|    | Server N|
     |  thumbnails)    |  photo file  +----+----+    +----+----+    +----+----+
     +--------+--------+                   |              |              |
              ^                            +--------------+--------------+
              |                               |           |           |
              | write thumbnail               | read/     | metadata  | enqueue
              |                               | write     | writes    | "make thumbnail"
              |                               v           v           v
     +--------+--------+               +-----------+  +---------+  +---------+
     | Thumbnail Worker|<--------------|   Queue   |  |  Cache  |  |         |
     | (resizes image) |  take job     +-----------+  | (Redis) |  |         |
     +-----------------+                              +----+----+  |         |
              |                                            | miss  |         |
              | update photo row                           v       |         |
              +-------------------------------> +-------------------+         
                                                |  Primary Database |
                                                |  (writes)         |
                                                +---------+---------+
                                                          | replication
                                                          v
                                                +-------------------+
                                                |   Read Replica    |
                                                |   (feed reads)    |
                                                +-------------------+
```

Simplified flow: `User -> CDN` for images; `User -> Load Balancer -> App Servers -> Cache -> Read Replica` for feeds; `App Servers -> Object Storage + Primary DB + Queue -> Worker -> Object Storage` for uploads.

## 6. Components (one sentence each)

- **CDN:** Solves slow image loading and heavy load on storage by caching photos and thumbnails at edge locations close to users.
- **Load balancer:** Solves the single-server bottleneck and single point of failure by spreading requests across many app servers and skipping unhealthy ones.
- **App servers:** Solve the need to run business logic (auth, feed building, upload handling) in a stateless way that can be scaled by simply adding more servers.
- **Cache (e.g. Redis):** Solves repeated, slow database reads by keeping hot data such as feed pages and photo metadata in memory.
- **Database (primary):** Solves the need for reliable, consistent storage of users, follows and photo metadata, and handles all writes.
- **Read replica:** Solves the read-heavy load on the primary by serving feed queries from a copy of the data.
- **Object storage:** Solves the problem of storing hundreds of terabytes of photo files cheaply and durably outside the database.
- **Queue:** Solves slow or spiky uploads by letting the app server hand off thumbnail work and respond to the user immediately.
- **Thumbnail worker:** Solves the cost of resizing images inside the request by creating the 50 KB thumbnails in the background, at a pace that can be scaled independently.

## 7. Upload flow, step by step

1. The user selects a photo in the app and taps upload; the request goes to the **load balancer**.
2. The load balancer forwards it to a healthy **app server**.
3. The app server authenticates the user and validates the file (type, size limit).
4. The app server stores the original file (about 2 MB) in **object storage** under a unique key.
5. The app server writes a metadata row (photo ID, user ID, caption, timestamp, object key, `thumbnail_status = pending`) to the **primary database**.
6. The app server puts a "create thumbnail" job (photo ID and object key) on the **queue**.
7. The app server returns "upload successful" to the user. The user does not wait for the thumbnail.
8. A **thumbnail worker** takes the job from the queue, downloads the original from object storage and resizes it to a thumbnail (about 50 KB).
9. The worker saves the thumbnail to object storage and updates the photo row (`thumbnail_status = ready`, thumbnail key).
10. If the job fails, it is retried (and moved to a dead-letter queue after repeated failures); until it is ready, the feed shows a placeholder or the original.
11. The cache entries for followers' feeds are invalidated or updated, so the new photo appears; the CDN then caches the thumbnail after the first request.

## 8. Trade-offs

1. **Caching vs. freshness.** Caching feeds and metadata greatly reduces database load, but cached data can be stale, so a new photo may appear a little late. Shorter cache lifetimes mean fresher data but more database load.
2. **Read replicas vs. consistency.** Replicas let us scale reads, but replication lag means a replica may be slightly behind the primary. We accept eventual consistency for feeds, but should read from the primary when a user views their own just-uploaded photo.
3. **Asynchronous thumbnails vs. simplicity.** A queue and worker make uploads fast and let us scale processing separately, but add moving parts (retries, monitoring, a window where the thumbnail is not ready) compared with resizing inside the request.
4. **CDN cost vs. speed.** A CDN gives fast image loads and protects storage, but costs money per GB served and can serve outdated images until they expire or are purged.
5. **Object storage vs. database for photos.** Object storage is cheap and scalable, but it adds a second system, and keeping the database row and the file in sync (for example when an upload fails halfway) needs care.
