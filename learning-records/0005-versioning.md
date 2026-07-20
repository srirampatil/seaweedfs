# Versioning internals

The user understands the complete versioning system:

1. Versioning states: Enabled, Suspended, None — stored as extended attribute on the bucket entry, cached in BucketConfig.
2. Versioned objects live in .versions/ directory as v_&lt;versionId&gt; files. The .versions/ directory entry carries cached latest-version metadata (size, mtime, ETag, owner, delete-marker flag) for single-scan list efficiency.
3. Version IDs use inverted timestamp format (MaxInt64 - nanos) so newer versions sort first in lexicographic filer listings. Old format (raw timestamp) coexists for backward compatibility.
4. PutObject with versioning writes directly to .versions/v_&lt;id&gt;, never to the regular path. The prior latest is stamped with ExtNoncurrentSinceNsKey for lifecycle.
5. Delete with versioning creates a delete marker (zero-chunk version file with ExtDeleteMarkerKey=true) — no data is removed.
6. GetObject with versioning reads the latest-version pointer from .versions/ and fetches the version file. Self-healing rescans and repairs stale pointers.
7. Null versions (pre-versioning or suspended objects) live at the regular path, not under .versions/.
8. DeleteSpecificVersion uses repoint-before-delete pattern — pointer is updated before the blob is removed, so crashes never leave dangling pointers.
9. RECOMPUTE_LATEST mutation lets the filer derive the latest-version pointer from the directory listing, used in routed transactions.

**Implications:** Ready for lifecycle engine (how NoncurrentDays and ExpiredObjectDeleteMarker work), SSE key management, or Iceberg catalog.