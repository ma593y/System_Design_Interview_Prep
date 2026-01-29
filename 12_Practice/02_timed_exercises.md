# Timed Practice Exercises

Structured 45-minute practice sessions that simulate real interview conditions. Each session includes specific timing guidance and checkpoints.

---

## How to Use These Exercises

### Before You Start
1. **Set up your environment** - Whiteboard, paper, or digital drawing tool
2. **Remove distractions** - No phones, close unnecessary tabs
3. **Set a timer** - Use the phase timings strictly
4. **Record yourself** (optional) - Review later to improve communication

### During the Exercise
- Speak out loud as if explaining to an interviewer
- Draw diagrams as you go
- Note assumptions you're making
- Flag areas you'd explore with more time

### After Each Exercise
- Review against the provided checklist
- Note what went well and what to improve
- Compare your design to published solutions

---

## Session 1: URL Shortener (Beginner)

**Total Time: 45 minutes**

### Phase 1: Requirements & Scope (0:00 - 5:00)

**Your task:** Define what you're building

Questions to answer out loud:
- What operations does the system support?
- What's the expected scale? (URLs created per day, reads per second)
- Do short URLs expire?
- Are custom short URLs allowed?
- Do we need analytics?

**Checkpoint at 5:00:**
- [ ] Listed core features (shorten, redirect)
- [ ] Stated scale assumptions (e.g., 100M URLs/day, 10:1 read:write)
- [ ] Identified nice-to-haves vs must-haves

### Phase 2: Capacity Estimation (5:00 - 10:00)

**Your task:** Calculate storage and bandwidth needs

Work through:
- Storage per URL (short code + long URL + metadata)
- Total storage for 5 years
- Read QPS and write QPS
- Bandwidth requirements

**Checkpoint at 10:00:**
- [ ] Calculated storage estimate
- [ ] Estimated read/write QPS
- [ ] Numbers are reasonable (sanity check)

### Phase 3: High-Level Design (10:00 - 20:00)

**Your task:** Draw the system architecture

Include:
- Client interaction flow
- API endpoints
- Core services
- Database choice
- Caching layer

**Checkpoint at 20:00:**
- [ ] Diagram shows complete flow from client to database
- [ ] API endpoints defined (POST /shorten, GET /{shortCode})
- [ ] Database and cache shown
- [ ] Can explain why you chose each component

### Phase 4: Deep Dive - ID Generation (20:00 - 30:00)

**Your task:** Design the short code generation system

Consider:
- How to generate unique short codes
- Collision handling
- Distributed generation (multiple servers)
- Custom short codes

**Checkpoint at 30:00:**
- [ ] Explained at least 2 approaches (e.g., hash-based, counter-based)
- [ ] Discussed tradeoffs of chosen approach
- [ ] Addressed collision handling
- [ ] Explained how it works with multiple servers

### Phase 5: Deep Dive - Database & Caching (30:00 - 40:00)

**Your task:** Detail the data storage strategy

Consider:
- Schema design
- SQL vs NoSQL choice and why
- Caching strategy (what to cache, TTL, invalidation)
- Replication and partitioning

**Checkpoint at 40:00:**
- [ ] Database schema defined
- [ ] Justified database choice
- [ ] Caching strategy explained
- [ ] Discussed how to scale storage

### Phase 6: Wrap-up & Extensions (40:00 - 45:00)

**Your task:** Discuss improvements and edge cases

Cover:
- How would you handle a spike in traffic?
- What happens if the database goes down?
- How would you add analytics?
- Rate limiting for abuse prevention

**Final Checkpoint:**
- [ ] Addressed at least one failure scenario
- [ ] Mentioned monitoring/observability
- [ ] Identified potential bottlenecks

---

## Session 2: Rate Limiter (Beginner-Intermediate)

**Total Time: 45 minutes**

### Phase 1: Requirements & Scope (0:00 - 5:00)

Questions to answer:
- What are we rate limiting? (API calls, login attempts, etc.)
- Client-side or server-side?
- What's the limiting criteria? (user ID, IP, API key)
- Hard limit or soft limit?
- Distributed across multiple servers?

**Checkpoint at 5:00:**
- [ ] Defined what's being limited
- [ ] Specified limiting criteria
- [ ] Clarified distributed requirement

### Phase 2: Algorithm Selection (5:00 - 15:00)

**Your task:** Explain rate limiting algorithms

Cover at least 3:
- Token Bucket
- Leaky Bucket
- Fixed Window Counter
- Sliding Window Log
- Sliding Window Counter

**Checkpoint at 15:00:**
- [ ] Explained 3+ algorithms with pros/cons
- [ ] Selected one for this design with justification
- [ ] Discussed memory requirements of each

### Phase 3: High-Level Design (15:00 - 25:00)

**Your task:** Draw the architecture

Include:
- Where does rate limiter sit in the request flow?
- Rate limiter service design
- Data store for counters
- What happens when limit exceeded?

**Checkpoint at 25:00:**
- [ ] Diagram shows rate limiter placement
- [ ] Showed interaction with data store
- [ ] Defined response when rate limited (429, headers)

### Phase 4: Deep Dive - Distributed Rate Limiting (25:00 - 35:00)

**Your task:** Handle multiple servers

Consider:
- Synchronization across servers
- Race conditions
- Consistency vs performance tradeoffs
- Sticky sessions alternative

**Checkpoint at 35:00:**
- [ ] Addressed race condition problem
- [ ] Discussed Redis atomic operations or similar
- [ ] Explained consistency tradeoff

### Phase 5: Rules & Configuration (35:00 - 42:00)

**Your task:** Design the rules system

Consider:
- Different limits for different endpoints
- Different limits for different user tiers
- Configuration storage and updates
- Dynamic rule changes without restart

**Checkpoint at 42:00:**
- [ ] Showed how rules are stored
- [ ] Explained rule evaluation logic
- [ ] Discussed rule update mechanism

### Phase 6: Monitoring & Wrap-up (42:00 - 45:00)

Cover:
- What metrics would you track?
- How to alert on issues?
- How to test the rate limiter?

---

## Session 3: Twitter Feed (Intermediate)

**Total Time: 45 minutes**

### Phase 1: Requirements & Scope (0:00 - 5:00)

Questions to answer:
- Core features: tweet, follow, timeline
- Scale: users, tweets/day, timeline requests/day
- Timeline: home timeline vs user timeline
- Real-time updates needed?

Target numbers to state:
- 300M monthly active users
- 600 tweets/second
- 600K timeline reads/second

**Checkpoint at 5:00:**
- [ ] Listed core features
- [ ] Stated scale numbers
- [ ] Clarified home timeline is the focus

### Phase 2: High-Level Design (5:00 - 15:00)

**Your task:** Draw the system

Include:
- Tweet service
- User service
- Timeline service
- Fan-out service
- Data stores

**Checkpoint at 15:00:**
- [ ] Major services identified
- [ ] Data flow for posting a tweet shown
- [ ] Data flow for reading timeline shown

### Phase 3: Deep Dive - Fan-out Strategy (15:00 - 28:00)

**Your task:** Design how tweets reach followers

Compare approaches:
- Fan-out on write (push model)
- Fan-out on read (pull model)
- Hybrid approach

For each, discuss:
- Write amplification
- Read latency
- Storage requirements
- Celebrity problem

**Checkpoint at 28:00:**
- [ ] Explained both push and pull models
- [ ] Discussed celebrity problem (users with millions of followers)
- [ ] Proposed hybrid solution
- [ ] Justified where to draw the line (e.g., >10K followers = pull)

### Phase 4: Data Model & Storage (28:00 - 38:00)

**Your task:** Design the data layer

Cover:
- Tweet storage (schema, database choice)
- User relationships (follow graph)
- Timeline cache structure
- Media storage

**Checkpoint at 38:00:**
- [ ] Tweet schema defined
- [ ] Follow relationship storage explained
- [ ] Timeline cache structure (Redis list of tweet IDs)
- [ ] Partitioning strategy mentioned

### Phase 5: Additional Features & Wrap-up (38:00 - 45:00)

Discuss:
- How to handle tweets with media?
- How to implement search?
- How to show trending topics?
- Failure scenarios

**Final Checkpoint:**
- [ ] Addressed at least 2 additional features
- [ ] Mentioned one failure scenario
- [ ] Could explain any component in more depth if asked

---

## Session 4: Chat System (Intermediate)

**Total Time: 45 minutes**

### Phase 1: Requirements & Scope (0:00 - 5:00)

Questions to answer:
- 1:1 chat, group chat, or both?
- Scale: concurrent users, messages/day
- Features: read receipts, typing indicators, presence?
- Message history: how long stored?
- Media support?

**Checkpoint at 5:00:**
- [ ] Scoped to 1:1 and small group chat
- [ ] Stated scale (e.g., 50M DAU, 1B messages/day)
- [ ] Listed must-have vs nice-to-have features

### Phase 2: High-Level Design (5:00 - 15:00)

**Your task:** Draw the architecture

Include:
- Connection layer (WebSocket servers)
- Chat service
- Message storage
- Presence service
- Notification service

**Checkpoint at 15:00:**
- [ ] WebSocket servers shown
- [ ] Message flow: sender -> server -> receiver
- [ ] Offline message handling addressed
- [ ] Major services identified

### Phase 3: Deep Dive - Connection Management (15:00 - 25:00)

**Your task:** Design the real-time connection layer

Cover:
- WebSocket vs long polling
- Connection server design
- How to route messages to the right connection server
- Handling connection drops and reconnects

**Checkpoint at 25:00:**
- [ ] Justified WebSocket choice
- [ ] Explained connection server statelessness challenge
- [ ] Showed how to find which server a user is connected to
- [ ] Discussed reconnection and message sync

### Phase 4: Deep Dive - Message Storage (25:00 - 35:00)

**Your task:** Design message persistence

Cover:
- Message schema
- Database choice (SQL vs NoSQL)
- Message ordering guarantees
- Read/unread status tracking
- Message retention and deletion

**Checkpoint at 35:00:**
- [ ] Message schema defined
- [ ] Partitioning strategy (by conversation ID)
- [ ] Ordering discussed (timestamps, sequence numbers)
- [ ] Sync protocol for message history

### Phase 5: Group Chat & Presence (35:00 - 42:00)

**Your task:** Handle group chat and online status

Cover:
- Group message fan-out
- Group size limits and why
- Online/offline status updates
- Typing indicators

**Checkpoint at 42:00:**
- [ ] Group fan-out strategy explained
- [ ] Presence system design (heartbeats, pub/sub)
- [ ] Typing indicator implementation

### Phase 6: Wrap-up (42:00 - 45:00)

Discuss:
- End-to-end encryption considerations
- Push notification integration
- How to handle message edits/deletes

---

## Session 5: File Storage (Dropbox) (Intermediate-Advanced)

**Total Time: 45 minutes**

### Phase 1: Requirements & Scope (0:00 - 5:00)

Questions to answer:
- Upload, download, sync across devices
- File size limits?
- Version history?
- Sharing and collaboration?
- Conflict resolution?

**Checkpoint at 5:00:**
- [ ] Core features listed
- [ ] Scale assumptions stated
- [ ] Clarified sync is the key challenge

### Phase 2: High-Level Design (5:00 - 15:00)

**Your task:** Draw the architecture

Include:
- Client application
- API servers
- Metadata database
- Block storage (S3)
- Sync service
- Notification service

**Checkpoint at 15:00:**
- [ ] Separation of metadata and file content shown
- [ ] Upload and download flows diagrammed
- [ ] Sync notification mechanism included

### Phase 3: Deep Dive - Chunking & Deduplication (15:00 - 27:00)

**Your task:** Design efficient file transfer

Cover:
- Why chunk files? (resume, delta sync)
- Chunk size selection (4MB typical)
- Content-defined chunking vs fixed-size
- Deduplication using content hashing
- Compression

**Checkpoint at 27:00:**
- [ ] Explained chunking benefits
- [ ] Described dedup using SHA-256 or similar
- [ ] Discussed delta sync (only changed chunks)
- [ ] Addressed chunking for different file types

### Phase 4: Deep Dive - Sync Protocol (27:00 - 38:00)

**Your task:** Design multi-device synchronization

Cover:
- How client detects local changes
- How server notifies clients of remote changes
- Conflict detection and resolution
- Handling offline edits

**Checkpoint at 38:00:**
- [ ] Local change detection (file watcher, checksums)
- [ ] Server notification (long polling, WebSocket)
- [ ] Conflict resolution strategy (last writer wins, or save both)
- [ ] Sync state machine explained

### Phase 5: Metadata & Wrap-up (38:00 - 45:00)

Cover:
- Metadata schema (files, folders, versions)
- Database choice and sharding
- How to handle very large folders
- Failure scenarios

**Final Checkpoint:**
- [ ] Metadata schema reasonable
- [ ] Can explain end-to-end upload flow
- [ ] Can explain end-to-end sync flow
- [ ] Addressed at least one edge case

---

## Session 6: Video Streaming (YouTube) (Advanced)

**Total Time: 45 minutes**

### Phase 1: Requirements & Scope (0:00 - 5:00)

Questions to answer:
- Upload and playback
- Scale: videos uploaded/day, concurrent viewers
- Video quality options?
- Live streaming or just VOD?
- Recommendations?

**Checkpoint at 5:00:**
- [ ] Focused on VOD (not live)
- [ ] Scale: 500 hours uploaded/minute, 1B daily views
- [ ] Listed adaptive bitrate as requirement

### Phase 2: High-Level Design (5:00 - 15:00)

**Your task:** Draw the architecture

Include:
- Upload service
- Transcoding pipeline
- CDN
- Video metadata service
- Streaming servers

**Checkpoint at 15:00:**
- [ ] Upload to transcoding pipeline shown
- [ ] CDN for video delivery
- [ ] Metadata separate from video content

### Phase 3: Deep Dive - Video Processing Pipeline (15:00 - 27:00)

**Your task:** Design the transcoding system

Cover:
- Upload handling (chunked upload, resume)
- Transcoding to multiple resolutions
- Encoding formats (H.264, VP9, etc.)
- Thumbnail generation
- Processing queue and parallelization

**Checkpoint at 27:00:**
- [ ] Explained transcoding workflow
- [ ] Discussed multiple resolution outputs
- [ ] Covered queue-based processing
- [ ] Addressed how to handle failures mid-transcode

### Phase 4: Deep Dive - Video Delivery (27:00 - 38:00)

**Your task:** Design the streaming system

Cover:
- HLS/DASH adaptive streaming
- Video segmentation
- CDN caching strategy
- Handling popular vs long-tail content
- Playback experience (seeking, quality switching)

**Checkpoint at 38:00:**
- [ ] Explained adaptive bitrate streaming
- [ ] Discussed CDN edge caching
- [ ] Addressed seeking in video
- [ ] Covered origin shielding for long-tail

### Phase 5: Data Model & Wrap-up (38:00 - 45:00)

Cover:
- Video metadata schema
- View count tracking at scale
- Recommendation integration points
- Cost optimization strategies

**Final Checkpoint:**
- [ ] Can trace upload from client to playback
- [ ] Understands CDN role
- [ ] Can discuss one optimization
- [ ] Addressed cost considerations (storage, bandwidth)

---

## Post-Session Review Template

After each session, fill out this review:

```
Session: _______________
Date: _______________

Time Management:
- Did I finish each phase on time? Y/N
- Which phase took too long? _______________
- Which phase was I rushing? _______________

Requirements Phase:
- Did I ask clarifying questions? Y/N
- Did I make reasonable assumptions? Y/N
- Did I scope appropriately? Y/N

Design Quality:
- Was my diagram clear? Y/N
- Did I justify my choices? Y/N
- Did I discuss tradeoffs? Y/N

Communication:
- Did I think out loud? Y/N
- Was I structured in my explanation? Y/N
- Did I use technical terms correctly? Y/N

Areas to Improve:
1. _______________
2. _______________
3. _______________

What I Did Well:
1. _______________
2. _______________
```

---

## Weekly Practice Schedule

| Day | Session | Focus |
|-----|---------|-------|
| Monday | Session 1 | Fundamentals |
| Wednesday | Session 3 | Fan-out patterns |
| Friday | Session 4 | Real-time systems |
| Sunday | Session 6 | Media systems |

Repeat with Sessions 2 and 5 the following week.
