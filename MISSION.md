# Mission: Understand SeaweedFS Distributed Object Store Architecture

## Why
I want to understand how distributed object stores work under the hood. SeaweedFS is a real, production-grade, open-source system that implements the S3 API on a novel distributed architecture. By understanding its design — how it distributes data and metadata for buckets, objects, users, and IAM — I build a transferable mental model for distributed storage systems.

## Success looks like
- I can draw and explain the four-layer architecture (Master → Volume Servers → Filer → S3 API) from memory
- I can trace the full lifecycle of an S3 PutObject and GetObject request through every component
- I understand how metadata (bucket config, IAM, object listings) is separated from data (volume blobs), and why that matters
- I can reason about how the system scales horizontally without a central metadata bottleneck

## Constraints
- Pure curiosity-driven; no production cluster to operate or project deadline
- Learning through codebase exploration and lessons, not running a live cluster
- Available time: short, focused sessions

## Out of scope
- Operating or deploying SeaweedFS clusters
- SeaweedFS FUSE mount, WebDAV, or Hadoop integration
- Erasure coding internals, replication configuration, cloud tiering
- Iceberg/S3 Tables catalog details (for now)