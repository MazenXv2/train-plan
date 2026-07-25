---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/4/entity-relationship-diagrams-er-ds/","dg-note-properties":{}}
---

### Representing Relationships in an ERD

#### One-to-One
One-to-one relationships are often created to remove frequently NULL valued fields from a table. They are hopefully rare in relational database models, unless in exceptional circumstances because the price of storage space is cheap. One-to-one relationships are typical of 4th Normal Form transformations (see Chapter 4) where potentially NULL valued fields are removed from the parent table, possibly saving storage space. Not only is storage space cheap in modern times, but variable-length records in most relational databases tend to make this type of normalization bad for performance. SQL code joins get bigger as more tables are created. SQL code with more tables in joins can cause serious performance issues. Bad database performance means slow applications and unhappy users looking to pay your competitors what they pay you.

#### One-to-Many
One-to-many relationships are extremely common in the relational database model between tables.

#### Many-to-Many
Amany-to-many relationship means that for every one record in one table there are many possible records in another related table, and visa versa (for both tables). The classic example of a many-to-many relation ship is many students enrolled in many courses at a university. The implication is that every student is registered for many courses and every course has many students registered. The result is a many-to many relationship between students and courses. This is not a problem as it stands; however, if an appli cation or end-user must find an individual course taken by an individual student, a uniquely identifying table is required. Note that this new table is required only if unique items are needed by end-users or an application.


### Identifying and Non-Identifying Relationships
- An identifying relationship means that the child table cannot exist independently without the parent table—it is partially identified by and partially dependent on the parent. In this case, the parent table's primary key is included as part of the child table's composite primary key, meaning the child's identity is tied directly to the parent.
- A non-identifying relationship, by contrast, means the child table is not dependent on the parent table for its own identity—it simply includes the parent's primary key as a foreign key for reference, but that foreign key does not form part of the child table's primary key. The child can exist and be uniquely identified independently, even if the parent is removed.
