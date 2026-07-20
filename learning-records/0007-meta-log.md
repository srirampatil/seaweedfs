# The filer meta-log

The user understands the complete meta-log system:

1. The meta-log is stored as regular filer files under /topics/.system/log/YYYY-MM-DD/HH-MM.filerId — no separate storage subsystem, no reserved space, no automatic truncation.
2. LogBuffer: in-memory 8MB buffer with 1-minute time-based flushes and size-based flushes. Four previous sealed buffers retained for catch-up subscribers.
3. Wire format: [4-byte big-endian uint32][protobuf LogEntry] where data is a marshaled SubscribeMetadataResponse.
4. Every filer mutation generates a log entry unconditionally — no lifecycle dependency. Only SystemLogDir entries are excluded (recursion guard).
5. Two read modes: server-streamed gRPC (SubscribeMetadata) and client-direct volume server reads (LogFileChunkRef).
6. Two LogBuffers: LocalMetaLogBuffer persists to disk; MetaAggregator.MetaLogBuffer is memory-only, merges peer events.
7. Multi-filer: MetaAggregator subscribes to peers, does full tree traversal on first connect, replays events into local store, tracks per-peer offsets.
8. Multi-filer log read uses OrderedLogVisitor with min-heap priority queue merge across per-filer iterators.
9. Subscribers include S3 API server, lifecycle worker, filer sync, and FUSE mount.
10. Offset system: sequential counter within a LogBuffer, persisted across restarts via InitializeOffsetFromExistingData.