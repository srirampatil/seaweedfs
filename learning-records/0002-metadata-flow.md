# Bucket config flow and IAM credential distribution

Three paths populate the BucketConfigCache: lazy-load (first access → filer fetch), push-based metadata subscription (millisecond-latency cache refreshes from the filer), and write-through invalidation (modifications invalidate so next read re-fetches merged state). The subscription engine is the backbone — it keeps bucket configs and IAM credentials consistent across all S3 API servers without polling.

The user understands:
1. BucketConfig is derived from the bucket's filer entry's extended attributes — no separate "config DB".
2. IAM credentials live on the filer as JSON files under /etc/iam/identities/ and are loaded into in-memory indexes (accessKeyIdent, nameToIdentity) with O(1) lookup.
3. The metadata subscription pushes events to every S3 API server, keeping caches warm and consistent.
4. The negative cache prevents repeated 404s from hammering the filer.

**Implications:** Ready for the data path deep-dive: chunking, file ID assignment, streaming from volume servers, and the ReaderCache.