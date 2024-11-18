# Understanding Your Queries
### Important note
The way all relational databases handle queries is fundamentally the same across current products on the market. In this repository, I use MySQL as an example, but you can apply this knowledge to any database you work with.

### Brief introduction of Mysql

MySQL is an open-source relational database management system (RDBMS) that uses SQL to manage data. It’s known for being fast, reliable, and widely used in web applications. MySQL supports multiple storage engines, with InnoDB being the default. It’s free, scalable, and maintained by Oracle with a large developer community.

## Overview

In this repository, you will learn:

1. **Database Architecture**
- MySQL architecture
- Storage engines architecture

2. **Query Optimization**
- Misconceptions
- The execution plan
- Indexing and Partitioning  (Updating)



## 1. Database Architecture
### I. General Architecture
Before diving into the query optimization section, it’s important to understand the architecture of the database system. 

Most databse follow a structure similar to a three-tier architecture, which can be simplified as shown in this illustration.
![](resources/3tier.jpg)

The architecture consists of three layers:
- Client (Applications layer)
- Server (Logical layer)
- Storage (Physical layer)

For example, this is the architecture of MySQL:

![](resources/mysql_architecture.jpg)


### Client Layer
The client, or application layer, is straightforward to understand. It represents any application used to connect to and interact with the database server. This can be a web application, an SQL GUI, SQL CLI, Azure Studio, MySQL Workbench and more. In fact, you can’t directly interact with database without a client. For example, when you open a terminal and type ```mysql```, you’re using a query interface, which is also a type of client.

Here is an example of connecting to MySQL server using ```mysql```

```zsh
(base) an@Ans-MacBook-Pro ~ % mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.1.0 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

The client connects to the server through ```connectors```, or you can also call them APIs. There are various connectors available, such as JDBC, ODBC, C++, C, and Python, depending on the application you’re using.

### Server Layer
The server layer consists of many components, but we can simplify it into two sub-layers: the service layer and the storage layer.

The service layer includes everything above the storage engine in the architecture. There are two main components to focus on here: the cache and buffer area, and the query processing components.

**Query Processing Components**: These include the parser, query resolver, query planner, query optimizer, and query executor. These components may have different names in different DBMS, but they all serve a same purpose.

- Parser: When a query is received, the parser breaks it down into small tokens and checks the syntax.
- Query Resolver: This component checks the user’s privileges on the schema, table, and database.
- Query Planner: The planner generates an initial plan for executing the query, typically based on table structure and available indexes.
- Query Optimizer: The optimizer refines the initial query plan to find the most efficient execution path.
- Query Executor: Finally, the executor executes the optimized query plan, retrieves the data, and returns the results to the cache and buffer.

**Cache and Buffer**: This section holds all information related to the session, including queries, changes, and the storage engine’s cache and buffer. So if you run a query twice, the second time will run faster since all data is already stored here.

### II. Storage engine

The storage engine is responsible for interacting with the underlying data at the physical server level. Its job is to manage data files, table spaces and logs.

One unique feature of MySQL, compared to other databases, is its modular storage engine architecture. MySQL developers refer to this as its **"pluggable storage engines"**. This modularity offers significant benefits, especially for specific applications, such as data warehousing, transaction processing, or high-availability situations. It allows users to take advantage of a set of interfaces and services that remain independent of any single storage engine.

Storage engine in-memory architecture:

![](resources/storage_engine_architecture.jpg)

The graph above shows a simplified version of the storage engine architecture. I’ve simplified it to emphasize one key point: the page is the smallest data unit that a storage engine works with. This concept is similar across many database systems. For example, in Oracle, these units are called blocks, while in MySQL and MS SQL, they’re called pages.

When discussing how a database searches for data in storage, it’s important to note that **database doesn’t work with rows directly, but with pages, which contains data rows.**

These pages are stored inside a tablespace. What is a tablespace? It’s exactly what the name suggests: a space for a table. This is a logical concept. By default, each table has its own tablespace, and in MySQL, you can find these in your hardware storage as .ibd files (InnoDB data files).

```
(base) an@Ans-MacBook-Pro ~ % cd /usr/local/mysql/data
(base) an@Ans-MacBook-Pro data % ls
#ib_16384_0.dblwr       mysql.ibd
#ib_16384_1.dblwr       mysqld.local.err
#innodb_redo            performance_schema
#innodb_temp            bprivate_key.pem
Database1               public_key.pem
Database2               server-cert.pem
Practice                server-key.pem
ib_buffer_pool          sys
ibdata1                 undo_001
ibtmp1                  undo_002
mysql
(base) an@Ans-MacBook-Pro data % cd Database1
(base) an@Ans-MacBook-Pro Database1 % ls
employees.ibd		youtube.ibd
```

Inside each tablespace, there are pages. By default:
- Each page is 8 KB (the same for every other database system).
- A .idb data file containing one page is 115 KB in size.

To summarize:
- A tablespace is a logical structure used by databases to manage data.
- A tablespace can contain many data files.
- Each data file consists of pages.
- A data file can contain data of multiple tables, but by default of MYSQL, each tablespace has one data file, and that data file contains the data for only one table.

### Important Note  
You may wonder why understanding architecture is important for query optimization?  

For query processing components, in 99% of cases, you only need to know that they exist and help execute your query. There’s not much you can do to interfere with them. However, one key thing to remember is that the database does not understand queries the same way we do.  

For example, these two queries:  

```sql
SELECT * FROM employees WHERE salary >20000;
```
and
```sql
SELECT * FROM employees 
WHERE salary >20000;
```
will be understood as different queries by the database. Although they are exactly the same, the parser interprets them differently, leading to two different queries being stored in the cache.

Despite this, these queries still share the same ```Execution Plan``` (which we will discuss in the next section). This means the data in the cache for the two queries are one; and whatever version of above query run first will help the other one run faster nextime. For some versions of database, for example MySQL 8.0, query_cache was removed, so I guess it is not much of the deal.

The next thing to pay attention to is the cache and buffer, as they directly affect your database performance. You can configure the size and other attributes of the cache and buffer in the configuration file.

The most important component in the server layer is the storage engine. The key focus should be on how the storage engine searches for data on the disk, how data is read and written and how it interact with the buffer pool.

## 2. Query Optimization  

### I. Misconceptions
Before diving into this section, I want to address some common misconceptions about query optimization:
- **Misconception number 1: Adding `WHERE` conditions or `LIMIT` makes queries faster because they return less data.**
- **Misconception number 2: Never use SELECT * because it is always slow**
- **Misconception number 3: Using indexes always improves query performance.**  

Both of these assumptions are false. While these methods can sometimes make your query run faster, it’s often just coincidental.  

By exploring these misconceptions, we will deepen our understanding of how queries actually work.  


#### Understanding Query Execution  

We’ve already covered how storage engines store data in the Architecture section. Now, let’s go deeper to truly understand what happens when you retrieve data from a database.  

When you run a query, you might expect the server to search through the data files stored on disk for the relevant rows and load them into memory. However, that’s not how it works.  

Remember, data is stored in **pages**—the smallest piece of data that database handles. When you run a query, the database doesn’t retrieve individual rows; instead, it loads the **entire page** containing the data into memory. From there, the storage engine filters the relevant rows.  
 
In practice, even if you’re retrieving only a few rows, the execution time can still be very long. This often happens when your data is **spread across many pages**. The storage engine needs to scan all these pages and load the relevant ones into memory, which increases the runtime. 

---

#### Example
For example, consider this query:
```sql
SELECT * FROM employees 
WHERE salary >40000 AND age < 30;
```

Let’s assume the ```employees``` table has 10,000 data pages.

**This is how the Storage Engine Processes This Query**

1. Scanning Data Pages:
The engine will scan all of the ```employees``` table’s data pages to identify the pages containing valid rows.

2. Scenario 1: Valid data is concentrated in 1,000 pages.
In this case, the query will run faster because only 1,000 pages need to be loaded into memory, reducing the I/O operations.

3. Scenario 2: Valid data is spread across all 10,000 pages.
Here, the query won’t benefit from the ```WHERE``` condition since the engine still needs to load all 10,000 pages to identify the relevant data.

**Will This Query Run Faster Without the WHERE Clause?**

Surprisingly, yes—removing the WHERE clause can make the query run faster in certain scenarios.

Here’s why: Without the WHERE clause, the engine directly loads all the pages related to the table into memory. In contrast, when you include the WHERE clause, the engine must first evaluate and filter rows during the initial page scan, adding an extra processing step. This additional filtering increases the query runtime.

```
19:20:28	SELECT * FROM employees  WHERE salary >40000 AND age < 30	315729 row(s) returned	0.0027 sec / 0.288 sec

19:20:45	SELECT * FROM employees	1185550 row(s) returned	0.0010 sec / 0.342 sec
```

Notice how it takes less than half the time to return a million rows compared to just 315,729 rows. **Less data returned does not always mean a faster query.**

Similar to using `LIMIT`, here's an example:  

This query:
``` sql
SELECT * FROM employees 
WHERE age < 30 ORDER BY age DESC LIMIT 200;
```
Takes more time to run compared to this query:

``` sql
SELECT * FROM employees 
WHERE age < 30;
```
```
19:34:25	SELECT * FROM employees  WHERE age < 30 ORDER BY age DESC LIMIT 200	200 row(s) returned	0.259 sec / 0.000073 sec

19:34:33	SELECT * FROM employees  WHERE age < 30	394772 row(s) returned	0.0021 sec / 0.259 sec
```
Notice how it takes significantly more time to return a much smaller number of records?

The difference lies in the ```ORDER BY``` clause. When the query includes ORDER BY, the database must:

1. Retrieve all the relevant data from the employees table.
2. Sort the data in descending order.
3. Finally, filter out the unnecessary rows to satisfy the ```LIMIT``` condition.

These two additional steps—sorting and filtering—make the query with ```ORDER BY``` and ```LIMIT``` take longer, despite returning fewer rows. Even though the result set is smaller, both queries use the same number of data pages stored in cache. The extra processing steps in the first query result in a longer runtime.

### II. Execution Plan

Remember the query planner and query optimizer? Their job is to determine the **best way** for the storage engine to retrieve data. All RDBMs form a execution plan for a query before it actually execute it.


#### What Does "Best" Mean?  

The "best" execution plan is the one with the **minimal I/O**. While databases consider many criteria and parameters to assess the best plan, I/O remains the most critical factor. These parameters are combined to calculate a value called the **cost**. The execution plan with the **lowest cost** is consider the **best** and chosen by the database.

---

#### How Can We See the Execution Plan?  

In MySQL, by using the `EXPLAIN` keyword, you can view the execution plan for each query. This allows you to understand how the database processes your query and how it determines the most efficient plan. Different databases have their different commands for showing the execution plan, but they all contain the same key informations.

Here's an example of using `EXPLAIN` in MySQL to understand the execution plan for a query, it gives the astimated value for parameters.

```bash
mysql> USE database1
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> EXPLAIN SELECT * FROM employees 
    -> WHERE salary >40000 AND age < 30;
+----+-------------+-----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1180959 |    11.11 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> 
```
#### Key Parameters to Pay Attention To
1. table: Which table(s) are used in the query.
2. partitions: If any partitions are used.
3. type: How the table is scanned. Common values:
- ALL: Full table scan.
- ref: Uses an index.
4. possible_keys: Possible indexes that could be used.
5. key: Current index used
6. rows: Estimated number of rows scanned.

To get actual values, use ```EXPLAIN ANALYZE```:

```bash
mysql> EXPLAIN ANALYZE SELECT * FROM employees 
    -> WHERE salary >40000 AND age < 30;

EXPLAIN
-> Filter: ((employees.salary > 40000.00) and (employees.age < 30))  (cost=119193 rows=131191) (actual time=0.33..277 rows=315729 loops=1)
-> Table scan on employees  (cost=119193 rows=1.18e+6) (actual time=0.315..206 rows=1.19e+6 loops=1)

1 row in set (0.30 sec)
```

**Explanation**
- Filter: Shows the filtering conditions applied, along with the estimated and actual number of rows processed (rows) and execution time (actual time).
- Table scan: Indicates the type of scan performed on the table, along with estimated (cost) and actual row counts.

```EXPLAIN ANALYZE``` provides more precise insights by including actual execution times and row counts, making it extremely helpful for understanding and optimizing queries.

Here, you will notice the **cost** (cost=119193) that I mentioned in earlier section. The database chooses the execution plan based on this value. You can also use **cost** to compare your queries and determine which one performs the best by having the lowest cost.

What Determines Cost?
The cost of an execution plan is influenced by several factors, including:

- **I/O Operations**:
How many data pages need to be read or writen from disk. This is often the dominant factor in cost calculation.
- **Indexes**:
Whether an index can be used to skip scanning unnecessary data pages. Indexes reduce I/O and, consequently, cost.
- **Joins and Sorting**:
Queries with complex joins or sorts have higher computational costs, reflected in the cost value.
- **Buffer Pool Efficiency**:
Data already cached in memory is faster to access, reducing cost compared to fetching from disk.

In practice, a lower cost typically means the query will require fewer resources (such as I/O operations and CPU time), which usually leads to better performance, often reflected in a shorter runtime. 

There are many ways to reduce cost. The most common one is using indexes. Proper indexing can greatly reduce the number of rows and pages the database needs to scan, speeding up query execution. Other strategies include optimizing joins, reducing the number of unnecessary columns selected (for some database like Reddis or Snowflake), and avoiding full table scans. In some cases, adjusting the query cache or buffer pool settings can also help improve performance.

To conclude this part, you can now address misconception number 2: *"Never use SELECT * because it is always slow."* 

*Hint: Its depends on the query's execution plan and the data distribution. You can look at the cost of each query to compare their performance.*

### III. Indexing and Partitioning
Using an index is an excellent way to enhance query performance. Let’s look at the following example:
```sql
EXPLAIN SELECT name FROM employees 
WHERE salary >79000;
```
Without index, the result is:
```
+----+-------------+-----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1180959 |    33.33 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
With index, the result is:

```
+----+-------------+-----------+------------+-------+---------------+------------+---------+------+-------+----------+-----------------------+
| id | select_type | table     | partitions | type  | possible_keys | key        | key_len | ref  | rows  | filtered | Extra                 |
+----+-------------+-----------+------------+-------+---------------+------------+---------+------+-------+----------+-----------------------+
|  1 | SIMPLE      | employees | NULL       | range | salary_idx    | salary_idx | 6       | NULL | 43946 |   100.00 | Using index condition |
+----+-------------+-----------+------------+-------+---------------+------------+---------+------+-------+----------+-----------------------+
```
As you can see, with the index, the number of rows scanned is significantly lower, and the ```WHERE``` condition is no longer needed because the index itself efficiently filters the relevant rows.

Now, look at the cost:
```
-> Index range scan on employees using salary_idx over (79000.00 < salary), with index condition: (employees.salary > 79000.00)  (cost=19776 rows=43946) (actual time=0.0933..58.5 rows=23613 loops=1)
```
The cost is now just 19776, which is much lower compared to the full table scan cost of 119193.

Index works by creating a smaller, sorted data structure (typically a B-tree or hash) that references the locations of rows in the main table. Instead of scanning the entire table to find rows that match the `WHERE` condition, the database can use the index to quickly locate relevant rows.

For example, if you create an index on the `salary` column, MySQL creates a sorted structure of `salary` values with pointers to the rows in the table where those values are stored. When you execute a query with a condition like `salary > 79000`, the database scans only the relevant portion of the index, significantly reducing the number of rows and pages it needs to read from disk.

However, indexes come with a trade-off. They put additional pressure on your hardware storage, as extra space is required to store the index structures. 

Coming back to misconception number 3: *Using indexes always improves query performance.*

One thing we can immediately deduce from the information above is that maintaining indexes can also slow down write operations (inserts, updates, deletes). Whenever data changes, the indexes need to be updated as well, which adds overhead. This is because the index structures need to reflect changes in the underlying data, and each write operation will trigger an update to the associated indexes. In high-volume write environments, this can lead to significant performance degradation.

Sometimes, an index might not work as expected. Usually, this happens when an index is inefficient because it’s not selective enough, meaning it doesn’t filter out many rows. For example, if you have an index on a column where most of the rows have the same value, the index won’t narrow down the search significantly. In such cases, the query planner might decide to use a full table scan instead of the index.

To create an effective index, it’s important to choose columns with high uniqueness, also known as high ```cardinality```.

However, what if you don’t have any columns with high uniqueness to create an index? In this case, one option is to create a composite index—an index that involves multiple columns. This approach helps increase the uniqueness of the index by combining values from multiple columns, which can make the index more selective and improve query performance.

Example:
We create this index using age column, and run the ```EXPLAIN``` query
```sql
CREATE INDEX age_idx ON employees (age);
EXPLAIN SELECT name FROM employees 
WHERE age = 25;
```
we get the result like this:
```
+----+-------------+-----------+------------+------+---------------+---------+---------+-------+-------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys | key     | key_len | ref   | rows  | filtered | Extra |
+----+-------------+-----------+------------+------+---------------+---------+---------+-------+-------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | ref  | age_idx       | age_idx | 5       | const | 79790 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+---------------+---------+---------+-------+-------+----------+-------+
```
Then we add a composite index using two column age and salary columns and run the query again
```sql
CREATE INDEX idx_age_salary ON employees (age, salary);
EXPLAIN SELECT name FROM employees 
WHERE age = 25;
```
the result now looks like this:
```
+----+-------------+-----------+------------+------+------------------------+----------------+---------+-------+-------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys          | key            | key_len | ref   | rows  | filtered | Extra |
+----+-------------+-----------+------------+------+------------------------+----------------+---------+-------+-------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | ref  | age_idx,idx_age_salary | idx_age_salary | 5       | const | 35952 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+------------------------+----------------+---------+-------+-------+----------+-------+
```
Notice how the database now chooses the composite index as the index used in the execution plan, and the number of rows scanned decreases significantly (79790 vs 35952). This is because the composite index, which includes multiple columns, provides a more selective filter than individual indexes on each column.

One thing you need to pay attention when using a composite index is that it has a rule to follow.

Now let's look at these three queries and their result.

Query 1:
```sql
EXPLAIN SELECT name FROM employees 
WHERE age = 25 AND salary >50000;
```
```
+----+-------------+-----------+------------+-------+----------------+----------------+---------+------+-------+----------+-----------------------+
| id | select_type | table     | partitions | type  | possible_keys  | key            | key_len | ref  | rows  | filtered | Extra                 |
+----+-------------+-----------+------------+-------+----------------+----------------+---------+------+-------+----------+-----------------------+
|  1 | SIMPLE      | employees | NULL       | range | idx_age_salary | idx_age_salary | 11      | NULL | 46898 |   100.00 | Using index condition |
+----+-------------+-----------+------------+-------+----------------+----------------+---------+------+-------+----------+-----------------------+
```
Query 2:
```sql
EXPLAIN SELECT name FROM employees 
WHERE salary >50000;
```
```
+----+-------------+-----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1180959 |    33.33 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
```
Query 3:
```sql
EXPLAIN SELECT name FROM employees 
WHERE age = 25;
```
```
+----+-------------+-----------+------------+------+----------------+----------------+---------+-------+-------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys  | key            | key_len | ref   | rows  | filtered | Extra |
+----+-------------+-----------+------------+------+----------------+----------------+---------+-------+-------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | ref  | idx_age_salary | idx_age_salary | 5       | const | 75952 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+----------------+----------------+---------+-------+-------+----------+-------+
```

As you can see only query 1 and 3 are using the index, while query 2 is using full table scan. In a composite index, the order of the columns in the index is very important. The database uses the index in a left-to-right order. This means the query must use the leading column(s) in the same order as they appear in the index for the index to be fully utilized. This is called the ```Leftmost Prefix Rule```.