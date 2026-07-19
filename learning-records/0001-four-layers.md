# Four-layer architecture is understood at structural level

The user can identify the four layers of SeaweedFS's object store (Master → Volume Servers → Filer → S3 API) and describe each layer's responsibility. They understand:

1. The master is NOT a metadata bottleneck — it only tracks volumes, not files.
2. Volume servers store blobs in append-only 32 GB volumes with 16-byte in-memory metadata per blob (O(1) reads).
3. The filer provides a hierarchical namespace on top of the flat blob store, using a pluggable external metadata store. It stores chunk references, not data.
4. The S3 API server translates S3 REST calls to filer operations. It streams data directly from volume servers (bypassing the filer for data).

They can trace a PutObject and GetObject through all four layers.

**Implications:** Ready to zoom into a specific subsystem (bucket metadata flow, IAM mechanics, or the filer's chunk-chaining for large objects). The next lesson should deepen one of these areas rather than introduce new layers.
