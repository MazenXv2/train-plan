---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/4/primary-and-foreign-keys/","dg-note-properties":{}}
---


### Understanding Keys
A key in a database serves a dual purpose: it is both an index and a key. As an index, it physically copies the values of designated fields into a separate, optimized search structure—like a B-tree or hash table—that allows the database engine to locate rows much faster than scanning the entire table. As a key, it acts as a special tag or identifier for that field, enabling it to be used as a primary or foreign key to establish and enforce relationships between tables, thereby linking them together into a cohesive relational structure. Essentially, the index provides the performance benefit for quick lookups and joins, while the key provides the logical meaning and integrity that defines how tables relate to one another.

#### Primary Keys
A primary key is used to uniquely identify a record in a table. Unique identification for each record is required because there is no other way to find a record without the possibility of finding more than one record, if the unique identifier is not used.

#### Foreign Keys
Foreign keysare the copies of primary keys created into child tables to form the opposite side of the link in an inter-table relationship—establishing a relational database relation. Aforeign key defines the reference for each record in the child table, referencing back to the primary key in the parent table.