---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/4/indexes/","dg-note-properties":{}}
---


### Understanding Indexes
Indexes are not really part and parcel of the relational database model itself; however, indexes are so important to performance and overall database usability that they simply have to be introduced without going into the nitty-gritty of how each different type of index functions internally. It is important to understand the fundamentals of indexes and their different types and attributes to get a basic understanding as to why exactly indexing is so important for relational databases in general.


### What Is an Index?
An index in a database is essentially a physical copy of a small portion of a table—typically a single, short-length field—stored separately from the table itself, often in a different file or disk area. Its purpose is to provide a fast, efficient lookup mechanism, much like the index at the back of a book or a table of contents: instead of scanning every row in the table to find a specific value, the database engine can first search the much smaller index to quickly locate the exact location of the data, using page references or row pointers to jump directly to the relevant records. When a query is executed, a component called the Optimizer decides whether to scan the entire table or use the index, based on which approach is faster. By physically separating the index from the table data, the database can dramatically speed up searches and joins, preventing the need to read the entire table every time—which would be as impractical as reading an entire book cover-to-cover just to look up a single term.

#### Following are some things to be avoided when indexing:
- Creating too many indexes on a table can severely degrade the performance of data modification operations—such as INSERT, UPDATE, and DELETE—because every time a row in the table changes, every single index associated with that table must also be updated to reflect the change. This means the database engine has to write not only to the table itself but also to each separate index structure, multiplying the number of physical disk writes and processing overhead required for a single change.
- Indexing too many fields not only confuses the query optimizer by presenting it with excessive, often redundant options for executing a query, but it also physically bloats the index structures themselves. Since an index is meant to be a small, lightweight copy of the table for fast searches, loading it with multiple columns makes it nearly as large as the table itself, defeating the very purpose of an index.


### Foreign Key Indexing
In database design, a foreign key column in a child table does **not** automatically receive an index from the database because foreign keys are allowed to contain duplicate values and can be NULL, so the system only creates automatic indexes on primary keys to enforce their uniqueness. This means that **you, as the designer, must manually create indexes on foreign key columns** using a `CREATE INDEX` command to ensure efficient joins between parent and child tables. Without such manual indexing, queries that link an author to their publications would force the database to perform a slow full table scan of the child table every time, severely degrading performance as data grows. Thus, manually indexing foreign keys is a critical, non-optional step that complements the automatic indexing of primary keys, ensuring both referential integrity and fast query response in relational databases.