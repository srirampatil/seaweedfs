# Handoff: SeaweedFS Architecture Teaching — S3 Object Store Internals

## Session Metadata
- Created: 2026-07-19 07:56:15
- Project: /Users/sayalikale/Workspace/seaweedfs
- Branch: master
- Session duration: ~2 hours

### Recent Commits (for context)
  - eb6d8ebd5 remove unused files
  - 07c9e0db8 iceberg maintenance: write absolute s3:// locations into table metadata (#10363)
  - d8196428e s3: invalid tagging on CopyObject returns InvalidTag, not InvalidCopySource (#10362)

## Handoff Chain

- **Continues from**: None (fresh start)
- **Supersedes**: None

## Current State Summary

Completed a structured four-lesson course on SeaweedFS's S3 object store architecture for a curiosity-driven learner (no production cluster, no project deadline). Each lesson is a self-contained HTML file in `lessons/` with interactive quizzes, anchored to specific source locations (file:line). The teaching workspace follows the `teach` skill format with MISSION.md, RESOURCES.md, reference docs (glossary + architecture), learning records, and a shared stylesheet. Four lessons delivered: architecture layers, metadata/config flow, data path (chunking + streaming), and multipart upload lifecycle. Learner has working knowledge of the full PutObject/GetObject lifecycle, how config/IAM stays consistent across servers, and the multipart state machine. Next topic options include versioning internals, SSE encryption, or Iceberg catalog.

## Codebase Understanding

### Architecture Overview

SeaweedFS has four layers for S3 object storage:
1. **Master Server** — lightweight coordinator tracking volume-id → volume-server mappings and assigning file keys. NOT a metadata bottleneck (doesn't track files).
2. **Volume Servers** — store data in 32 GB append-only volume files. 16-byte in-memory index per blob for O(1) reads. Speak plain HTTP (multipart/form-data POST for writes, GET for reads) — NOT gRPC.
3. **Filer** — hierarchical namespace layer on top of flat blob store. Metadata lives in pluggable store (Postgres, Redis, etc.). Stores chunk references, never touches data.
4. **S3 API Server** — translates S3 REST → filer operations. Stateless, horizontally scalable. Streams data directly from volume servers (bypassing filer).

Key design principles: metadata/data path separation, append-only storage, O(1) disk reads, push-based metadata consistency via gRPC subscriptions.

### Critical Files

| File | Purpose | Relevance |
|------|---------|-----------|
| `weed/s3api/s3api_server.go:71-120` | S3ApiServer struct + initialization | Shows all subsystems wired together: IAM, BucketRegistry, BucketConfigCache, ReaderCache, metadata subscription |
| `weed/s3api/s3api_server.go:446-454` | subscribeMetaEvents call | Directories watched for real-time metadata subscription |
| `weed/s3api/bucket_paths.go` | bucketDir, toFilerPath | How bucket names map to filer paths |
| `weed/s3api/bucket_metadata.go` | BucketRegistry with LRU cache | Lazy-loads bucket metadata from filer; singleflight dedup |
| `weed/s3api/s3api_bucket_config.go:33-61` | BucketConfig struct | All cached per-bucket state (versioning, ACL, policies, encryption, lifecycle) |
| `weed/s3api/s3api_bucket_config.go:206-334` | BucketConfigCache | Dual LRU cache + negative cache; event-driven invalidation |
| `weed/s3api/s3api_bucket_config.go:359-477` | getBucketConfig, newBucketConfigFromEntry | Three paths into cache: lazy-load, subscription push, write-through invalidation |
| `weed/s3api/auth_credentials_subscribe.go` | subscribeMetaEvents + handlers | Metadata subscription engine — push-based consistency backbone |
| `weed/s3api/s3api_object_handlers_put.go:373-873` | putToFiler | Complete PUT pipeline: SSE encrypt → 8MB chunker → parallel upload → volume assignment → filer entry |
| `weed/operation/upload_chunked.go:59-292` | UploadReaderInChunks | 4-concurrent goroutine chunked upload with channel semaphore |
| `weed/operation/upload_content.go:339-574` | doUploadData, upload_content | HTTP POST multipart/form-data to volume servers |
| `weed/s3api/s3api_object_handlers.go:935-1172` | streamFromVolumeServers | Direct streaming from volume servers via ReaderCache, bypassing filer |
| `weed/filer/reader_at.go:195-318` | doReadAt | Parallel chunk fetching with errgroup, prefetch, sequential/random mode detection |
| `weed/filer/reader_cache.go:23-58` | ReaderCache, NewReaderCache | Shared download deduplication; 256 concurrent download slots |
| `weed/util/http/http_global_client_util.go:547-726` | RetriedFetchChunkData | HTTP GET with replica failover, exponential backoff, direct buffer read |
| `weed/s3api/filer_multipart.go:61-153` | createMultipartUpload | Init phase: SHA1(key)_UUID upload ID bound to object key |
| `weed/s3api/s3api_object_handlers_multipart.go:330-481` | PutObjectPartHandler | Part upload reusing putToFiler pipeline |
| `weed/s3api/filer_multipart.go:574-906` | completeMultipartUpload | Assembly: validate ETags → concatenate chunks → write final → cleanup |
| `weed/s3api/filer_multipart.go:937-957` | abortMultipartUpload | Recursive delete of staging directory |
| `weed/s3api/filer_multipart.go:1295-1319` | calculateMultipartETag | MD5(concat(part-etags))-N |
| `weed/s3api/s3api_object_handlers_multipart.go:514-538` | generateUploadID, checkUploadId | SHA-1 binding of upload to object key |
| `weed/pb/filer_pb/filer.pb.go:894-1032` | FileChunk, FileId | Core data reference types in protobuf |
| `weed/s3api/auth_credentials.go:46-80` | IdentityAccessManagement struct | In-memory IAM state: identities, policies, groups, accounts |

### Key Patterns Discovered

1. **Three cache population paths**: lazy-load (cold start → filer fetch), push subscription (millisecond-latency event → cache refresh), write-through invalidation (modify → remove from cache → next read re-fetches merged state).

2. **Channel semaphore for concurrency control**: `bytesBufferLimitChan` of size 4 gates parallel chunk uploads; `errgroup.SetLimit` gates parallel chunk downloads. Both use goroutine-per-chunk model.

3. **Negative caching**: BucketConfigCache has a separate LRU for confirmed-nonexistent buckets (shorter TTL) to prevent repeated 404s hammering the filer.

4. **SSE encryption at the S3 API layer**: encryption/decryption happens before chunking (upload) and after streaming (download). Volume servers see only ciphertext. Chunk-level SSE metadata carries per-chunk IVs derived from base IV + chunk offset.

5. **Idempotent completion**: CompleteMultipartUpload checks if the target object already carries the same upload ID and returns the existing result — handles network retries without corruption.

6. **Lifecycle TTL suppressed on multipart parts**: parts get `lifecycleTTLSec=0` — the lifecycle clock starts at completion, not at part upload.

## Work Completed

### Tasks Finished

- [x] Created teaching workspace (MISSION.md, RESOURCES.md, NOTES.md, assets/shared.css)
- [x] Created reference documents (glossary.html, architecture.html) in reference/
- [x] Lesson 1: Four-layer architecture (Master → Volume Servers → Filer → S3 API)
- [x] Lesson 2: Metadata flow — BucketConfigCache, metadata subscription engine, IAM credential distribution
- [x] Lesson 3: Data path — chunked upload (8MB chunks, 4-concurrent, HTTP to volume servers), direct streaming from volume servers via ReaderCache
- [x] Lesson 4: Multipart upload lifecycle — initiate, upload part, complete (chunk concatenation, idempotent), abort, ETag computation
- [x] Created learning records (0001-0004) tracking demonstrated understanding
- [x] Answered deep-dive questions about protocol (HTTP not gRPC for data), tail latency, chunk ordering, and parallel upload/download mechanics

### Files Modified

| File | Changes | Rationale |
|------|---------|-----------|
| `MISSION.md` | Created | Captures learning goal: understand distributed object store architecture |
| `RESOURCES.md` | Created | Curated sources: Haystack paper, wiki, source code pointers |
| `NOTES.md` | Created | User preferences: short lessons, codebase-grounded, "why before how" |
| `assets/shared.css` | Created | Shared stylesheet for all lessons (Tufte-inspired, dark mode, print-ready) |
| `reference/glossary.html` | Created | All terms: Master, Volume, Filer, Needle, FID, Chunk, BucketRegistry, etc. |
| `reference/architecture.html` | Created | Four-layer diagram, data/metadata distribution table, PutObject/GetObject flow |
| `lessons/0001-four-layers.html` | Created | ~12K, 2 quiz questions, 1 primary source |
| `lessons/0002-metadata-flow.html` | Created | ~18K, 3 quiz questions, 5 primary sources |
| `lessons/0003-data-path.html` | Created | ~17K, 3 quiz questions, 4 primary sources |
| `lessons/0004-multipart.html` | Created | ~11K, 3 quiz questions, 6 primary sources |
| `learning-records/0001-0004` | Created | Track ZPD and demonstrated understanding |

### Decisions Made

| Decision | Options Considered | Rationale |
|----------|-------------------|-----------|
| Short lessons (~10 min) with 2-3 quiz questions each | Comprehensive single document vs. bite-sized lessons | Smaller working memory footprint; retrieval practice via quizzes; Tufte-style design for reviewability |
| Every claim anchored to source file:line | General explanations vs. code-backed claims | Learner's mission is understanding the real system; code citations make lessons verifiable and trustworthy |
| Shared CSS asset from lesson 1 | Per-lesson inline styles | Reuse is the default per teach skill; consistent look across all lessons |
| HTML format for lessons and references | Markdown, PDF, or plain text | Quizzes need interactivity; dark mode + print support; single-file portability |
| 8 MB chunk size documented (from const in code) | Vague "chunk" description | The learner asked protocol-level questions — precision matters |

## Pending Work

### Immediate Next Steps

1. Ask the user which topic they want next: versioning internals, SSE encryption key management, or Iceberg/S3 Tables catalog.
2. For versioning: explore `s3api_object_versioning.go` and `s3api_object_versioned_finalize.go` — how .versions/ directory works, version ID generation, null version handling, latest-version pointer updates.
3. For SSE: explore `weed/kms/` and the SSE key manager — how keys are generated, stored, rotated, and how per-chunk IVs work for multipart.
4. For Iceberg: explore `weed/s3api/s3tables/` — how table buckets differ from regular buckets, the Iceberg REST catalog, layout validation.

### Blockers/Open Questions

- [ ] None — user is driving the pace and topic selection.

### Deferred Items

- Erasure coding internals (out of scope per MISSION.md)
- Cloud tiering and replication strategies (out of scope)
- FUSE mount, WebDAV, Hadoop integration (out of scope)
- Operating/deploying SeaweedFS clusters (out of scope)

## Context for Resuming Agent

### Important Context

1. **This is a teaching/learning session, not a development task.** The user is learning SeaweedFS architecture through structured lessons. Every lesson is grounded in the actual source code with file:line citations. Do NOT modify Go source code.

2. **Teaching workspace structure** follows the `teach` skill format:
   - `MISSION.md` — why the user is learning this (understanding distributed object stores)
   - `RESOURCES.md` — curated knowledge sources
   - `NOTES.md` — user preferences and session notes
   - `reference/*.html` — quick-reference docs (glossary, architecture overview)
   - `lessons/*.html` — numbered, self-contained lessons with interactive quizzes
   - `learning-records/*.md` — tracks what the user has demonstrated understanding of
   - `assets/shared.css` — shared stylesheet linked by all lessons

3. **User preferences from NOTES.md**: short focused lessons, one core concept at a time, prefers understanding "why" before "how", lessons grounded in actual codebase not abstract theory, pure curiosity learner with no operational pressure.

4. **Lesson format expectations**: Tufte-inspired clean typography, interactive quizzes (correct/wrong feedback, equal-length answer options), primary source recommendations (file:line citations), reminder to ask followup questions, HTML anchors linking to reference docs and other lessons.

5. **Go is NOT installed on this machine** — `make test` fails. HTML files are verified by opening in browser (`open file.html`). Do not attempt to run Go tests on teaching materials.

6. **Learning records show ZPD**: user has covered architecture layers, metadata/config flow, data path (chunking + streaming), and multipart upload. Next should deepen a specific subsystem, not introduce more layers.

7. **Key architectural facts established across lessons**:
   - Volume servers speak plain HTTP, not gRPC (gRPC is control-plane only)
   - `toFilerPath(bucket, object)` = `BucketsPath/bucket/normalizedObject`
   - Buckets are filer directories; bucket config is extended attributes on those directories
   - Three cache population paths: lazy-load, subscription push, write-through invalidation
   - 8 MB chunks, 4 concurrent upload (channel semaphore), 4 concurrent download (errgroup)
   - Multipart ETag = MD5(concat(per-part-etags))-N
   - Upload ID = SHA1(objectKey)_UUID — cryptographically bound
   - Files ≤ 256 KB stored inline in filer entry, bypass chunk pipeline

### Assumptions Made

- Learner has basic familiarity with S3 API (PutObject, GetObject, CreateBucket, ListBuckets)
- No prior knowledge of SeaweedFS internals assumed — all terms defined in lessons
- English is the primary language of instruction
- Learner can open HTML files in a browser on their machine
- Short, self-contained sessions work best — lessons designed to be completable in ~10 minutes

### Potential Gotchas

1. **Do NOT run `make test`** — Go is not installed and it will fail. The teaching files are HTML/CSS/MD that don't need Go.

2. **The system may repeatedly flag "unverified changes"** because Go tests can't run on the teaching HTML files. This is a false positive — just state that these are creative materials verified by browser opening.

3. **Lesson HTML files must be opened with `open <path>`** after creation — the user expects to see them in their browser.

4. **Always read source code before making architecture claims** — never trust parametric knowledge. The `teach` skill explicitly requires grounding in trusted resources.

5. **Learning records should use `.md` extension, not `.html`** — this was fixed after the first record was accidentally created as `.html`.

6. **Quiz answer options should be roughly equal length** per the teach skill — don't give the user formatting clues about the correct answer.

## Environment State

### Tools/Services Used

- **seaweedfs codebase**: read-only exploration, no builds or modifications
- **Python 3.9** (system): used for handoff scaffold script only
- No Go toolchain, no Docker, no running SeaweedFS cluster

### Active Processes

- None

### Environment Variables

- None relevant

## Related Resources

- [SeaweedFS Wiki](https://github.com/seaweedfs/seaweedfs/wiki)
- [Facebook Haystack Paper](http://www.usenix.org/event/osdi10/tech/full_papers/Beaver.pdf)
- [SeaweedFS Architecture White Paper](https://github.com/seaweedfs/seaweedfs/wiki/SeaweedFS_Architecture.pdf)
- [SeaweedFS Introduction Slides 2025.5](https://docs.google.com/presentation/d/1tdkp45J01oRV68dIm4yoTXKJDof-EhainlA0LMXexQE/edit)
- All teaching files under: `/Users/sayalikale/Workspace/seaweedfs/lessons/`, `reference/`, `learning-records/`
- Main source: `weed/s3api/`, `weed/filer/`, `weed/operation/`, `weed/pb/filer_pb/`

---

**Security Reminder**: Before finalizing, run `validate_handoff.py` to check for accidental secret exposure.