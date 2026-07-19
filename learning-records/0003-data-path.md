# Data path: chunked upload and direct streaming

The user understands the complete PutObject → GetObject data path:

1. PutObject chunks the stream into 8 MB segments, each independently assigned a file ID by the master (via filer.AssignVolume) and uploaded directly to a volume server. Up to 4 chunks upload in parallel.
2. FileChunk is a pointer (Fid + Offset + Size) — not the data itself. Offset is the position in the logical file, not in the volume.
3. The filer entry stores the chunk list + extended attributes. The filer never touches object data.
4. GetObject streams data directly from volume servers via the shared ReaderCache, bypassing the filer entirely (~19ms saved). Concurrent reads of the same chunk share one in-flight download.
5. Chunk manifests enable very large files by nesting chunk lists inside blob content, resolved transparently on read.
6. SSE encryption/decryption happens at the S3 API layer — volume servers see only ciphertext.
7. Files ≤ 256 KB are stored inline in the filer entry, not chunked.

**Implications:** Ready to explore specific subsystems — versioning internals, multipart upload lifecycle, SSE key management, Iceberg catalog.