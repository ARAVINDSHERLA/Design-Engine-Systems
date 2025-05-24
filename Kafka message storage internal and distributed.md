# Kafka message storage internal and distributed
Apache Kafka is a distributed streaming platform designed for high-throughput, low-latency, and fault-tolerant handling of real-time data feeds. Its strength lies in how it stores and manages messages across a distributed cluster.

## Kafka Message Storage: Internal and Distributed Working

### HLD (High-Level Design)

At a high level, Kafka's architecture is built around:

1.  **Producers:** Applications that publish (write) messages to Kafka topics.
2.  **Topics:** Logical categories or feeds of messages.
3.  **Partitions:** Each topic is divided into one or more partitions. Partitions are the fundamental unit of parallelism and storage in Kafka.
4.  **Brokers:** Kafka servers that store messages for topics/partitions and serve client requests. A Kafka cluster consists of multiple brokers.
5.  **Consumers:** Applications that subscribe to and read messages from Kafka topics.
6.  **ZooKeeper (or KRaft):** (Historically ZooKeeper, now increasingly KRaft for newer Kafka versions) A distributed coordination service used for managing metadata, leader election, and maintaining cluster state.

**Distributed Nature:**

* **Scalability:** Partitions allow Kafka to horizontally scale. Messages for a topic are distributed across its partitions, which can reside on different brokers. This means a single topic can spread its load across multiple servers.
* **Fault Tolerance:** Each partition can have multiple replicas (copies) on different brokers. If a broker fails, another replica can take over, ensuring data availability and durability.
* **High Throughput:** The append-only nature of logs and sequential disk I/O, combined with batching and zero-copy transfers, contributes to Kafka's high throughput.

### LLD (Low-Level Design) & Internal Working

Let's dive into the specifics of how messages are stored and managed.

#### 1. Topics and Partitions

* **Topics:** A logical collection of messages. Producers write to topics, and consumers read from topics.
* **Partitions:** A topic is sharded into partitions. Each partition is an ordered, immutable sequence of messages (an append-only log).
    * **Ordering Guarantee:** Message ordering is guaranteed *within a single partition*. There's no global order across partitions of a topic.
    * **Parallelism:** Partitions are the unit of parallelism for both producers (writing to different partitions) and consumers (different consumers in a group reading from different partitions).
    * **Physical Storage:** Each partition is stored on disk at a Kafka broker. A broker can host multiple partitions from different topics, and multiple partitions from the same topic.

#### 2. Message Structure

A Kafka message (or record) typically consists of:

* **Key (optional):** A byte array. Used for partitioning (messages with the same key go to the same partition) and for log compaction.
* **Value (payload):** The actual message data (byte array).
* **Timestamp:** Time at which the message was created or appended to the log.
* **Offset:** A unique, monotonically increasing ID assigned by Kafka to each message within a partition. It represents the message's position in the partition's log.
* **Headers (optional):** Key-value pairs for attaching metadata.
* **Checksum:** For data integrity verification.
* **Message Size:** Size of the message.
* **Magic Byte:** Indicates the message format version.
* **Compression Codec:** If compressed (Snappy, GZip, LZ4, ZSTD).

#### 3. Log Segments

* Each partition's log is further divided into **log segments**.
* When a Kafka broker receives data for a partition, it appends messages to the currently active log segment.
* A segment is **rolled** (closed and a new one opened) when:
    * Its size reaches a configurable limit (`log.segment.bytes`, default 1GB).
    * A configurable time limit has passed since its creation (`log.segment.ms`, default 7 days).
* **Immutability:** Once a segment is rolled, it becomes immutable. New messages are always appended to the active segment.
* **Retention Policy:** Kafka retains messages for a configurable period (e.g., `log.retention.ms`, `log.retention.bytes`). When a segment's retention period expires, the *entire segment file* is deleted, which is an efficient disk operation.
* **Storage Location:** For a topic `my_topic` with 3 partitions, on a broker's `log.dirs` path, you'd find directories like:
    ```
    /kafka-logs/my_topic-0/
    /kafka-logs/my_topic-1/
    /kafka-logs/my_topic-2/
    ```
    Inside each partition directory, you'll find segment files.

#### 4. Segment Files (`.log` and Index Files)

Each log segment comprises a `.log` file and associated index files:

* **`.log` file:**
    * This is where the actual messages are stored sequentially as a raw byte array.
    * Messages are appended to the end of this file.
    * Kafka uses a "zero-copy" optimization to send messages directly from disk to network sockets when consumers fetch data, avoiding intermediate buffer copies and improving performance.
* **`.index` (Offset Index) file:**
    * Stores a mapping between message **offsets** and their physical **positions** (byte offsets) within the corresponding `.log` file.
    * These are sparse indexes, meaning not every message offset is indexed. Entries are typically created at regular intervals (e.g., every `log.index.interval.bytes`).
    * **DSA:** A sorted array or similar structure, allowing for binary search to quickly find the file position for a given offset.
    * **Purpose:** To quickly locate a message given its offset without scanning the entire `.log` file. When a consumer requests an offset, Kafka uses the index to find the nearest entry and then scans forward from that position in the `.log` file.
* **`.timeindex` (Timestamp Index) file:**
    * Stores a mapping between message **timestamps** and their corresponding **offsets**.
    * **Purpose:** Enables efficient retrieval of messages based on their timestamp (e.g., "give me all messages from the last hour").
* **`.txnindex` (Transaction Index) file:**
    * (Introduced with transactions) Stores information about aborted transactions to support exactly-once semantics.

#### 5. Producer Workflow (Writing Messages)

1.  **Serialization:** The producer application serializes the message key and value into byte arrays.
2.  **Partitioning:**
    * If a partition is explicitly specified, the producer sends the message to that partition.
    * If a key is provided, a default partitioner (e.g., Murmur2 hash of the key modulo number of partitions) ensures that all messages with the same key go to the same partition, preserving order for that key.
    * If no key is provided, a round-robin or sticky partitioner is used to distribute messages evenly across partitions.
3.  **Batching:** Producers buffer messages internally and send them in batches to reduce network overhead and improve throughput.
4.  **Sending to Leader Broker:** The producer client connects to the **leader** broker for the target partition. Kafka brokers manage which replica is the leader for each partition.
5.  **Appending to Log:** The leader broker appends the message batch to the end of the active `.log` file for that partition on its local disk.
6.  **Replication:**
    * The leader broker replicates the message to its **follower replicas** on other brokers.
    * Producers can configure `acks` (acknowledgments) to control durability:
        * `acks=0`: Producer doesn't wait for any acknowledgment (lowest latency, highest risk of data loss).
        * `acks=1`: Producer waits for the leader to acknowledge successful write to its local log.
        * `acks=all` (or `-1`): Producer waits for the leader *and* all in-sync replicas (ISRs) to acknowledge the write (highest durability, highest latency).
7.  **Acknowledgment:** Once the specified `acks` level is met, the leader sends an acknowledgment back to the producer.

#### 6. Consumer Workflow (Reading Messages)

1.  **Consumer Groups:** Consumers organize into consumer groups. Each message in a topic partition is delivered to exactly one consumer within a consumer group. This allows for parallel processing across multiple consumers.
2.  **Partition Assignment:** Partitions are assigned to consumers within a group. This assignment is managed by the group coordinator (a Kafka broker) and can involve rebalancing when consumers join or leave the group.
3.  **Fetch Requests:** Consumers send "fetch" requests to the **leader** broker for their assigned partitions, specifying the desired `offset` to start reading from.
4.  **Reading from Log:** The broker reads messages from its local `.log` files (using the index files for efficient lookup) starting from the requested offset.
5.  **Offset Management:** Consumers track their progress by committing their consumed offsets.
    * **Automatic Commit:** Consumers can automatically commit offsets periodically.
    * **Manual Commit:** Applications can explicitly commit offsets, giving more control over message processing semantics (e.g., "at least once," "at most once," "exactly once").
    * **Offset Storage:** Committed offsets are stored in a special Kafka topic (`__consumer_offsets`).

#### 7. Distributed Coordination (ZooKeeper/KRaft)

Historically, Kafka relied heavily on Apache ZooKeeper for cluster coordination. Newer versions of Kafka are moving towards a self-managed metadata quorum using the **KRaft (Kafka Raft) protocol**, eliminating the need for ZooKeeper.

**Role of ZooKeeper (in older versions or if still used):**

* **Controller Election:** ZooKeeper facilitates the election of a "Controller" broker in the Kafka cluster. The Controller is responsible for managing partition leaders, follower synchronization, and overall cluster metadata.
* **Cluster Membership:** Brokers register ephemeral nodes in ZooKeeper, allowing the Controller to detect when brokers join or leave the cluster.
* **Topic Configuration:** Stores metadata about topics (e.g., number of partitions, replication factor).
* **ACLs (Access Control Lists):** Stores authorization information.
* **Consumer Group Coordination:** Manages consumer group membership and offset commits.

**Role of KRaft (in newer versions):**

* KRaft implements a Raft-based consensus protocol directly within Kafka brokers.
* It combines the roles of the Kafka Controller and the metadata storage, removing the ZooKeeper dependency.
* This simplifies Kafka deployment and operation, and improves scalability of the metadata layer.

### DSA (Data Structures and Algorithms) in Kafka Storage

* **Append-Only Log Files:** Sequential writes are highly efficient on disk (especially HDDs).
* **B-Trees / Sorted Arrays (for Indexes):** The `.index` and `.timeindex` files can be thought of as sparse B-tree-like structures or sorted arrays that enable efficient binary search for offsets and timestamps.
* **Hash Functions:** Used by producers for partitioning messages based on keys (e.g., Murmur2 hash).
* **Distributed Hash Tables (Conceptual):** The combination of topic-partitioning and leader election can be seen as a form of distributed hash table where the key is (topic, partition) and the value is the set of brokers holding its replicas.
* **Replication Protocols:** Kafka implements its own replication protocol (ISR - In-Sync Replicas) to ensure data consistency and fault tolerance.
* **Consensus Algorithms (KRaft):** Raft algorithm for distributed agreement on metadata and leader election.
* **Zero-Copy Optimizations:** Uses `sendfile()` system call to transfer data directly from disk to network without buffering in application memory.
* **Batching:** Producers batch messages to optimize network utilization.
* **Log Compaction:** A feature for topics with keys, where Kafka retains only the latest message for each key within a partition, effectively cleaning up older versions.
* This involves internal merging and rewriting of log segments.

Kafka's design prioritizes high throughput, low latency, and durability through its combination of distributed, append-only logs, efficient indexing, and robust replication mechanisms.
