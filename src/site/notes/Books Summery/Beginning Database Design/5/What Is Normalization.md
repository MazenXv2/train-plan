---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/5/what-is-normalization/","dg-note-properties":{}}
---


In general, normalization removes duplication and minimizes redundant chunks of data. The result is better organization and more effective use of physical space, among other factors.

While normalization undeniably improves data organization and storage efficiency by removing duplication and minimizing redundant data, the academic insistence on universally applying every successive Normal Form layer in every situation is fundamentally flawed; in my commercial experience, pushing data models into these deeper, more precise refinements is nearly always a mistake, as this approach tends to over-define the database structure for the sake of theoretical completeness rather than addressing genuine, practical business requirements.

Before going into the details of normalization, some specifics should be covered briefly, including the concept of anomalies and some rather technical mathematical jargon.

### The Concept of Anomalies
In relational theory, an **anomaly** is a logical inconsistency or error that arises in a database when you perform **insert**, **update**, or **delete** operations on a poorly structured (under-normalized) table.

The fundamental objective of relational database theory is to prevent logical data corruption by eliminating anomalies. In practice, however, environments like data warehouses routinely process transactions involving millions of records at once, making obsessive attention to such granular, single-record anomalies seem overzealous or impractical. Even though this approach can feel impractical in the real world, relational theory stays stubbornly rigid because it is built on mathematics. In math, data integrity is an all‑or‑nothing concept—your data is either completely correct or it is flawed, with no middle ground. As a result, the theory demands that we guard against every possible anomaly, no matter how many records we happen to be updating at once.

- Insert anomaly—Caused when a record is added to a detail table, with no related record existing in a master table.
- Delete anomaly—Caused when a record is deleted from a master table, without first deleting all sibling records, in a detail table.
- Update anomaly—This anomaly is similar to deletion in that both master and detail records must be updated to avoid orphaned detail records.

### Dependency, Determinants, and Other Jargon

- Functional dependency—Y is functionally dependent on X if the value of Y is determined by X. In other words, if Y = X +1, the value of X will determine the resultant value of Y. Thus, Y is dependent on X as a function of the value of X.
- Determinant—The determinant in the description of functional dependency in the previous point is X because X determines the value Y, at least partially because 1 is added to X as well.
- Transitive dependence—Z is transitively dependent on X when X determines Y and Y deter mines Z. Transitive dependence thus describes that Z is indirectly dependent on X through its relationship with Y.
- Candidate key—A candidate key (potential or permissible key) is a field or combination of fields that can act as a primary key field for a table — thus uniquely identifying each record in the table.
- Full functional dependence —This situation occurs where X determines Y, but X combined with Z does not determine Y. In other words, Y depends on X and X alone. If Y depends on X with anything else, there is not full functional dependence. Essentially X, the determinant, cannot be a composite key. A composite key contains more than one field (the equivalent of X with Z).
- Multiple valued dependency—This is also known as a multi-valued dependency. A commonly used example of a multi-valued dependency is a field containing a comma-delimited list or collection of some kind. A collection could be an array of values of the same type. Those multiple values are dependent as a whole on the primary key, as a whole meaning the entire collection in the comma delimited list. More precisely, a trivial multi-valued dependency occurs between two fields when they are the only two fields in the table. One is the primary key and the other the multi-valued list.
- Cyclic dependency—The meaning of the word “cyclic” is a circular pattern, recurrent, closed ring, or a circular chain structure. In the context of the relational database model, cyclic dependence means that X is dependent on Y, which in turn is also dependent on X, directly or indirectly. Cyclic dependence, therefore, indicates a logically circular pattern of interdependence. Cyclic dependence typically occurs with tables containing a composite primary key of three or more fields (for example, where three fields are related in pairs to each other). In other words, X relates to Y, Y relates to Z, and X relates to Z. Ultimately Z relates back to X.


## Defining Normal Forms

- ###  Defining Normal Forms the Academic Way
	- 1st Normal Form (1NF)—Eliminate repeating groups such that all records in all tables can be identified uniquely by a primary key in each table. In other words, all fields other than the primary key must depend on the primary key.
	- 2nd Normal Form (2NF)—All non-key values must be fully functionally dependent on the primary key. No partial dependencies are allowed. A partial dependency exists when a field is fully dependent on a part of a composite primary key.
	- 3rd Normal Form (3NF)—Eliminate transitive dependencies, meaning that a field is indirectly determined by the primary key. This is because the field is functionally dependent on another field, whereas the other field is dependent on the primary key.
	- Boyce-Codd Normal Form (BCNF)—Every determinant in a table is a candidate key. If there is only one candidate key, 3NF and BCNF are one and the same.
	- 4th Normal Form (4NF)—Eliminate multiple sets of multivalued dependencies
	- 5th Normal Form (5NF)—Eliminate cyclic dependencies. 5NF is also known as Projection Normal Form (PJNF).
	- Domain Key Normal Form (DKNF)—DKNF is the ultimate application of normalization and is more a measurement of conceptual state, as opposed to a transformation process in itself.


	Maintenance of data with respect to accessing of individual records in a database can be more effectively and easily managed using “beyond 3NF.” Any querying, however, is adversely affected by too many tables. In some cases, the performance factor can be completely debilitating, making a database useless. Additionally, even in highly accurate, single-record update environments, the extra functionality and accuracy given by beyond 3NF structures (BCNF, 4NF, 5NF, DKNF) can always be provided by application coding and SQL code to find those individual records.
	
	Is “beyond 3NF” unnecessary? It might be, but probably in many commercial situations it is unnecessary. Remember that application SDKs are just as powerful as database engine structural and functional capabilities. Extreme implementation of normalization using layers beyond 3NF tends to place too much functionality into the database. Why not use the best of both worlds—both database and application capabilities? Use the database to store data and allow applications to manipulate and verify data to a certain extent.



- ### Defining Normal Forms the Easy Way
	Many modern-day commercial relational database implementations do not go beyond the implementation of 3NF. This is often true of OLTP databases and nearly always true in properly designed data warehouse databases. Application of Normal Forms beyond that of 3NF tends to produce too many tables, resulting in too many tables in SQL joins. Bigger joins result in poor performance. In general, good performance is much more important than granular perfection in relational database design. 
	
	How can normalization be made simple? Why is it easy? I like to offer a simplified interpretation of normalization just to get the novice started. In a perfect world, most relational database model designs are very similar. As a result, much of the basic database design for many applications from accounting to manufacturing (and anything else you can think of) is all more or less the same.
	
	**Most of the time**, normalization is just **common sense**. You understand how your business works, and you organize your tables so things aren't messy or duplicated. You don't need a university degree to figure out what goes where. **However**, you'll occasionally run into a **tricky, rare situation** where this common-sense approach just isn't enough to keep your data perfectly clean. When that happens, instead of making your _entire_ database overly complicated and slow, you can just **use the advanced, strict rules (like 4NF or 5NF) only on that one specific problem area** to fix it, while keeping the rest of your database simple and easy to work with.
	
	- 1st Normal Form (1NF)—Removes repeating fields by creating a new table where the original and new table are linked together with a master-detail, one-to-many relationship. For example, a master table could contain parent records representing all the ships owned by a cruise line. A detail table would contain detail records, such as all the passengers on a cruise to the Caribbean. Create primary keys on both tables where the detail table will have a composite primary key containing the master table primary key field as the prefix field of its primary key. That prefix field is also a foreign key back to the master table.
	
	- 2nd Normal Form (2NF)—Performs a seemingly similar function to that of 1NF, but creates a table where repeating values (rather than repeating fields as for 1NF) are removed to a new table. The result is a many-to-one relationship rather than a one-to-many relationship, created between the original and the new tables. The new table gets a primary key consisting of a single field. The master table contains a foreign key pointing back to the primary key of the new table. That foreign key is not part of the primary key in the original table.
	
	- 3rd Normal Form (3NF)—It is difficult to explain 3NF without using a mind bogglingly confusing technical definition. Elimination of a transitive dependency implies creation of a new table for something indirectly dependent on the primary key in an existing table. There are a multitude of ways in which 3NF can be interpreted.
	
	- Beyond 3NF—Many modern relational database models do not extend beyond 3NF. Sometimes 3NF is not used at all. The reason why is because of the generation of too many tables and the resulting complex SQL code joins, with resulting terrible database response times. One common case that bears mentioning is removal of potentially NULL valued fields into new tables, creating a one-to-one relationship. In modern high-end relational database engines with variable record lengths, this is largely irrelevant. Disk space is cheap and, as already stated, increased numbers of tables leads to bigger SQL joins and poorer performance.