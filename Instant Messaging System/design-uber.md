# Design Uber: Real-Time Ride Matching

A rider enters a pickup point and a destination, gets a fare estimate, then requests a ride. The system finds nearby available drivers and offers the ride to one of them. After a driver accepts, both apps see the assignment and follow the trip through pickup and drop-off.

This interview design uses the supplied sketch and its scale assumptions. [Open the full-resolution sketch](images/design-uber-sketch.png)

## 1. Requirements and scale

- **Rider.** Get a fare estimate, request a ride from that estimate, and see the match result and trip progress.
- **Driver.** Publish location while online, accept or decline an offer, navigate to pickup and drop-off, and update trip milestones.
- **Latency.** Aim to assign a driver or return a clear outcome within about one minute. That budget includes queue time, candidate search, and time for drivers to respond.
- **Consistency.** A ride has at most one assigned driver, and a driver has at most one active ride. The baseline also allows at most one live offer per ride and per driver. Driver coordinates can be slightly stale when searching for candidates.
- **Availability.** Keep ride status and GPS ingestion usable if matching is overloaded. If the assignment store is unavailable, pause new assignments until the system can reserve drivers safely.

Focus on on-demand rides with one vehicle class. Scheduled rides, ratings, payments, and fare optimization can be discussed separately if the interview calls for them.

| Workload | Interview assumption and impact |
| --- | --- |
| Driver location updates | 6 million registered drivers, 3 million online at a busy moment, and one update every 5 seconds give about **600,000 updates per second globally**. This is before retries, index updates, and replication. |
| Matching load | Size it from peak ride requests per metro and searches or offers per request. For example, 1,000 ride requests per second with up to 3 candidate searches each can generate up to 3,000 searches per second in that metro. |
| Hotspots | A stadium or airport can concentrate riders and drivers in a few neighboring cells. Global averages hide local queue pressure, index load, and contention for the same drivers. |

## 2. Architecture and request flow

![Architecture crop showing rider and driver apps, API gateway, ride request queue, matching workers, DynamoDB, Redis, mapping, and push notifications](images/design-uber-architecture.png)

1. **Quote a fare.** The API gateway authenticates and routes requests and applies rate limits. The ride service calls a mapping provider for route distance and travel time, then returns a quote with an ID and expiry. Keep the inputs and quoted amount with the quote so ride creation can validate what the rider accepted.
2. **Accept the ride request.** The ride service validates the quote, then writes the ride in `searching` and a dispatch command in one DynamoDB transaction. Return `202 Accepted` and the `ride_id` after commit. An outbox relay sends the command to the ride request queue. Workers tolerate repeated delivery and use the stored ride state to decide what work remains.
3. **Ingest driver locations separately.** The driver app sends location and freshness information to the location service, which updates a regional Redis geospatial index. Keep assignment and availability in DynamoDB so location writes do not compete with assignment transactions on the same records.
4. **Match and notify.** A matching worker queries Redis for nearby candidates, ranks a shortlist by estimated pickup time, and reserves a driver in DynamoDB before sending an offer. The notification service uses APNs or FCM to alert the driver. Acceptance commits the assignment, and the service publishes the new ride state to both apps.
5. **Track the trip.** The driver app reports pickup and drop-off. The ride service checks that the driver owns the ride and that each transition is valid. The rider sees near-real-time status and driver position over an authenticated live connection while the app is active. On reconnect, the app fetches a fresh ride snapshot. A push notification can prompt that fetch when the app is in the background.

The sketch shows the gateway feeding a ride request queue directly. Writing the ride and its dispatch command first adds an immediately readable ride record and lets the relay recover if the API process crashes after commit. [Stripe case study on the transactional outbox](<../Internal-Consistency-System/design-payment-stripe.md#4-qa--at-high-qps-how-do-you-choose-between-outbox-and-kafka-first>)

### Core entities

| Entity | Example fields and role |
| --- | --- |
| `Rider` | `rider_id` and account metadata |
| `Ride` | `ride_id`, `rider_id`, `quote_id`, pickup, destination, state, `current_offer_id`, optional `driver_id`, and `match_deadline` |
| `Driver` | `driver_id`, dispatch state (`offline`, `available`, `offered`, `on_trip`), `current_offer_id`, and optional `active_ride_id` |
| `Offer` | `offer_id`, `ride_id`, `driver_id`, state, and `expires_at`. The offer ID lets the server reject a response to an earlier offer. |
| `DriverLocation` | `driver_id`, latitude, longitude, update sequence, and timestamp. Redis uses this for proximity search and trip tracking. |

### Example APIs

| Endpoint | Contract |
| --- | --- |
| `POST /fare-estimates` | Accepts pickup and destination. Returns `quote_id`, estimate, currency, and expiry. |
| `POST /rides` | Accepts `quote_id` and an idempotency key scoped to the rider. Returns `202 Accepted` and a `ride_id`. Repeating the same submission returns the same ride. |
| `GET /rides/{ride_id}` | Returns the current ride state and the assignment when available to the authenticated participant. |
| `POST /drivers/me/location` | Accepts coordinates, a timestamp, and an update sequence for the authenticated driver. |
| `GET /drivers/me/current-offer` | Returns the authenticated driver's current unexpired offer, if one exists. |
| `PATCH /offers/{offer_id}` | Accepts `accept` or `decline` from the driver who received the offer. Returns the committed outcome. |
| `PATCH /rides/{ride_id}/status` | Records pickup or drop-off from the assigned driver and checks the current ride state. |

## 3. Finding nearby drivers

![Spatial-index crop with a quadtree that subdivides a dense area and a geohash grid highlighting the children of cell 2](images/design-uber-spatial-indexes.png)

| Approach | Strengths | Trade-offs |
| --- | --- | --- |
| **Geohash** | Encodes a point as a string whose prefixes represent progressively smaller rectangular cells. In the sketch's simplified grid, cell `2` contains `20`, `21`, `22`, and `23`. Prefixes work with ordinary database indexes and give you predictable keys for caching and partitioning. With a bucket-based index, a point that moves within the same bucket can keep its membership while updating its coordinates. | Nearby points across a cell boundary can have different prefixes, so cover all cells that intersect the search area and filter candidates by exact distance. Coarse cells can contain too many candidates downtown. Finer cells reduce candidates per cell but can require more lookups per search. Cell dimensions on the ground vary with latitude. |
| **Quadtree** | Recursively splits a region into four quadrants. In the sketch, dense quadrants split when they exceed `k = 5` points. A radius or bounding-box query visits overlapping branches and skips the rest. Dense regions get smaller leaves and sparse regions can keep larger ones. | The tree uses extra memory and needs a build and recovery strategy. Frequent movement requires updating points and moving them between leaves when they cross boundaries. Changes in density can trigger splits or merges. A distributed tree also needs to route searches across partitions. |

### Which index fits Yelp-style search and Uber-style dispatch?

| What changes | How it affects the choice |
| --- | --- |
| **Location update rate** | Restaurant coordinates rarely change, so a Yelp-style service can reuse an in-memory quadtree snapshot across many searches. With drivers reporting GPS every few seconds, Redis GEO is a practical baseline for frequent position updates. A database geohash index also suits restaurant search when location edits need to become searchable as soon as they commit. A quadtree can support incremental updates too. The hourly rebuild is a design choice in the Yelp case study. |
| **Density of indexed points** | If a fixed cell contains thousands of restaurants downtown and only a few in the suburbs, a quadtree can subdivide the downtown cell while keeping larger leaves elsewhere. Finer geohash prefixes in dense regions can address the same problem if the query covers all relevant cells. Measure how many candidates a typical search inspects before choosing. |
| **Search traffic** | More riders or restaurant searchers increase read load. If many users search the same area, cache reusable candidate sets for stable places or replicate the busy read partition. If a single geohash cell gets hot, splitting it helps when traffic can spread across the resulting partitions. With live drivers, keep any cache short-lived and verify availability when reserving a driver. |

For example, a 500-metre restaurant search might intersect a single coarse cell with 2,000 listings, only 30 of which are within the radius. A quadtree can skip leaves elsewhere in that cell. Finer geohash prefixes can also reduce the candidate set at the cost of more prefix lookups. If all 2,000 listings are inside the requested area, either index still has to account for them when returning a complete result set.

Start with the index your storage layer already supports if it meets the freshness and latency targets. Evaluate an adaptive tree if benchmarks show that coarse cells inspect too many candidates and finer cells require too many lookups. The Yelp case study works through both a snapshot-backed quadtree and a database-managed geohash index. [Yelp nearby business search case study](../data-retrieval-caching-and-indexing/design-yelp.md)

**Baseline.** Use regional Redis geospatial sorted sets with `GEOADD` to update positions and `GEOSEARCH` to get a shortlist by radius. A small metro can start with one key. For hot metros, split the index into geographic keys, query every key intersecting the search area, and deduplicate driver IDs. If a driver changes cells, old and new entries may coexist briefly. Use the latest version to skip old entries and clean them up asynchronously. [Redis GEOSEARCH](https://redis.io/docs/latest/commands/geosearch/)

**Freshness and ranking.** Record a server-side last-seen time and reject out-of-order updates within a driver's current online session using the sequence number. Exclude old positions at query time and remove stale members with a cleanup job. Use straight-line distance to shortlist candidates, then ask the routing provider for pickup ETAs for the best few. A driver across a river may be geographically close and still take longer to reach the pickup. Verify that the selected driver is still available in the assignment transaction.

## 4. Matching without double-booking

![Matching crop showing the one-offer-per-ride and one-offer-per-driver rules and a ten-second sequential offer loop](images/design-uber-matching.png)

The sketch offers a ride to one driver at a time and waits about ten seconds for a response. Store that wait as an offer deadline and schedule a follow-up so other rides can keep using the worker. An accepted offer advances the ride from `searching` to `matched`. Pickup and drop-off then advance it to `in_progress` and `completed`.

| Step | Conditional write in DynamoDB |
| --- | --- |
| **Reserve and offer** | In one transaction, require a `searching` ride within its match deadline, no `current_offer_id`, and an `available` driver. Create the offer with an expiry capped by the ride deadline, set the same `current_offer_id` on the ride and driver, set the driver to `offered`, and persist an outbox event for notification. A competing worker can try another candidate if the driver was already claimed. |
| **Accept** | Verify the responding driver and condition the transaction on an open offer within its deadline, a `searching` ride, an `offered` driver, and both matching `current_offer_id` values. Mark the offer accepted, set the ride to `matched`, and set the driver to `on_trip` with `active_ride_id`. Clear the offer pointers and persist the match event in the same transaction. Retrying the same accepted offer returns its recorded outcome. |
| **Decline or expire** | Close the offer and clear the reservation only if both records still reference that offer. Persist the next dispatch command if the ride is still eligible and time remains. |
| **Finish or cancel** | Before pickup, cancellation closes an outstanding offer or releases the assigned driver in the same transaction. At drop-off, mark the ride complete and clear the driver's active assignment only if it still refers to that ride. |

### How DynamoDB acts as a per-driver distributed lock

A Redis-based design could acquire a short lease with `SET lock:driver:D offer-A NX PX 10000`. `NX` lets one worker create the lock while the key is absent, `PX` sets its lifetime, and `offer-A` identifies the owner. If the worker releases the lease early, it must check the owner and delete the key atomically so a delayed release cannot delete a newer lease. [Redis distributed locks](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)

In this design, **the conditional update to the driver's DynamoDB record serves the same role**. All matching workers reserve driver D through the same authoritative record. The transaction checks that D is `available`, changes the state to `offered`, and records `offer-A` as the reservation owner. It also reserves the ride and writes the notification event atomically. Even if many workers read `available` at the same time, only one can commit that transition. Workers can still reserve different drivers in parallel. Redis supplies nearby candidates, and DynamoDB decides who can receive an offer. [DynamoDB transactions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html)

An offer stops being eligible for acceptance at its `expires_at` deadline. A scheduled job closes the expired offer and clears its reservation only if the ride and driver still point to that offer ID. If D moves on to offer B, a late accept or a delayed cleanup for A cannot change B. DynamoDB TTL can remove closed offer records later. [DynamoDB TTL](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)

### What happens when a Taylor Swift concert lets out?

1. Hundreds of riders request a pickup near the venue. Several matching workers get the same nearby driver D from Redis, possibly before the location search reflects any change in availability.
2. One worker reserves D and queues offer A. The other workers' reservations for D fail or conflict, so they try other candidates within the ride deadline. Only the committed transaction queues an offer for D during that reservation.
3. If D accepts, the transaction moves D to `on_trip`, which keeps D out of subsequent reservations until the trip ends. If D declines or the offer expires, the transaction releases the reservation and the ride can move on to its next candidate.

The reservation limits D to **one live offer at a time**. A short per-driver cooldown after a decline or timeout can also limit back-to-back offers during the surge. Store a `next_offer_at` timestamp when releasing the reservation and check it in the next reservation transaction. Push delivery can retry or arrive late, so include the offer ID and expiry, deduplicate by ID in the app, and fetch the current offer before displaying an actionable prompt.

## 5. Handling bursts and delays

- **Partition work by market.** Route ride requests by pickup metro and scale matching workers against queue age and expected wait time. Each driver's assignment state needs one write owner across adjacent markets. Nearby index queries can span cells or market boundaries while reservations still target the same authoritative driver record.
- **Budget the search and offer time together.** Four sequential ten-second offer windows consume about 40 seconds before queueing and spatial queries. Push delivery also uses part of each offer window. Pass the original match deadline through retries. Once it expires, condition the `no_driver_found` transition on the ride still being `searching` and release any open offer in the same transaction. Widen the search when early attempts find few eligible drivers and time remains.
- **Protect dispatch from GPS spikes.** Scale location ingestion separately. Under pressure, coalesce pending GPS updates per driver to the newest position. Keep ride creation, acceptance, and trip milestones on their durable paths. Monitor match latency, queue age, rejected reservations, and stale location rates by metro.
- **Recover from delayed delivery.** Persist each offer before sending push and make notifications retryable by offer ID. The driver app can fetch its current offer on reconnect. If a worker crashes after a state transition, the outbox can still publish the committed result. A periodic sweep can find overdue offers and rides whose follow-up work was delayed.
