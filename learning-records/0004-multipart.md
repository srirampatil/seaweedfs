# Multipart upload lifecycle

The user understands the complete multipart upload lifecycle:

1. Initiate creates a SHA1(key)_UUID upload ID, cryptographically bound to the object key, and a staging directory at /buckets/<bucket>/.uploads/<uploadID>/.
2. UploadPart writes each part as a .part file in the staging directory using the same chunked upload pipeline as PutObject. Parts are independent and can be uploaded in any order. Re-uploads of the same part number create new files; the newest wins at completion.
3. CompleteMultipartUpload validates ETags, sorts parts by number, concatenates chunk references with new offsets (no data copying), writes the final object (respecting versioning state), computes the multipart ETag as MD5(concat(part-etags))-N, and cleans up the staging directory.
4. Abort recursively deletes the staging directory.
5. Completion is idempotent — retries return the existing result.
6. Lifecycle TTL is suppressed on parts (clock starts at completion, not upload).

**Implications:** Ready for versioning internals, SSE key management, or Iceberg catalog.