![](./images/database.png)

# Introduction

Today's systems are becoming more complex and data-intensive than ever before. The databases are getting bigger and bigger and become a challenge for applications, it will have problems with storage, processing time. More complex requirements such as statistics, reports or supporting user interactions on large amounts of data. To ensure fast application performance, in-time response allows for more advanced processing techniques. In this article we will give some solutions to help you find a solution to your problems

# Problem

We have more than 500M records data and users can filter or sort randomly with multiple complex aggregation fields so making a regular query with a partition table and indexed fields won’t work. Timeout when querying big data with multiple complex aggregation fields.

# Solution

As we know out side have a lot solution to do this but will depend on some constraint such as: price, time, infrastructure and others. In this article i will review the more recent solution and providing some use case

- Level 1: From big database we will extra data and move it to many smaller tables we call cache table, we also create index and create partition for it to make query quicker. To create cache table quicker we use multiple thread
- Level 2: From the beginner one database will lead the read performance so when database become bigger, we can consider to create slave database for read and backup
- Level 3: With the data bigger every day, we need an process like ETL, Hadoop ecosystem

Almost common database, solution level 1 can help us handle query speed

## Level 1: Cache table, partition and multithread processing

We can run parallel processes to build cached data row by row because queries with where clause and without order will be fast and we can run similar base server resources.

### Detail

- When running aggregation in partition tables it will take a very long time to scan all tables
  ⇒ if we have a **WHERE** clause with a partition field it will help the system take some tables to query and reduce query time
- **ORDER BY** takes a long time because it has to wait for all aggregation fields done and then order it
  ⇒ skip order aggregation fields will reduce query time
- Making a query to do everything in our report is hopeless so we need to run parallel to get one-by-one independent row results and then return it async to the user

### Solution build the cached table by getting row by row result

In normal way, we use `VIEW` or `MATERIALIZED VIEW` to cache the query but with large data we will have timeout or wait so long time to return all records result.

Query with **WHERE** clause with a partition field is fast but we have to wait query by query.

So if we can combine the speed from **WHERE** clause query and run multiple query at the same time then insert into one cached table, we can get result set faster and make use of application resources.

Example Elixir code for this

```elixir
def update_filtered_result_report(index_id, params) do
    entries = 1..99

    entries
    |> Task.async_stream(
      fn entry ->
        entry
        |> DemoTradeResultsService.get_summary_by_entry() ## This function will return list of records after group by
        |> Enum.chunk_every(1000)
        |> Enum.map(fn chunk ->
          create_all(index_id, chunk)
        end)

        Logger.info("Done entry #{entry} for filtered table #{index_id}")

        length(result)
      end,
      max_concurrency: 64,
      timeout: :infinity
    )
    |> Enum.reduce(0, fn {:ok, total}, acc -> total + acc end)

    Logger.info("Done to build cached filter table #{index_id}")
  end

  def get_summary_by_entry(entry) do
    default_query()
    |> where([t], t.entry == ^entry)
    |> Repo.all(timeout: :infinity)
  end

  def default_query() do
    from(d in DemoTradeResults,
      group_by: [d.entry, d.target, d.stoploss, d.rr],
      select: %{
        target: d.target,
        entry: d.entry,
        stoploss: d.stoploss,
        rr: d.rr,
        win_rate: fragment("cast(round(cast(cast(sum(win) * 100 as float) / cast(count(price_structure_id) as float) as numeric), 2) as float)"),
        lose_rate: fragment("cast(round(cast(cast(sum(lose) * 100 as float) / cast(count(price_structure_id) as float) as numeric), 2) as float)"),
        na_rate: fragment("cast(round(cast(cast(sum(na) * 100 as float) / cast(count(price_structure_id) as float) as numeric), 2) as float)"),
        win_count: sum(d.win),
        lose_count: sum(d.lose),
        na_count: sum(d.na),
        price_structure_count: count(d.price_structure_id),
      }
    )
  end
```

**Pure SQL**

```sql
SELECT
  target,
  entry,
  stoploss,
  rr,
  cast(round(cast(cast(sum(win) * 100 as float) / cast(count(price_structure_id) as float) as numeric), 2) as float) AS win_rate,
  cast(round(cast(cast(sum(lose) * 100 as float) / cast(count(price_structure_id) as float) as numeric), 2) as float) AS lose_rate,
  cast(round(cast(cast(sum(na) * 100 as float) / cast(count(price_structure_id) as float) as numeric), 2) as float) AS na_rate,
  sum(win) AS win_count,
  sum(lose) AS lose_count,
  sum(na) AS na_count,
  count(price_structure_id) AS price_structure_count
FROM
  backtest_trade_results
WHERE
	entry = 1, target = 0, stoploss = 99
GROUP BY
  entry,
  target,
  stoploss,
  rr
```

**How to build fully cached table data**

- We need to define a list of values in WHERE CLAUSE so we can save time to get it when running the query
  - Can define a static list in code if it’s static
  - Can get distinct data from the table and cached it somewhere
- In this example, WHERE CLAUSE contains `entry`, `target`, `stoploss` so we need to define these lists.
  - `entry`: [1..99]
  - `target`: [0..98]
  - `stoploss`: [1..100]
- After defining list to run all rows we can run a loop and run it parallel to fill up the cached table

Return data while building cached table and it takes a long time to build

What if the user queries new filter data and we haven’t had a cached table for this, how can we return exact data to a user in an acceptable time?

1. We need to define the data event and can build ASAP after receiving new data to update cached tables which are used regularly. In this example, we have to rebuild cached table for all price structures
2. We need to order values in the `WHERE CLAUSE` list base on the order by and where clause from the user filter and then run multiple processes to get paginated data. Example
   1. Based on the requirement
      `Get all entries to have win rate from **60%** to **80%** from backtest result table (table has more than 500M records) and order by rr (reward & risk) desc`
      We need to analyze to define the order value in this query

#### Solution Indexing

#### Solution Optimize query

**ORM Query**

**Raw query**

- Do not use `SELECT *` if you do not need all the data in the queried tables.
- Write the appropriate query to avoid calling multiple queries for 1 processing logic (for, loop query)
- Split large queries into moderately sized queries (eg you can limit 10000 records at a time)
- Remove join, remove join in query instead join in application (avoid resource locking, more efficient caching, more efficient data distribution)
- Limited use of DISTINCT because it slows down the query

You can test with the real database:

#### 1. Optimize index

#### 2. Optimize query

Before optimize database and query

```
SELECT
 miner_name,
  MAX(fan_percentage)
FROM miner_data
WHERE miner_name IN
  (SELECT DISTINCT "name"
   FROM miners)
GROUP BY 1
ORDER BY 1;
 miner_name |        max
------------+-------------------
 Bronze     |  94.9998652175735
 Default    | 94.99994839358486
 Diamond    | 94.99999006095052
 Gold       |  94.9998083591985
 Platinum   | 94.99982552531682
 Silver     | 94.99996029210493
Time: 9173.750 ms (00:09.174)
```

Optimize:

```
SELECT
  DISTINCT ON (miner_name) miner_name,
  MAX(fan_percentage)
FROM miner_data
GROUP BY miner_name;
miner_name |        max
------------+-------------------
 Bronze     |  94.9998652175735
 Default    | 94.99994839358486
 Diamond    | 94.99999006095052
 Gold       |  94.9998083591985
 Platinum   | 94.99982552531682
 Silver     | 94.99996029210493
Time: 2794.690 ms (00:02.795)
```

Before optimize database and query

By writing query in a smarter way, we saved ourselves time.

Pre-optimization: 9173.750 ms

Post-optimization: 2794.690 m

### 3. Optimize Partition

### 4. Database tuning

SQL query which we execute, has to load the data into memory to perform any operation. It has to hold the data in memory, till the execution completes.

If the memory is full, data will be spilled to disk which causes the transactions to run slow, because disk I/O is time taking task.

So, a good amount of memory enough to fit the data that is being processes as part of SQL is needed.

So you need increase server resource as much as you can base on your scale and acceptable cost to prevent out of memory or out of CPU utilization and increase query process speed

We suggest to check the benchmark chart and config to your database server to make sure `CPU utilization` and `Memory usage` are under 70%, so it will make query speed is stable

## Level 2: Database architecture

For large systems, more and more data makes 1 database or 1 piece of hardware will not be able to serve many users or more and more data. To optimize the query and search for data, we have many techniques that can be applied to handle the above problem.

### Replication

For systems that serve many users with many different tasks such as reading, writing, updating data ... to optimize data reading we can install replica databases so that clients can optimize reading on database replicas and optimize data writing on database master while ensuring performance and data integrity for clients.

We talk about how replica works, when a client sends enough data to database A, databases B and C are 2 replicas will also read the corresponding changes to update. Database A can actively send information through B and C and wait for confirmation when it is called synchronous, it ensures the data on all servers is the same, but it takes an extra time to wait for data. synchronized and send the response back to the client. If there is a replica machine that does not respond, all data will be rolled back.

The second way is that the client sends data to server A and server A will respond to the successful result to the client immediately, then the changed data will be synchronized to database servers B and C. This option is asynchronous when the job copies data to other servers when network problems occur.

### Partitioning

For a large data table we can split that table according to some criteria into many smaller data tables based on some criteria such as subdividing a table by day, month or year, by region, by region. IP range. We can understand this is the process of dividing tables, indexes, views at a low level, each partition as smaller compartments.

This is a popular method in small and medium projects, it allows programmers to divide a large table of data into smaller intervals according to a criterion to speed up the query for a certain data range.

**Attention when creating partition:**

- Partition key is the column that frequently appears in the search condition
- The column has many different values but not too many
- The values in the column are evenly distributed
- Do not choose columns of type varchar whose data can be anything

**Chọn kiểu partition:**

- List partition: The table will be divided into partitions based on the values of the partition key column, these values are finite and discrete (discrete value).

```sql
CREATE TABLE sales_list(
    salesman_id NUMBER(5),
    salesman_name VARCHAR2(30),
    sales_state VARCHAR2(20),
    sales_amount NUMBER(10),
    sales_date DATE)
PARTITION BY LIST(sales_state)
(
    PARTITION sales_west VALUES ('California', 'Hawaii'),
    PARTITION sales_east VALUES ('New York', 'Virginia', 'Florida'),
    PARTITION sales_central VALUES ('Texas', 'Illinois'),
    PARTITION sales_other VALUES (DEFAULT)
);
```

- Range partition: The table will be divided into partitions based on the range of values of the partition key column.

```sql
create table order_details (
    order_id number,
    order_date date)
partition by range (order_date)(
    partition p_jan values less than (to_date('01-FEB-2022','DD-MON-YYYY')),
    partition p_feb values less than (to_date('01-MAR-2022','DD-MON-YYYY')),
    partition p_mar values less than (to_date('01-APR-2022','DD-MON-YYYY')),
    partition p_2009 values less than (MAXVALUE)
);
```

- \***\*Hash partition:\*\*** Streams of data will be randomly distributed into partitions, using a hash value column partition key function. Every time new data is available, the hash value will be calculated and decide to which part the data should belong. With the Hash partition type, the partitions will have the same data

```sql
CREATE TABLE employees (
     empno NUMBER(4),
     ename VARCHAR2(30),
     sal NUMBER
)
PARTITION BY HASH (empno) (
     PARTITION h1 TABLESPACE t1,
     PARTITION h2 TABLESPACE t2,
     PARTITION h3 TABLESPACE t3,
     PARTITION h4 TABLESPACE t4
);
```

Partition theo row hay column

### Clustering

Clustering is a group of nodes that store the same database schema on the same database software with some form of data exchange between servers. From outside the cluster, servers are viewed as a single unit containing a data union that is spread across the nodes in the cluster. When a client accesses a cluster, the request is eventually routed to a single node for read and write.

This setting makes reading large amounts of data guaranteed to be safe.

**Advantage:**

Load balancing: Requests are distributed and moderated across nodes

- High availability
- Data redundancy: DB nodes are synchronized, in case there is a problem in one node, data can still be accessed at another node.
- Scalability: Easily upgrade, add equipment and fix problems.

![Cluster](./images/cluster.png)

**Disadvantage:**

- Operating costs

### **Sharding**

Split a large data table horizontally. A table containing 100 million rows can be divided into multiple tables containing 1 million rows each. Each table due to the split will be placed into a separate database/server. Sharding is done to distribute the load and improve the access speed. Facebook /Twitter is using this architecture.

Sharding will be suitable for super large data, it is a scalable database architecture, by splitting a larger table you can store new blocks of data called logical fragments. Sharding can achieve horizontal scalability and improve performance in a number of ways:

- parallel processing, leveraging computing resources on the cluster for every query
- Individual pieces are smaller than the entire logic table, each machine has to scan fewer rows when answering a query.

**Advantage:**

- Expand the database horizontally
- Speed up query response time
- Minimize the impact if there is 1 database down

**Disadvantage:**

- There is a risk of data loss or table corruption
- Complex in management and access
- Possible segmental imbalance

**Classification of sharding architectures:**

- Key based sharding
  - Use table keys to determine which shard to save data, Prevent data hot spots, no need to maintain a map of where data is stored
  - Difficulty adding new shards, data imbalance between shards
- Range base sharding
  - Based on the range of a value of a certain data column
  - Can be unevenly distributed leading to data hotspots
- Directory based sharding (List)
  - Need to create and maintain lookup table using segment key
  - Each key is tied to its own specific segment
- Vertical sharding
  - Split table into smaller table with different number of columns data
  - Small tables can be stored in different databases, different machine
- Horizontal sharding
  - Split into smaller table by the row of the table
  - Small tables can be stored in different databases, different machine

### [Database comparison](./database-system.md)

## Level 3: More trending solutions for Big data

Input: We have a dataset of e-commerce with information about product, price, category … we need to process this data and generate a report with some criteria.

### **Props:**

Database:

Language:

### **Cons:**

### System diagram

### Implementation

## Compare

**Apache Spark**

Apache spark is designed to help process and analyze terabytes of data

Apache spark is a data processing framework that provides an interface for programming parallel computing clusters with fault tolerance. Advantages of distributed computing capabilities

The components:

- **Spark core**: core component, connecting other components. Calculation and processing in memory, and reference to data stored in external storage systems.
- **Spark Streaming**: is a plugin that helps Apache spark respond to real-time or near-real-time processing requests. Spark Streaming breaks down the processing stream into a continuous sequence of microbatches which can then be manipulated using the Apache Spark API. In this way, the code in the batch and streaming processes can be reused, run, and run. on the same framework, thus reducing costs for both developers and operators.

![Apache Spark](./images/apache-spark.png)

- **Spark SQL**: Spark SQL focuses on structured data processing, using a data frame approach borrowed from the R and Python languages (in Pandas). As the name suggests, Spark SQL also provides an interface with SQL syntax to query data, bringing the power of Apache Spark to data analysts and developers alike.
- **MLlib** (**\***Machine Learning Library)\*_\*\* : MLlib is a distributed machine learning platform on top of Spark with a distributed memory-based architecture. According to some comparisons, Spark MLlib is 9 times faster than the equivalent Hadoop library Apache Mahout._
- **GraphX**: Spark GraphX comes with a selection of distributed algorithms for dealing with graph structure. These algorithms use Spark Core's RDD approach to data modeling; The GraphFrames package allows you to perform graph processing on data frames, including taking advantage of the Catalyst optimizer for graph queries.
  Map reduce

**Hadoop ecosystem**

- Apache kafka
- Apache hadoop ⇒ Referral qua Brain.d.foundation.

# Conclusion

While these are some basic optimization techniques, they can bear very big fruit. Also, although these techniques are simple, it is not always easy to:

- know how to optimize query
- create a sufficient and valid number of database indexes without creating huge amounts of data on disk — thereby perhaps doing a counter effect and encouraging the database to search in the wrong way