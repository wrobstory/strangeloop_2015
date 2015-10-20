* Dan Frank Aggregator: MapReduce in the Type System
  * A lot of data analysis looks like Raw Data -> Statements about properties you care about
  * Facts That Happened -> Statements about Properties
  * Dimension reduction: Matrix to a array, array to scalar
  * Dimensionality reduction
  * Map-reduce feels like telling the program what you want to do, rather than what you want to calculate
  * It's totally tied to how you want the platform to work
  * Breaking the decomposition boundary
    * Group data points by entities of interest (keys)
    * Reduce a sequence of data points to a quantity of interest
  * `SELECT merchant, COUNT(DISTINCT(card))
     FROM charges
     GROUP BY 1;`
  * Aggregator: Scala trait in Algebird
  * Use types to guide composition and reuse
  * Abstracting away expression from computation!
  * Generic types are enormously powerful in building expression libraries
  * Algebird lets you swap out exact data structures/algorithms with approximate ones
  * Aggregator lets you combine stream + batch computations at any point

* Ines Sombra: Architectural Patterns of Resilient Distributed Systems
  * Resilience: It's what really matters
    * Very, very expensive for services to fall over. In many ways.
  * Three models that changed perspective
    * Harvest & Yield Model: "Harvest, Yield, and Scalable Tolerant Systems", Fox & Brewer
      * Yield: fraction of successfully answered queries.
        * Focus on yield rather than uptime.
      * Harvest: fraction of complete result
        * data available / total data
      * 1. Probabilistic availability
        * Graceful degradation under faults
      * 2. Decomposition and Orthogonality
        * Decomposing into subsystems independently intolerant to degradation but can continue to fail
    * Cook & Rasmussen Model: "Going solid: A Model of System Dynamics and Consequences for patient safety"
    * Borrill's Model: Many papers recommend by Borrill
      * ArchPatterns1.jpg 15:20
      * Different failure areas need different strategies!
      * ArchPatterns2.jpg 17:36
      * Borrill: "Thinking about building system resilience using a single discipline is insufficient. We need different strategies"
  * Resilience in Industry
    * Netflix
      * "Fault Tolerance in a High Volume, Distributed System"
      * Patterns: aggressive network timeouts and retries; semaphores
      * Separate threads on per-dependency thread pools
      * Circuit breaks to relieve pressure in underlying systems
      * Exceptions cause app to shed load until things are healthy
    * Google
      * Key insights
        * Library vs. Service? Service + client library control + stroage of small data files
        * Restricting user behavior increases resilience
    * Fastly
  * Resilient architectural patterns
    * Redundancies are key
      * Resources
      * Checks
      * Data replication
      * Message replay
      * Capacity planning matters!
    * Operations matter
      * Complex ops make systems less resilient and more incident-prone
      * You design operability too!
    * Not all complexity is bad
      * Complexity if increases safety is actually good
      * Adding this comes as cost of other goals: performance, simplicity, cost
    * Leverage Engineering best practices
      * Resiliency and testing are correlated
      * Versioning from the start
    * ArchPatterns3.jpg: Really great lessons here

* Martin Kleppmann: Transactions: myths, surprises and opportunities
  * No/NewSQL less about SQL, more about transactions
  * ACID: "More mnemonic than precise" - Brewer
  * Consistency: "Tossed in to make the acronym work" - Hellerstein
  * Atomicity: How faults are handled.
    * Transactions: multi-object atomicity
    * Roll back writes on abort/crash
    * "Abortability"
    * Allow you to collapse whole range of failure classes down to one thing: Abort
      * Deadlock
      * Network fault
      * Constraint violation
      * Crash/power failure
  * Isolation
    * Serializable
    * Repeatable Read
    * Snapshot Isolation
    * Read Committed
    * Read Uncommitted
  * Really nice description of Read Skew and Write Skew
  * How can we actually implement serializability?
    1. Lock everything you read
      * 2-phase Locking
        * Lock is held until commit/abort. If you need full-scan, you lock all rows!
    2. Literially execute all transactions in serial order
      * Means they need to be _really_ fast
      * Translation: keep everything in memory, locality of data/computation (stored procedures/partitioning)
    3. Detect conflicts and abort
      * Serializable Snapshot Isolation (SSI)
      * Idea: locks like in 2PL
      * BUT: locks dont block, only gather information
      * At the end it compares information, if there's a conflict, it aborts and retries
      * Great description of SSI!
  * Serializability across services?
    * Atomic commitment (2PC, 3PC)
    * Total ordering (Atomic broadcast)
    * Consensus (Paxos, Raft, Zab)
  * Coordination makes service infra brittle
  * Without cross-service transactions
    * Compensating transactions ~ abort/rollback at app level
    * Apologies - detect and fix constraint violations after the fact
  * "Every sufficiently large deployment of microservices contains an ad-hoc, informally-specified,
     bug-ridden, slow implementation of half of transactions"
  * Right now we have eventually consistent, and Serializability is too hard
    * How can we capture causality?
  * "Causality _can_ be maintained without global coordination"

* John Hugg, All In With Determinism for Performance and Testing in Distributed Systems"
  * So you need a replicated setup...
     * Option 1: Run Primary/Replica and sync
     * Option 2: Allow writes to two servers, do conflict detection and repair
  * Do same things in same order to two copies: order is a log
  * Can send logical log to all-the-things
  * Sources of Non-Determinism
    * Wall clock time
    * Random numbers
    * External systems
    * Duplicate data
  * Non-user Sources of Non-Determinism
    * Bad Memory
    * Libraries that randomize for security
  * Deterministic SQL
    * VoltDB SQL planner understand determinism
  * Deterministic APIs
  * No Divergence Allowed
    * Hash SQL write statement and params
    * Replicas compute hash of changes you made, coordinator confirms hashes are the same
  * Why Deterministic Logical Log for Sync Replication?
    * Replicate Faster: do tasks simultaneously
    * Persist Faster: start logging when the operation is ordered, rather than when it completes
    * Bounded Size
  * VoltDB Benchmarks:
    * 100-500k ACID Transactions/Server/Second
    * Millions of SQL Statement/Second/Server
    * Linear scale measured to 30 nodes, probably higher
    * Sync replication with minimal impact on latency
    * Sync disk persistence
  * Tradeoffs and Compromises
    1. It's more engineering work
    2. Running mixed versions is scary
    3. If trip non-determinism safety checks, snapshot divergent replicas and shut down cluster!
      * Data is safe, availability goes away
  * Testing
    * ACID, SQL correctness, integrations, performance
  * How does VoltDB test ACID?
    * Message/ State Machine Fuzzing
    * Unit Tests
    * Smoke Tests
    * Self-checking workload
  * Leveraging Internal Checking
    * SQL & parameters for SQL writes are hashed
    * "When in doubt, write it out!"
  * Develop nasty, long-running transactions with invariants that must be met for correctness. Clever.
    * VoltDB1.jpg
  * Really nice approach in creating most evil possible query to try and break their isolation
    * https://github.com/VoltDB/voltdb/tree/master/tests/test_apps/txnid-selfcheck2

* Camille Fournier, Hopelessness and Confidence in Distributed Systems Design
  * Distributed systems: Ugly, Hard, Here to Stay
  * How do we get through dist sys hope and fear
  * "Tradeoffs are what we need to fully embrace, fully understand about our systems..."
  * "We are never going to build a perfect thing: there are always going to be tradeoffs"
  * Fundamental Tradeoff/Conflict:
    * Scale
    * Failure Tolerance
  * "Scaling makes failure inevitable"
  * "Failure tolerance makes scaling harder"
  * "Rewriting successful production systems is one of the hardest things you will do in your life"
  * "The tradeoffs of microservices are just as much human tradeoffs as system tradeoffs. You trade off
     common understanding of the state of the world"

* Catie McCaffrey, Building Scalable Stateful Services
  * We're starting to hit the limits where one database doesn't cut it anymore
  * Data Locality: Ship the request to the machine that holds the relevant data
  * Building Sticky Connections
    * Client -> Request -> Server. Request always routed to same server.
    * Persistent Connection
      * Problems:
        * Load Balancing
        * No stickiness once connection breaks
      * Must implement backpressure to break connections when overwhelmed
    * Routing Logic
      * Client can talk to any server, request is routed to the correct data server
      * Cluster Membership
        * Static Cluster Membership
          * Not very fault tolerant
        * Dynamic Cluster Membership
          * Gossip Protocols
          * Consensus Systems
  * Work Distribution
    * Random Placement
      * Write anywhere, read from everywhere
    * Consistent Hashing
      * Deterministic Request Placement
    * Distributed Hash Tables
      * Non-Deterministic Placement
  * Stateful Services in the Real World
    * Scuba: in-memory database at Facebook
      * Random writes, fan-out reads, composed results
    * Uber Ringpop
      * Application-layer sharding to dispatching platform services
      * Swim gossip protocol + consistent hashing
    * Orleans
      * Runtime and programming model for dist sys based on Actor Model
        * Gossip Protocol + Consistent Hashing + Distributed Hash Table
        * Able to move work around in a non-deterministic fashion
  * Caution: Stateful Services Ahread
    * Unbounded Data Structures! Twin of Unbounded Queues for stateful services...
    * Memory Management (Make friends with the GC profiler)
    * Reloading State
      * First connection
      * Recovering from crashes
      * Deploying new code

* Michael Drogalis: Onyx: Distributed Workflows for Dynamic Systems
  * Take apart monolith- data and fns
  * Batch/stream hybrid
  * Transactions!
  * API dedicated to side-effects & state
  * Problem: Storage Medium -> Transformation -> Storage Medium
  * What if the DAG isn't known at compile time?
  * Specification of distributed execution:
    * Language agnostic
    * Location agnostic
    * Temporally agnostic
    * Tolerant to machine generation
  * "We've kind of become infatuated with how short we can write these programs.
     We're saying 'Oh, I can do word count in like 5 lines'""
  * Data Representation
    * Segment - just a Clojure map
    * No ordering, no explicit declaration
    * Values *must* be named
  * Expressions in Onyx are plain clojure functions
  * Positional representation: order that data is flowing through the system
  * Some similarities between Onyx and Dask, for sure
  * Extensibility: How do you reach new storage mediums without application code change?
  * Onyx makes it easy to swap different ingest/output modules, because you're just passing around maps & functions
  * Onyx lifecycles: managed setup/teardown of state
