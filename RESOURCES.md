# SeaweedFS Architecture Resources

## Knowledge

- [SeaweedFS White Paper (PDF)](https://github.com/seaweedfs/seaweedfs/wiki/SeaweedFS_Architecture.pdf)
  Primary architecture document. Use for: high-level design decisions, volume management, the file key structure.

- [SeaweedFS Introduction Slides (2025.5)](https://docs.google.com/presentation/d/1tdkp45J01oRV68dIm4yoTXKJDof-EhainlA0LMXexQE/edit?usp=sharing)
  Visual walkthrough of all components. Use for: component diagrams, feature overview.

- [Facebook Haystack Design Paper (OSDI 2010)](http://www.usenix.org/event/osdi10/tech/full_papers/Beaver.pdf)
  The original inspiration for SeaweedFS's volume model. Use for: understanding why append-only volumes beat POSIX filesystems for small blobs.

- [f4: Facebook's Warm BLOB Storage (OSDI 2014)](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-muralidhar.pdf)
  Inspiration for SeaweedFS's erasure coding on warm data. Use for: understanding the hot/warm storage split.

- [SeaweedFS Wiki: Amazon S3 API](https://github.com/seaweedfs/seaweedfs/wiki/Amazon-S3-API)
  S3 API compatibility docs. Use for: understanding which S3 operations are supported and how they map to filer operations.

- Source code (this repo): `weed/s3api/s3api_server.go`, `weed/s3api/s3api_object_handlers.go`, `weed/s3api/s3api_bucket_handlers.go`, `weed/s3api/bucket_metadata.go`, `weed/s3api/bucket_paths.go`
  The ground truth. Use for: verifying any claim about how a specific feature works.

## Wisdom (Communities)

- [SeaweedFS Slack](https://join.slack.com/t/seaweedfs/shared_invite/enQtMzI4MTMwMjU2MzA3LTEyYzZmZWYzOGQ3MDJlZWMzYmI0OTE4OTJiZjJjODBmMzUxNmYwODg0YjY3MTNlMjBmZDQ1NzQ5NDJhZWI2ZmY)
  Active community with developers. Use for: architecture questions, design rationale.

- [SeaweedFS Subreddit](https://www.reddit.com/r/SeaweedFS/)
  Use for: community experience, deployment stories.

- [SeaweedFS GitHub Issues](https://github.com/seaweedfs/seaweedfs/issues)
  Use for: understanding edge cases, design discussions, and known limitations.

## Gaps
- No textbook or structured course on distributed blob store architecture yet. We build our own through these lessons.