# Storage and Retrieval

## Data Structures that power the Database

```bash
#!/bin/bash
db_set () {
 echo "$1,$2" >> database
}
db_get () {
 grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

Many databases internally use a log (append-only data file). 

They have to deal with concurrency control, reclaiming disk space, handling errors, etc.

### Indexes

To efficiently find a particular key in the database we need index.

The general idea of index is to keep some additional metadata on the side, which acts as a signpost and helps you to locate the data you want.

It is an additional structure which only affects performance of queries.

Maintaining index incurs an additional overhead of updating it everytime a write operation is performed.

*Indexes slow down the writes*.

### 1. Hash Indexes

This is index for key-value data.

The indexing strategy is to keep an in-memory hash map where every key is mapped to a byte offset in the data file.

This is useful in the kind of workload where there are a lot of writes but not too many distinct keys.

As we are appending to a file, we can avoid running out of disk spaces by **breaking the log into segments of certain size** by closing a segment file when it reaches a certain size and making subsequent writes to a new file.

- The segments are never modified after they have been written.
- We can perform compaction on the frozen segments in a background thread.
- While compaction we can still continue to server read and write requests as normal on old files.
- After merging/compaction we switch the read requests to using the new files.

In order to find the value of a key, we first check the most recent segment's hash map. If not present we check second most and so on.

For implementing this:

- **File Format**: A binary format that first encodes the length of string in bytes, followed by raw string.

- **Deleting Records**: Append a special deletion record to the data file (**tombstone**). When segments are merged tombstone tells to discard previous values.

- **Crash Recovery**: In memory hash maps are gone. We can restore them by reading the entire file but it takes time. Better way is to store a snapshot of each segment's hash map on disk.

- **Partially written records**: Use checksums, the corrupted parts of the log are detected and ignored.

- **Concurrency Control**: As writes are appended to log in a strictly sequential order, keep only one writer thread. As file segments are append-only and immutable they can be concurrently read by multiple threads.

**Appending vs Updating**:

- Appending and merging are sequential write operations which are generally much faster then random writes.

- Concurrency and crash recovery are much simpler if segments files are append-only and immutable.

- Merging old segments avoids the problem of data files getting fragmented over time.

**Limitations**:

- Hash table must fit in memory. Not suitable for large number of keys.

- Range queries are not efficient. Everytime we have to look up each key individually in the hash maps.

### 2. SSTables and LSM-Trees

**SSTables**: Sorted String Tables.

- The sequence of key-value pairs is sorted by key.

**Advantages**:

- Merging of tables becomes very easy. Merge algorithm in mergesort can be used here.

- In memory index table can be sparse because all the keys are sorted and we can find some with the help of others. No need to store all the keys.

- We can divide the data into blocks and keep the starting key in in-memory index. These blocks can be compressed.

**Construction**:

- Use in-memory balanced tree (AVL trees) structure while writing keys known as **memtable**.

- When memtable gets big enough read and write the keys (which are in sorted form) to the disk.

- Read first goes through memtable then to the disk logs.

- Keep a backup log of writes (not in sorted order) on disk which helps us in cases where system crashes and memtable is lost.

**LSM Trees**: Log-Structures Merge-Tree

SSTables are file format and LSM Tree is the storage engine which uses memtable and SSTables.

- These are used in RocksDB, Lucene which is full-text search indexing engine used by Elasticsearch

**Performance Optimizations**:

- These algorithms are slow for looking up a key which does not exist because they have to check memtable and multiple segments on disk.

- Bloom filters are used for these cases. A **Bloom Filter** is a memory-efficient data structure for approximating the contents of a set.

- Order and timing of compaction:

    1. Size-tiered: Newer and smaller SSTables are successively merged into older and larger SSTables. Use if write throughput is important. Used in HBase.

    2. Leveled: The key range is split up into smaller SSTables and older data is moved into seperate levels. Use if read throughput is important. Used in LevelDB, RocksDB.

### 3. B-Trees

- Key-value pairs sorted by key.

- Break the database into fixed-size blocks or pages (usually 4kb) and read/write one page at a time.

- Each page can be identified using an address or location. One page is designated as the root of the B-Tree.

- The page contains several keys and references to child pages. Each child page is responsible for a continuous range of keys.

- Eventually we get down to a leaf page which contains value for each key inline or references to the pages where value can be found.

- The number of references to child pages in one page of the B-Tree is called the **branching factor**.

- B-Tree is balanced and has max depth of O(logn)

**Making BTree Resilient**:

- The basic underlying write operation of a B-Tree is to overwrite a page on disk with new data.

- It is actual hardware operation and takes time. Moreover, some operations (insertion) require several different pages to be overwritten.

- A crash in between this can make the B-Tree corrupt.

- Due to this an addition data structure: a Write Ahead Log (**WAL**) is used. This is an append only file to which every B-tree modification must be written before it can be applied to the pages.

- An  addition complication while updating is that concurrency should be taken care of. Typically done using latches (lightweight locks).

**B-Tree Optimizations:**

- Instead of overwriting pages and maintaining WAL some DBs use copy-on-write scheme. A modified page is written to different location and a new version of parent pages in the tree is created, pointing to the new location

- Space can be saved by not storing entire key but abbreviating it.

- It is better to have leaf pages appear in sequential order on disk (especially for range query operations).

- Leaf pages may have additional pointers to the sibling pages.

- Variants like fractal trees borrow some log-structured ideas to reduce disk seeks.

### B-Trees VS LSM-Trees

- LSM Trees are better for write heavy workload and B-trees are better for read heavy workloads.

**Advantages of LSM Trees:**

- A B-Tree index must write every piece of data at least twice (WAL and tree).
 
- Log strucutred indexes also rewrite data multiple times due to compaction. 

- This effect that one write to the database resulting in multiple writes to the disk over the course of the database's lifetime is known as write amplification.

- LSM trees have lower write amplification because they are sequentially write compact SSTable files rather than having to overwrite several pages in the tree.

- LSM trees can be compressed better and don't have fragmentation.

**Advantages of B-trees**:

- Compaction process of LSM trees can sometimes interfere with the performance of ongoing reads and writes.

- If write throughput is high and compaction is not configured carefully, the compaction can't keep up.

- In B-trees each key exists in exactly one place in the index.

### Other Indexing Structures

- Key-value indexes are like primary key index.

- Secondary indexes can also be there. They are also key value pairs but they may not be unique. This can be represented in two ways:
    
    1. Postings list: Value->row mapping
    2. Making each key unique by appending row identifier.

**Storing values within index**:

- Key is the thing the queries search for and value can be actual row (clustered index) or reference to row (heap file)

- Heap file is common because it avoids duplicating data when multiple secondary indexes are present.

- While updating, the record can be easily overwritten in place, provided that the new value (size of row) is not larger than old one as if larger every next element has to move and index has to be updated.

- A forwarding pointer is used in this case.

- The extra hop from index to head file causes performance issues. 

- In MySQL's InnoDB engine, primary key is always clustered row and secondary indexes refer to primary key.

- A compromise between storing clustered index and references is known as *covering index (covers the query by having columns) or index with included columns*.

- Clustered and covering indexes can speedup reads but require additional storage.

**Multi-column Indexes**:

- For querying multiple columns of a table.

- Most common type is called a concatenated index, which combines several fields into one key.

- Concatenated index is useless if we want to query specific column in it.

- Multidimensional indexes are a more general way. Particulary important for geospatial data

```
SELECT * FROM restaurants WHERE latitude > 51.4946 AND latitude < 51.5079
 AND longitude > -0.1162 AND longitude < -0.1004;
```

- One option is to translate a two-dimenstional location to a single number using space-filling curve then use B-Trees. ALso known as R-trees.

**Full-text search and fuzzy indexes**:

- All indexes have exact data. They don't allow search of similar keys.

- Full text search engines allow this like synonyms, words within a certain edit distance.

- Lucene uses SSTables for its term dictionary. It requires a small in-memory index that tells queries at which offset in a sorted file they need to look for a key.

- In Lucene this index is a finite state automaton over the characters in the keys, similar to trie.

**Keep everything in memory**:

- Data needs to carefully written to disks for good performance but they are durable and lower in cost.

- But as RAM becomes cheaper it became feasible to keep small data in memory distributed across machines known as *in-memory databases*.

- In-memory stores like Memcached are intended for caching use only, where it's acceptable for data to be lost if a machine is restarted.

- Some stores aim for durability, which can be achieved by:
    1. Special hardware (Battery-powered RAM)
    2. Writing a log of changes to disk
    3. Writing periodic snapshots to disk
    4. Replicating in-memory state to other machines

- Redis and Couchbase provide weak durability by writing to disk async.

- The performance advantage of in-memory databases is not due to the fact that they don’t need to read from disk. 

- Even a disk-based storage engine may never need to read from disk if you have enough memory, because the operating system caches recently used disk blocks in mem.

- They can be faster because they can avoid the overheads of encoding in-memory data structures in a form that can be written to disk.

- In-memory database architecture could be extended to support datasets larger than available memory. This is called *anti-caching*.

- This works by evicting the LRU data from memory to disk and then bring back when needed.

## Transaction Processing or Analytics?

- A transaction needn't necessarily have ACID properties. Transaction processing just means allowing clients to make low-latency reads and writes as opposed to batch processing jobs.

- **Online transaction Processing (OLTP)**: Databases which are typically used to look up a small number of records by some key, using an index and then records are inserted/updated based on the user's input.

- **Online Analytic Processing (OLAP)**: Databases which are used for data analytics. Usually an analytic query needs to scan over a large number of records, only reading a few columns per record, and calculate aggregate statistics.

- **OLTP vs OLAP**:
    
    - Reads: Small no. of records per query vs aggregate over large.

    - Writes: Random access, low latency writes from user input vs Bulk import or event stream

    - Users: Customers vs Analysts
    
    - Data: Latest state vs History of events

    - Size: GB/TB vs TB/PB

- Initially same databases (like sql) were used for both use cases. But the analytics were run on seperate databases known as **data warehouse**.

### Data Warehousing

- An enterprice may have different transaction processing systems, each of which is complex and need people for maintaining and run autonomously from each other.

- These OLTP systems are expected to be highly available and process transactions with low latency.

- A **Data Warehouse** is a seperate database that analysts can query without affecting OLTP operations.

**Extract-Transform-Load (ETL)**:

- Data warehouse contains read only copy of OLTP databases and the data is extracted either periodically or a continuous stream.

- It is then transformed into an analyst-friendly schema, cleaned up and loaded into the data warehouse.

**The divergence between OLTP databases and data warehouses**:

- Data model of a data warehouse is mostly relational as SQL is good fit for analytic queries.

- The SQL query interface is same but the internals of the systems are different.

### Stars and Snowflakes: Schemas for Analytics

- Many data warehouses use a formulaic style data model known as **star schema** (also known as dimensional modeling).

**Fact and dimension tables**:

- At the centre we have *fact table*. Each row of fact table represents an event that occured at a particular time.

- Some of the columns in these fact table are attributes but other columns are foreign key references to other table called *dimension tables*. 

- Dimension tables represent the who, what, where, when, how and why of the event.

- Star schema: Fact table in the middle and dimension tables around it.

- Snowflake schema: The dimensions are further broken down into subdimensions.

- Snowflake schemas are more normalized than star schemas, but star schemas are simple.

## Column-Oriented Storage

- Fact tables have trillions of rows and may be 100 columns wide. But for a query we need only a small subset of the rows.

- In general OLTP databases, storage is laid out in a row-oriented fashion. Even if we have indexes on the queried columns, a row-oriented engine needs to load all rows each having 100s of columns from disk to memory and process them which takes a long time.

- **Column-oriented storage** store all the values from each column together. A query only needs to read and parse those columns that are used in the query.

### Column compression

- Depending on the data in the column, different compression techniques can be used. One technique which is majorly used is *bitmap encoding*.

**Bitmap encoding**:

- Number of distinct values in a column is small compared to the number of rows (only 10000 distinct products, 200 countries). So n distinct values can be turned into n seperate bitmaps.

- Each bitmap has one bit for each row. The bit is 1 if the row has that value else 0. If n is very large then there will lot of zeroes in most of the bitmaps, then the bitmaps can be run-length encoded.

- Bitmaps are very useful for OR and AND queries which are very common in data analytics.

**Memory Bandwidth and vectorized processing**:

Developers of analytical database worry about effectively using bandwidth from disk to main memory, main memory to CPU cache, avoiding branch misprediction and bubbles in CPU instruction processing pipeline and making use of single-instruction-multi-data (SIMD) instructions.

- Column-oriented storage layouts are also good for making efficient use of CPU cycles. Column compression allows more rows from a column to fit in the same amount of L1 cache.

- **Vectorized processing**: Operators such as bitwise AND and OR can be designed to operate on such chunks of compressed data directly.