# Data Engineering Design Patterns

> Data just sitting in the storage is not a real asset

Kappa architecture is a new norm - combining streaming and batch processing
Medallion architecture - bronze, silver, gold - is about data maturity
- silver - data quality
- gold - data marts - aggregation for users

Backfilling - is when we process data from the past - with the new code

**Full load** - usually EL only
- if we start adapting input to output - it is become transform, data is NOT raw anymore
  - drop-and-insert - might lead to data consistency - because of full overwrite
  - view on top of table to prevent issues when smth try to read from it
      - single data exposition abstraction
  or
      - timetravel available appart of other things in GCP or Apache Iceberg

**Incremental** is a perfect fit when input data grow
- "delta column" - usually ingestion time
  - but not event time - to avoid issues without of order events and late data
- time partitioned dataset - usually much simpler to deal with - clear-cut of new data
  - might require "Readiness Marker" - hey, that crap is ready - sensor in many frameworks - yes, in airflow
- **_Hard delete_** for incremental might be tricky - **_soft delete_** to the rescue
	insert or append only tables might be an option - cost is to be more complex for data consumers 
	(or require re-shaping user facing layer)
- Backfilling for incremental data - shitload of compute resource
  - **_time windowing_** -> control for compute + option for parallel run
	
### CDC - change data capture

Read stream of changes from WAL or other kind of db logs/journal
- internal database commit log (which by default not usually enabled)
- re-frame the problem to consume history of changes (and handle hard delete out of the box)
- it is usually event state AND its metadata
- it consumes **_data at rest_** but produce **_data in motion_**
  - leading to eventuality of data - not processed YET
- who said Debezium
	
### Replication
is same-same but different
- you need _same_ data - PK, now(), etc - i.e. idempotent changes
  - compute replication - EL - very much replication
  - infra replication - data copied by data storage provider 
- either way, **_push_** - aka I am owning the data is ready - instead of **_pull_**, might be a safer bet
- TIL; aws s3 bucket replication
- **_Passthrough replicator_** - transform stage between - mapper function or data reduction (fields preselecting)
    ```sql
    SELECT * EXCEPT (pii_email, pii_credit_number, ... )
    ```
    - grant select on specific columns might do the tricks (Fine grainer accessor pattern)
		
### Compactor
To tackle problem of metadata overhead: 
- many small files (with data) = slow to list, too much IO -> merge to a single one
- **_compaction_** - merge many small file to a bigger one
- **_vacuum_** - get rid of small one (and tell metastore to forget about them)
  - ACID properties not available out of the box for csv or json
  - vacuum could remove the files that are being written and are not yet committed
- TIL; Apache Hudi use Merge-On-Read table - first writes in columnar, subsequent changes in row 0_o
- no one-fit-all, depending on data access pattern - run it outside of working hours
  - ingestion throughput or penalizing consumers

	
### Data readiness

Ready for consumption for downstream processes - when I should bother to read the data
- **_flag file_** - Spark's _SUCCESS or just job's "touch my_table.ready"
- **_next partition detection_** - if there are date 01-02 partition exist - 01-01 assumed to be fully processed
- explaining to consumers
  - risks of not respecting the readiness conventions
  - special treatment to dealing with late data - should we mutate existing partitions
- proper orchestrator layer (e.g. airflow) actually do not block the worker slot by constantly checking sensor status
	
#### Event Driven
No fixed schedule to check data availability
- **_push semantic_** -> producer inform consumer about changes
  - pros - do not bother with data crunching if nothing is changed
  - but can be pull as well - while True: if queue.pop(): proccessing() sleep(1)
- **_External trigger_** -> event to start the pipeline (or msg to process)
  - notification channel - even handler
  - error handler - keep events (with probably dead letter)
- and yep, you can have a dag in airflow triggered via external event - i.e. aws lambda function
  - post via airflow api

### Dead letter
- Skip bad data - transient errors (api or db unavailability with retries) 
  - vs poison pills that lead to job crash
  - key to success here - including problematic msg not just logging the fact of some issue
  - can be perfectly applicable for batch jobs - to raise visibility of data issues
- Replay Pipeline to integrate problematic events separately
  - downside - partial data + snowflake effects for all downstreams -> your consumers need to re-process data
  - extra metadata to mark records produced from dead letter	
  - ordering and consistency be affected
  - thats might be one of the source of late data

#### Consistency
Though is a challenge - i.e. try catch and return null without tackling exception - hiding the problem
- in declarative language - read sql - it can be extra fun because of verbosity and maintenance overhead
	
### Duplicates 
Stream is unbounded set of records -> set of prev processed records windowed 
- **_exactly once processing_** - require deduplication attributes
  - => Stateful transformation => state store 
    - local - (yep memory is a first candidate here) - easily can be lost
    - Local with fault-tolerance - periodic snapshotting to remote permanent storage layer - complexity
	- Remote - reliable but latency hit	
- Exactly once processing doesn't guarantee **_exactly once delivery_** or perfect deduplication
  - short time window - might miss duplicates
  - retries might be a reason for not so exactly-once delivery
	
### Watermark
- Define what is (most) **_late arrival_**
  - keep a state of processed records within it
- **Late Data Detector**
  - event time vs processing time
    - use watermark = max(event_time) - some_delta; 
  - for late events - data event capture
  - for moving watermark rely on max, as min will leave you back in time - to ensure monotonicity
  - side output available in Flink out of the box
- Static **lookback window** = process current + late date (yep downstream need to re-process all as well)
  - it can be sequentual or parallel (i.e. tackle late first then current, vice verse or do both parallel)
  - sequential good fit for stateful pipelines
  - and parallel - for stateless
  - backfill for that case have to take into account lookback window period
    - + it might be handy to generate backfill partitions to run backfill job per backfill partition
- to handle variability you might want to have **_dynamic lookback window_**
  - both of them might have an issue for cases when we do a parallel processing of multiple input partitions
  - with the same time -> how to integrate them into final output partition


### Filter interceptor:
Declarative language - read SQL - require more maintainance effort
	for streaming - expose states use a rolling window

### Checkpoint
As a tool to quickly recover from failures with the cost of some latency drop (complex state vs single numeric offest)
	not enough for idempotency (parallel jobs + failure in between - re-trie for processed points)	

### Delivery modes:
- Exactly once
- At least once
- At most once

### Idempotence
- great example is abs function
- remove-and-write - sort of works (loosing data history, compute intensive)
- metadata cleaners (truncate or drop table family of operations) or alias that expose data across many partitions
  - idempotency granularity - i.e. for weekly partitioned data
    - affect granularity of backfills
- schema evolution might be fun

### Data overwrite
- need special care in case of incremental data

### Merger
To tackle incremental data
- Upserts
  - However, for proper delete support - need to have soft delete
	- require definition of uniqueness identifier(s)
  - backfilling might be a problem here - as dataset return to proper state eventually
BUT
- in the middle of backfilling data maybe from the future

### Statefull Merger
Something that can help with the issue above
- however versioning every version of row for the case of change - looks like complex idea
- alternative - have an execution time - to keep track of the changes
- **Keyed Idempotency** - to combine all changes and be able to merge for RDBMs

### Proxy
We can solve any problem by introducing an extra level of indirection. (C)
- write once read many (WORM)
- Expose data in read-only fashion
	

### Data Value
- **Augment** the dataset to improve its usefulness
  - for example combining the data from multiple source
    - joins for a streaming data = Dynamic Joiner => key + time boundaries 
      - different dataset - different latencies utilising watermarking 
        - to tackle moving windows
- **Data Enrichment**
  - Static Joiner -> slowly changing dimensions (SCD)
    - SCD are different types
      - is_current
      - from to
- **Data Decoration** - Wrapper
  - keep original row and separately computed attributes, different layout possible:
    - collocate same table, fields composition - column by column vs structs
  - **_Metadata decorator_** - keep tech details - same-same but different semantic
  - **_Data Aggregation_** 
    - data + compute not always collocated => shuffle
      - which node which data should wrangle
        - data skew - salting (Spark now days have Adaptive Query Execution)
          - Shuffle service - responsible for data serving and storing
        - Local aggregator might help - but burden of partition the data is on "producer"
          - scaling is tricky - might need regenerate partition assignments
    - Bucketing/Clustering
  - **_Data Sessionization_**
    - incremental
      - Hell for backfilling
    - Stateful Sessionizer
      - session window with a gap duration (max allowed time between two events)
      - however definition of a gap can be dynamic
	
### Streaming 
Broker with partial commit semantics - challenge for ordered data delivery
- Bin Pack Orderer pattern ~ structure bins by order by key, time and run them sequentially
  - retries of pipeline
		
### Data flow:
**Orchestration** (DAGs - separate data crunching apps with stages) vs Processing itself 
  - to choose between - think about restart boundary or "transaction" ~ togetherness of step
- Sequence with Readiness Marker
  - die on failure - to avoid partially processed data - no data readiness flag
  - alternative for marker - data provider trigger a pipeline
- **Fan-In**
  - Aligned - i.e. ALL downstream ready
    - opportunities for parallel processing - faster and can retry specific stage only
        - for the cost of infra overhead and scheduling overhead and overall complexity
    - not-aligned - _some_ downstream ready = as soon as we have new partition - process it!
- **Fan-Out**
  - handy for cases when dataset is used by various child processes
  - **_Parallel Split_**
    - intermediate persistent dataset as input for downstream jobs
      - wait for the slowest
        - might require different and dedicated hardware resources 
  - **_Exclusive Choice_**
    - based on condition choose one of branch
      - might be in app or in orchestrator
        - might be heavy condition = need to crunch data to decide or based on schema
  - Single vs Concurrent Runner
    - Concurrent - beware of shared state but handy for backfilling

### Data Modeling
- **Vertical partitioner** - split data on mutable and non-mutable (birthday) ~ events + PII data
  - polyglot persistence - when data is written into multiple different data stores
- **In-Place Overwriter** - side file with tombstones - aka _deletion vector_ - separate file with deleted row - that consumer have to take into account on read time
  - alternative for that - staging dataset (private data not yet exposed to the user) and "compaction" job that prepare data to be public
    - still might be an issue when removal job have an issue and original dataset overwritten
      - dataset versioning can help in such cases
- **_Horizontal partitions_**
  - high cardinality => too many files to read
  - data skew -> when majority of records are bundled within specific partitions
- **_Vertical partitions_**
  - CTAS - Create Table As Select
- **Bucketing**
  - similar to horizontal, collocating records
  - bucket computed not on the actual value
  - hash(key) % buckets number
  - handy when key is often part of query
  - might result in Bucket Pruning or Shuffle illumination (in joins)
  - Changing bucket size is tricky - require backfilling
- **_Sorter_**
  - if query involve sorting by specific columns - you can take advantages of that knowledge when create table
    - sort by proximity of several attributes - z-order in data storage format
      - hence less  blocks to read
        - change sorting order -> re-sort the whole table
- **_Metadata Enhancer_**
  - data file contains stats ß data summary that can be used by query planner
    - still overhead to read however not the full scan
    - out of date statistics - require special process to refresh
- Materialization of frequent heavy queries - is a good idea
  - refreshing it, even manually, still not guarantee real state of data in upsource table
- **Manifest**
	with file name instead of listing operations (when metadata layer do not handle it) - might be a great speed up
- Data Consistency - **_dimensional (Star or Snowflake)_** VS **_normal forms_** for data representation
  - Star dimensional schema do not allow nested dimensions
    - with the overhead of joins multiple tables
      - broadcast join can help
- Denormalization help to reduce query time
  - updates might require to be done in several places - table for query and original with consistency	
    - storing extra data as a columns (**_One Big Table_**) or nested structures
    - beware of domain boundaries - naming might reveal redundant attributes

### Data Security
PII, PHI, PCI and other GDPR
- **Fine-Grainer Access** - to restrict access to specifc columns + Row Level access policies
  - Nested structure or limited db capabilities usually resolved via views
  - at-least privilege approach in the cloud world
    - resource level - GCS bucket
	- identity level - user
- **Encryption**
  - data at rest - server side or client-side encryption - either way key mgmt have to be thought about
  - data in transit - old good tls do you a favor
- **Anonymyzation**:
  - data removal
  - adding noise
  - syntheticly generated data
- **Pseudo-Anonymization**
  - masking/tokenization/hashing/encryption
- **Data Access**
  - **_Secret Pointer_**
    - get a secret value from 3rd party service on the fly, do not store as part of env
  - **_Secretless Connector_**
    - IAM or certificate based auth
      - still require assumed-role + something like AWS STS setup

### Data Quality
- **_Audit-Write-Audit-Publish (AWAP)_** - verify that both input and output dataset meet requirements (tech AND business): completeness and exactness
  - Non-blocking audit - expose dataset to users but annotate it with statistics of observed issues
  - For streaming - use time-window or staged intermediate storage
  - Enforcing Constraints
    - type/value/integrity/nullability constraints
    - Not only in rdbms - protobuf or avro have it out of the box
- Vary from consumer to consumer
- **_Schema Compatibility_**
  - validation via 3rd party service - i.e. schema registry
  - check for schema compatibility
  - alter schema 0_o
  - evolution is classical backward/forward/full
    - transitive i.e. throughout the whole history
      - renaming field -> create new and deprecating old
- **_Schema Evolution_**
  - rename/type change/field removal
  - break transitiveness
  - require grace period (for consumers to adapt)
- **_Offline Observer_** - external job(s) collecting data quality metrics
  - time accuracy
  - sampling - not to wrangle the whole dataset
- **_Online Observer_** - trigger DQ check right after the job itself
- **_Freshness_** - dataset availability check Flow Interruption Detector
  - Continues vs Irregular data delivery:
    - time window vs several time windows
  - Can be watched on different layers - Metadata/Data/Storage layer
    - metadata changes or compactions might distort metrics
- another "**_Data Skew_**" - use stats to find outliers - i.e. today we have 50% less or more then yesterday
  - use percentile as averaging might show good news even if everything falling appart
- Lag or more generally **_SLA Misses_** Detector
  - late data might be problematic

### Data Lineage
- vendor lock vs manual https://openlineage.io/ implementation	
- **_fine-grained_** - i.e. track source of column level
