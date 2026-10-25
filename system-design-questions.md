# System Design Interview Questions

Grouped by pattern. Tick off each one as you practice it.

## 1. Storage and Data-Heavy Systems

- [ ] Distributed cache (done) – eviction, consistent hashing, replication, stampede
- [ ] URL shortener (TinyURL) – ID generation (base62, counter vs hash), read-heavy scaling, caching
- [ ] Pastebin – blob storage, expiry, object store + metadata DB split
- [ ] Key-value store (DynamoDB-like) – partitioning, replication, quorum reads/writes, vector clocks
- [ ] Distributed file storage (S3/GFS) – chunking, metadata service, replication, durability
- [ ] Dropbox / Google Drive – chunk-level sync, dedup, conflict resolution, notifications
- [ ] Distributed unique ID generator – Snowflake-style IDs, clock skew, ordering
- [ ] Time-series DB / metrics system – write-heavy ingestion, downsampling, retention

## 2. Real-Time and Messaging

- [ ] Chat system (WhatsApp) – WebSockets, message ordering, delivery receipts, offline storage
- [ ] Notification service – fan-out, priority queues, retries, third-party providers (FCM/APNs)
- [ ] Live comments / live streaming – pub/sub, fan-out at scale, connection management
- [ ] Presence system (online/offline) – heartbeats, TTL, avoiding write storms
- [ ] Video conferencing (Zoom) – WebRTC, SFU vs MCU, adaptive bitrate

## 3. Feeds and Social

- [ ] News feed (Twitter/Facebook) – fan-out on write vs read, hybrid for celebrities
- [ ] Instagram / photo sharing – media pipeline, CDN, feed ranking
- [ ] Typeahead / autocomplete – trie, prefix indexing, top-K, caching
- [ ] Trending topics / Top-K – streaming counts, Count-Min Sketch, sliding windows
- [ ] Recommendation system – offline training vs online serving, feature store

## 4. Infrastructure and Platform

- [ ] Rate limiter – token bucket, sliding window, distributed counters (Redis)
- [ ] API gateway / load balancer – L4 vs L7, health checks, routing algorithms
- [ ] Distributed message queue (Kafka) – partitions, offsets, consumer groups, delivery guarantees
- [ ] Job scheduler / cron at scale – leader election, idempotency, retries, delayed queues
- [ ] Distributed lock service – leases, fencing tokens, Redlock debate, ZooKeeper/etcd
- [ ] Web crawler – URL frontier, politeness, dedup, BFS at scale
- [ ] Log aggregation / monitoring – agents, buffering, indexing, alerting

## 5. Transactional and Business Systems

- [ ] Payment system – idempotency keys, exactly-once effects, reconciliation, ledger design
- [ ] E-commerce / flash sale – inventory consistency, overselling, queueing, rate limiting
- [ ] Ticket booking (BookMyShow) – seat locking with TTL, concurrency control
- [ ] Ride sharing (Uber/Ola) – geospatial indexing (geohash, quadtree), matching, ETA
- [ ] Food delivery (Swiggy/Zomato) – order state machine, dispatch, real-time tracking
- [ ] Hotel / flight booking – search vs booking split, availability caching, double-booking
- [ ] Wallet / UPI-like system – double-entry ledger, atomicity across services, sagas

## 6. Search and Media

- [ ] Search engine / Elasticsearch – inverted index, sharding, ranking
- [ ] YouTube / Netflix – transcoding pipeline, CDN, adaptive streaming
- [ ] Google Maps – graph representation, routing algorithms, tile serving
- [ ] Spotify – audio streaming, playlists, offline sync

## 7. Recurring Patterns

1. **Fan-out:** write vs read (feeds, notifications)
2. **Partitioning:** consistent hashing, range vs hash sharding
3. **Replication and consistency:** quorum, leader-follower, CAP/PACELC
4. **Idempotency and retries:** payments, queues, APIs
5. **Async processing:** queues, workers, backpressure
6. **Caching layers:** CDN, local, distributed
7. **Concurrency control:** locks, optimistic versioning, leases
8. **Probabilistic structures:** Bloom filter, Count-Min Sketch, HyperLogLog

## 8. Suggested Prep Order

1. **Foundations:** URL shortener, rate limiter, key-value store
2. **Messaging:** notification service, chat system, Kafka-style queue
3. **Feeds and search:** news feed, typeahead
4. **Transactional:** payment system, ticket booking, flash sale
5. **Hard ones:** distributed lock, ride sharing, Dropbox

## 9. Framework to Reuse Every Time

1. Requirements (functional and non-functional) and scale estimates
2. API design
3. Data model and storage choice
4. High-level architecture
5. Deep dive on the 1-2 hardest parts
6. Bottlenecks, failure modes, trade-offs
