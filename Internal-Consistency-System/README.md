# Internal Consistency and Design Cases

Begin with the [Internal Consistency Design Review](<design review.md>) for invariants, transaction isolation, locks, optimistic concurrency, single writers, and replica coordination.

Then use two case studies to see where the consistency boundary lies:

- [Design Online Auction](design-online-auction.md) — preserve the highest-bid invariant under concurrent bids; compare database decisions with Redis and Kafka paths.
- [Design Dropbox / Google Drive](design-dropbox-google-drive.md) — coordinate file versions, chunked storage, device sync, conflicts, and deletion.
