# Cassandra Summit Notes

## Keynote
- Cassandra overview
- Microsoft demo for Portal Azure: https://portal.azure.com/

## Talk 1: Visualizing & Simulating Large Scale Cassandra Deployments
github.com/adrianco/spigo

## Talk 2: Cassandra at Apple
~ 100,000 nodes currently running.
  **Repair:** Incremental repair internals and applications
    - Issues with full repair
      - Merkle tree generation is costly
      - Streaming large CQL partition
      - Long process
    - Incremental Repair: Separate out repaired and non-repaired data.
    - What challenges? 
      - Repaired sstables should be in sync.
      - Only mark repaired
      - If all replicas are involved. Disable parallel repair on same data

### Talk 3: How to roll Cassandra into Production

  Need strategy

  #### The Problem
    - We're building a platform
    - It's a 3 year project
    - We'll build it from scratch with all the lessons learned from last app.
    - Rewrite everything using technology X

  #### How
    - Pick a pain point
    - website & monitoring
    - alert, event tracking, ...
    - event tracking: Kafka, Cassandra, Spark

  *Forget about Unicorn*
  ### When to use RDBMS?
    - Loose data model
    - Absolute consistency (ACID)
    - No need to use anything else
    - Go away from Oracle

  ### When to use Cassandra?
    - Uptime is top priority
    - Unpredictable or high scaling requirements
    - Workload is transactional

  ### Fallacies of distributed computing
    - The network is reliable
    - Latency is zero
    - Bandwidth is infinite
    - The network is secure
    - Topology doesn't change
    - There is one administrator
    - Transport cost is zero
    - network is homogeneous

### Talk 4: Change to Cassandra


  #### Subscription & Billing Platform
  - 88M subscriptions
  - 66M unique users
  - 105M transactions a day
  Require:
  - API Service
  - Manage users subscriptions
  - Charge users in carriers
  - Renew subscriptions

  Timeline: 2008 (RDBMS), 2009 started Cassandra, 2015 migrated most of
  complex queries away from RDBMS.

  Problem with original architecture:
  - Single point of failure (DB)
  - Slow response times
  - platform down often
  - hard/expensive to scale
  - if scale platform, but forgot scale database, other related resource will
      fail.

  Initial Migration: hybrid
  APIs <=> Cassandra <=> Engine <=> SQL queries to RDBMS
  Result:
  - performance: solved
  - availability: solved
  - single point of failure: partially solved
  - read/write increase

  What can be improved?
  APIs <=> Cassandra <=> [Engine <=> SQL queries to RDBMS]
  - query RD consumes time
  - side effects, locks data being updated & inserted
  - concurrency causes performance degraded
  - *not scale*
  - still need RB for complex query? (subscription table, expired subscription,
      must be ordeered by priority, criteria, type of user plan...)

  Solving complex query:
  - extract data from Cassandra instead of RB
  - No more single point of failure
  - improve performance

  How to sort data by multiple criteria?
  - Apache Spark
  Cassandra -> Data Extractor -> Processor <=> Spark
  Resources:
    https://github.com/eiti-kimura-movile/spark-cassandra

    #### Kiwi migration
    Use cases:
    - served as backend for smartphone platform.
    - user device managements
    - user event & tracker
    - analytics
    - push notifications

    Original architect
      Devices -> API [<=> Dynamo DB <=>] -> Queue SQS -> Consumer -> Postgresql

      Push notification
      Postgresql -> publisher -> (user services: apple, google) -> devices

      Problems:
      - single point of failure DB
      - high cost
      - DynamoDB doesn't not have a good read throughput for linear readings
      - Postgresql tuning limit reached
      - low throughput sending notifications

      Cost:
        AWS DynamoDB + Postgres = 10,825.00/m
        Amazon DynamoDB: 1.4k/s (linear)
        Postgresql Throughput: ~ 10k/s

    New architecture
      Cassandra -> publisher -> (user notification service) -> devices

      Cost
        8nodes c3.2xlarge: $2,580/m
        read throughput: 200k/s

    Learning:
      - Always upgrade to newest versions.
      - High throughput & availability
 
### Talk 5: When & How to migrate from relational DB to Cassandra
When:
  - Reaching physical scalability limits
  - Licensing costs becoming prohibitive
  - Need 100% availability
  - Increase DBA time to maintain perfomance / availability
  - Active/Active multi-DC / disaster recovery

    Weight against cost
    - Initial migration
    - additional logic maintained in app.

- Preparing application
    Some approaches while still using relational can help reduce migration costs
      + Abstract data access layer
      + Denormalize within relation DB
      + Minimize logic implemented in DB
      + Build data validation checks & data profiles
- Approaches
    - Big bang cutover
        + Build & Test verion of app using C* and convert data from relational
            to C*
        + Shutdown relational, convert data, start-up con CAssandra
        => Requires downtown, high rist but lower effort.
    - Parallel run
        + Build C* tables
        + Modify application to write to both C* and relational (application
            write to both database at same time)
        + Develop & execute tool to perform initial sync and reconcilation of
            dbs.
        + Run & regularly reconcile
        + Migrate reads to C*
        => More complex to build  manage
        => Lower risk and can be down with no downtime
    - Table by Table / Function by function
        + Either big-bang or parallel run approaches can be done on a
            table-by-table basic.
        + Need to be able isolate subject areas with minimal joins in relational
            DB (likely to correspond to denormalized C* tables)
        + Allows staged implementation, gradually moving load from relational to
            C* - useful if relational environment is under immediate capacity
            pressure
        + Incrementally educe pression on relational.

- Estimating Guide
    - Work items
        Revise & test operational procedures
        Performance test & soak test
        Trial conversions
        Execute production migration
        Application changes & regression test
        Build migration tool
        Build reconcillation tool
        Build C* schema

    - Effort Drivers
        # of source tables
        # of access paths
        migration approach
        level of preparedness
- Considerations
    - Don't forget about analytics/ad-hoc querying requirements
    - Denormalize - it should feel wrong
    - Modeling traps
        - partition keys
        - tombstones
        - secondary indexes
        - Make sure reads work before migrating.

## Day 2


### Talk 4: PS4 talk
- PS network is huge: 65 million montly active users. And continue developing.
- Including: PS: Network, Store, Video, Vue, Music
Serving live traffic
- 10s clusters
- nodes per clusters: 4 - 100
- 10s of GB per second in data transfer
- 100s TB raw data

Ex: Friend search and social graph for friends. 
Using personalized Search: PS4 -> Memory Index -> App -> Cassandra

Stories:
Astyanax/Thrift Memory Leak
Row Caching: using Redis, Memcache... -> Painful, takes forever to restart.

Using Spark:

Design for migration
- Change is inevitable
  + Applications and features may not scale as expected
  + New interactions and uses may cause more load on individual column families
  -> Strategy??
  Load Variation: where one or two node is overflowed.
- Migration:
    Replicate second clusters. 2xRead, 2xWrite. More expensive, but reliable.
    - Make sure monitor closely.
    - Integrity scripts.
- Migration & Segmentation:
    What to do when moving ~100 of nodes
  Isolate usecase to its own microservice.
What we need to keep in mind:
- Anticipate to critical areas and push into a separate keyspace
- Secure data
- Dependency to other services
- Unbound or potentially large amount of graph edges.
Application has separate connection pools for critical area interactions.
- Monitor, monitor, and monitor.

--- Issues ---
https://issues.apache.org/jira/browse/cassandra

Node decommisions itself
- Node has 2 disks with more than 50% free but not balanced


### Talk 4: Capital One: Using Cassandra In Building A Reporting Platform
- Platform: Kafka, Go / Docker, Cassandra, AWS
- Decisions: Move data when available, transform when all data available.
    -> Cassandra: CAP: Empasis on A & P with tunable C
    - Wide row ables.
        - Linear scalabilty
    - Migrate data between multi-DC
- W Consistency + R Consistency  > Replication Factor

Performance:
- 3 replication factor
- Write heavy
    - concurrent writes to 64
    - decreased concurrent reads to 16 (from 32)
    - virtual nodes 16.

- Solutions
    Go Service -> ( API Server, Kafka, Consul ) -> Go Service Processor -> Cassandra
    Plugin framework
    - Go service: Plugins chained in a single process
    - Packaged & deployed in a docker container
    - Bootstrapped from config

    [ Go Service -> ( API Server, Kafka, Consul ) -> Go Service Processor -> Cassandra ] => Mesosphere

- Challenges:
    - Not all query patterns are known in advance
    - Index rebuilds are costly

### Talk 5: The Last Pickle

Application Design
1. Use Mixed Workload: Separate table by read or write heavy.
2. Use LeveledCompaction Strategy as Tobmstone.
3. Create parallel data models so throughput increases with node count: Adding
   bucket to spreadout throughput.
4. Use concurrent asynchronous requests to complete tasks.

Application Development
  - Use token aware asynchronous requests with CL ONE where possible.
  - Monitoring & Alerting: Use what you like and what works for you. OpsCenter,
      Reimann, Grafana, Log Stash, Sensu.

  - Cluster wide aggregate. All nodes (if possible). Top 3 & Bottom 3.
  - Monitor latency: 75th, 95th, 99th percentile.

### Talk 6: Trove
- Architecture
- functionality:
  Provisioning: single instances, replicated groups, clusters
  - Backup & Restore
  - Replication
  - Clustering
  - Database configuration management
  - Resize instance & storage

Why this makes sense
  - Databases are complex: setup is complex, failure modes are complex,
      configuration options are numerous.
  - Data lost, and data security
  - There are number of databases in the organization
      - SQL, NoSQL, Relational ...

Why Trove?
- API for standard operations
- Abstractions for popular database
- Integrate best practice for each database

Currently support
  MySQL
  PostgreSQl
  Vertica
  Cassandra


