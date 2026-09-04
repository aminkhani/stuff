# Complete Interview Guide

A framework for thinking through *any* large-scale system, taught through YouTube as the running example.

---
## 1. The Meta-Skill: How to Think in a System Design Interview

Every system design problem, no matter the product, follows the same skeleton. Memorize this order, not the answer:

1. **Clarify requirements** (functional + non-functional) — 5 min
2. **Estimate scale** (back-of-envelope math) — 5 min
3. **Define the API** (contract between client and server) — 5 min
4. **High-level design** (boxes and arrows) — 10 min
5. **Deep dive** into 2–3 components the interviewer cares about — 15–20 min
6. **Identify bottlenecks and trade-offs** — ongoing
7. **Wrap up**: scaling, monitoring, failure modes — 5 min

**The #1 mistake candidates make:** jumping straight to architecture diagrams. The interviewer is testing whether you can scope a vague problem before you solve it. Silence at the start, spent asking good questions, is worth more than fast whiteboarding.

**The #1 signal interviewers look for:** trade-off reasoning, not "the right answer." There is no single correct YouTube architecture — there's a defensible one given the constraints you chose.

---
## 2. Requirements Gathering — The Questions to Ask

Never assume. Say these out loud in an interview; they show you understand ambiguity is the real problem.
### Functional questions to ask the interviewer
- Can users **upload** videos? Any format/size limits?
- Can users **watch/stream** videos? Live streaming too, or just VOD (video-on-demand)?
- Do we need **search**?
- Do we need **comments, likes, subscriptions, notifications**?
- Do we need a **recommendation feed** (home page)?
- Multiple **resolutions/bitrates** (adaptive streaming)?
- Is this **global**? Multi-region?

### Non-functional questions to ask
- What's the **read:write ratio**? (YouTube is read-heavy: ~massively more views than uploads)
- What's more important — **availability or consistency**? (For view counts: availability. For payments/ads: consistency.)
- **Latency** target for starting playback?
- Do we need **strong durability** guarantees on uploaded video (never lose a video)?

### For a 45-min interview, scope down explicitly
Say something like:
> "I'll focus on: video upload, transcoding, storage, and playback/streaming, plus basic metadata and view counts. I'll treat recommendations, live streaming, and comments as out of scope unless we have time — is that a fair scope?"

This single sentence demonstrates seniority. It shows you can *prioritize*, which is the actual skill being tested.

---

## 3. Functional Requirements (assumed scope for this guide)

- Users can **upload** videos.
- Users can **watch** videos with adaptive bitrate streaming.
- Users can **search** for videos.
- System tracks **views, likes, comments**.
- System **recommends** related videos.

## 4. Non-Functional Requirements

| Requirement | Target |
|---|---|
| Availability | High (99.9%+) — prefer availability over strict consistency for views/likes |
| Latency | Video should start playing in <2s |
| Durability | Uploaded videos must never be lost |
| Scalability | Billions of videos, billions of daily views |
| Consistency | Eventual consistency acceptable for view counts, comments |

---

## 5. Back-of-Envelope Capacity Estimation

This is where many backend engineers actually shine — do the math out loud.

**Assumptions (state your own, round numbers are fine):**
- 2 billion daily active users (DAU)
- Each user watches ~5 videos/day, average watch time 5 min
- 500 hours of video uploaded per minute (real YouTube-scale number, useful to know)

**Storage:**
- 500 hours/min uploaded × 60 min/hr × 24 hr = ~720,000 hours/day
- At ~1 GB/hour for compressed 1080p video: ~720 TB/day raw
- Multiply by ~5 for storing multiple resolutions/formats (adaptive streaming): ~3.6 PB/day
- Over a year: **~1.3 exabytes/year** — this is *why* YouTube needs distributed object storage + CDN, not a single database.

**Bandwidth (read-heavy):**
- 2B users × 5 videos × 5 min × (bitrate ~3 Mbps for average quality) → this quickly reaches **hundreds of Tbps** aggregate — this is why a CDN is non-negotiable, not optional.

**Takeaway to say in the interview:** "Storage grows linearly and is solvable with cheap object storage; the real bottleneck is *egress bandwidth* for playback, which is why CDN placement matters more than database choice."

---

## 6. API Design (high-level contract)

```
POST   /videos                     -> initiate upload, returns upload URL + video_id
PUT    /videos/{video_id}/complete -> mark upload finished, triggers processing
GET    /videos/{video_id}          -> metadata (title, description, views, etc.)
GET    /videos/{video_id}/stream   -> returns manifest (HLS/DASH) for playback
GET    /search?q=                  -> search results
POST   /videos/{video_id}/comments -> add comment
POST   /videos/{video_id}/like     -> like a video
GET    /feed                       -> personalized recommendations
```

Design note: uploads use **pre-signed URLs** direct to object storage (S3-style), not through your app servers — this avoids your API layer becoming a bandwidth bottleneck.

---

## 7. High-Level Architecture

```
                     ┌──────────────┐
                     │   Client     │
                     └──────┬───────┘
                            │
                    ┌───────▼────────┐
                    │  API Gateway /  │
                    │  Load Balancer  │
                    └───────┬────────┘
           ┌────────────────┼─────────────────┐
           ▼                ▼                 ▼
   ┌───────────────┐ ┌─────────────┐  ┌───────────────┐
   │ Upload Service │ │ Metadata Svc│  │ Search Service │
   └───────┬────────┘ └──────┬──────┘  └───────┬────────┘
           ▼                 ▼                 ▼
   ┌───────────────┐ ┌─────────────┐  ┌───────────────┐
   │ Blob Storage   │ │  Metadata DB│  │ Elasticsearch  │
   │ (raw uploads)  │ │ (SQL/NoSQL) │  │ / OpenSearch   │
   └───────┬────────┘ └─────────────┘  └───────────────┘
           ▼
   ┌───────────────┐
   │ Transcoding    │  (message queue triggers this)
   │ Workers        │
   └───────┬────────┘
           ▼
   ┌───────────────┐
   │ Object Storage │  (encoded videos, multiple resolutions)
   └───────┬────────┘
           ▼
   ┌───────────────┐
   │      CDN       │ ◄── clients stream from here, not origin
   └───────────────┘
```

---

## 8. Deep Dive #1: Video Upload & Transcoding Pipeline

This is the most commonly probed deep-dive. Walk through it as a **pipeline**, not a single service.

1. **Client requests an upload URL** from the Upload Service → gets a pre-signed URL to object storage (e.g., S3) and a `video_id`.
2. **Client uploads directly to storage** (bypasses app servers — critical for scale). Large files use **chunked/multipart upload** so a dropped connection doesn't restart the whole upload.
3. Upload completion triggers a message onto a **queue** (Kafka/SQS).
4. **Transcoding workers** pick up the job:
   - Validate the file (virus scan, format check — *this matters a lot given your security background*).
   - Transcode into multiple resolutions (144p–4K) and formats (H.264, VP9, AV1) for adaptive bitrate streaming.
   - Generate thumbnails, extract captions if present.
   - Split into small chunks (HLS `.ts` segments or DASH segments) with a manifest file (`.m3u8` / `.mpd`).
5. Encoded outputs pushed to **object storage**, then replicated to **CDN edge nodes** on first request (or pre-pushed for popular content).
6. Metadata service updated: video status → "ready", triggers notification to subscribers.

**Why chunked adaptive streaming?** The client's player switches bitrate mid-playback based on measured bandwidth — this is what prevents buffering. Mention **HLS** (Apple, widely supported) or **MPEG-DASH** by name; interviewers like specificity.

**Failure handling to mention:** transcoding jobs should be idempotent and retryable; use a dead-letter queue for jobs that fail repeatedly; store upload state so a failed transcode doesn't silently orphan a video.

---

## 9. Deep Dive #2: Playback & CDN Strategy

- Client requests `GET /videos/{id}/stream` → gets a manifest listing available bitrates and CDN URLs for segments.
- Player fetches segments from the **nearest CDN edge**, not your origin servers.
- **Cache popular content aggressively**; use pull-based CDN caching for long-tail (rarely watched) videos, and push/pre-warm for viral content.
- Origin shield layer between CDN and object storage to avoid a "thundering herd" hitting storage directly when cache misses spike.

**Trade-off to voice:** Pre-transcoding every video into every resolution the moment it's uploaded wastes compute for videos nobody watches. A common real-world optimization: transcode a couple of default resolutions immediately, transcode higher/rarer resolutions **on-demand** (lazily) the first time they're requested, then cache the result.

---

## 10. Deep Dive #3: Metadata Storage & View Counts

**Metadata DB choice:** relational (Postgres/MySQL) for core video metadata (title, owner, upload date) works fine at moderate scale with sharding by `video_id`. At YouTube's actual scale, a NoSQL wide-column store (Bigtable/Cassandra) is used because of massive write throughput and simple key-based access patterns.

**View counts — the classic follow-up question**: "Do you increment a counter in the database on every single view?" **No** — that's a write-amplification disaster at billions of views/day.

Standard answer:
1. View events go to a **message queue** (Kafka), not directly to the DB.
2. A stream processor (Flink/Spark Streaming) **batches and aggregates** counts (e.g., every few seconds or minutes).
3. Aggregated counts are periodically flushed to the metadata store.
4. Read path shows an **eventually consistent** count — this is fine; nobody notices a view counter is a few seconds stale.

This demonstrates you understand **eventual consistency as a deliberate trade-off**, not a limitation.

---

## 11. Deep Dive #4: Search & Recommendations (lighter treatment, mention if time allows)

- **Search**: index video metadata (title, description, tags, transcript) in **Elasticsearch/OpenSearch**. Update index asynchronously after upload/edit via the same event queue pattern.
- **Recommendations**: offline batch job (collaborative filtering / embeddings) computes candidate lists per user, stored in a fast key-value cache (Redis); served with a lightweight re-ranking model at request time. Full ML pipeline is usually out of scope — name-drop the pattern (candidate generation → ranking) and move on unless the interviewer pushes further.

---

## 12. Scaling Building Blocks (apply everywhere)

| Technique | Where it applies |
|---|---|
| **Load balancer** | In front of all stateless services |
| **Horizontal sharding** | Metadata DB, sharded by `video_id` or `user_id` |
| **Caching (Redis/Memcached)** | Hot metadata, video manifests, recommendation lists |
| **CDN** | All video segment delivery |
| **Message queues (Kafka)** | Decouple upload → transcode → notify; absorb traffic spikes |
| **Read replicas** | Metadata DB reads (views, comments are read-heavy) |
| **Async processing** | Anything not on the critical path of "user sees a response" |

---

## 13. Security Considerations (worth leading with, given your background)

This is an area most candidates skip — bringing it up unprompted is a strong differentiator for you specifically:

- **Upload validation**: virus/malware scanning before transcoding; strict MIME-type and file-signature checks, not just extension checks.
- **Pre-signed URL scoping**: short expiry, scoped to a single object, to prevent URL replay/abuse.
- **DRM / signed CDN URLs**: prevent hotlinking and unauthorized redistribution of paid/licensed content (signed cookies or tokenized CDN URLs with short TTL).
- **Rate limiting** on upload and comment/like endpoints to prevent abuse and scraping.
- **Content moderation pipeline**: automated (hash-matching against known-bad content, ML classifiers) + human review queue, running async in the transcoding pipeline.
- **Access control**: private/unlisted video visibility enforced at the metadata + CDN token layer, not just the app layer — a leaked direct storage URL shouldn't bypass visibility rules.
- **PII handling**: user watch history, comments — encryption at rest, scoped access for internal services (relevant if this ever comes up as a follow-up).

---

## 14. Common Interviewer Follow-Ups (practice these)

- "What if a video goes viral in the middle of transcoding a lower priority queue?" → priority queues / autoscaling transcoding workers.
- "How do you avoid recomputing transcodes for duplicate uploads?" → content hashing (perceptual hash) to detect duplicates before transcoding.
- "How would you support live streaming?" → different pipeline: ingest via RTMP, segment in near-real-time, push to CDN with very short segment durations; trade quality for latency.
- "Single point of failure?" → walk through each box in your diagram and name its redundancy strategy.
- "How do you roll back a bad transcode?" → keep raw upload in cold storage; re-trigger pipeline; never delete source until processing is confirmed successful.

---

## 15. Wrap-Up Checklist (use this as your mental interview template)

- [ ] Clarified functional scope out loud
- [ ] Clarified non-functional priorities (availability vs consistency)
- [ ] Did rough capacity math
- [ ] Defined a minimal API
- [ ] Drew high-level architecture with clear data flow
- [ ] Picked 2–3 components to go deep on (usually: upload/transcode pipeline + CDN/playback)
- [ ] Named specific technologies (Kafka, HLS/DASH, Redis, Elasticsearch) rather than staying abstract
- [ ] Discussed at least one consistency trade-off explicitly (view counts is the classic one)
- [ ] Mentioned failure handling / retries somewhere
- [ ] If time allows: security touchpoints (this is your edge)

---

## How to Practice This

1. Set a 45-minute timer, no notes, whiteboard/paper only, walk through this full flow out loud.
2. Then swap the product: redesign this same skeleton for Netflix, Instagram, or a URL shortener — the skeleton doesn't change, only the deep-dive details do.
3. Record yourself. The goal isn't memorizing YouTube's architecture — it's fluency in the *reasoning framework* in section 1.
