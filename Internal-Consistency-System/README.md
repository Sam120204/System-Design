# Internal Consistency and Design Cases

Begin with the [Internal Consistency Design Review](<design review.md>) for invariants, transaction isolation, locks, optimistic concurrency, single writers, and replica coordination.

The case studies show where the consistency boundary lies in each system.

- [Design Online Auction](design-online-auction.md) — preserve the highest-bid invariant under concurrent bids; compare database decisions with Redis and Kafka paths.
- [Design Dropbox / Google Drive](design-dropbox-google-drive.md) — coordinate file versions, chunked storage, device sync, conflicts, and deletion.
- [Stripe: Design a Payment System](design-payment-stripe.md) — keep orders, payment attempts, and ledger entries consistent across retries, PSP callbacks, and reconciliation.
