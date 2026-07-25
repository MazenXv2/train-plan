---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/4/relations-and-relationships-and-some-normalization/","dg-note-properties":{}}
---


In reality, normalization usually manages to divide information into smaller, more manageable parts, preferably not too small. The most obvious redundancies can usually be removed without getting too deeply mathematical about everything. Commercially speaking, primary objectives are usually to save space and organize data for usability and manageability, without sacrificing performance. All this is often a juggling act and commonly partially ironed out by trial and error. Additionally the demands of intensely busy applications and end-user needs can tend to necessitate breaking the rules of normalization in many ways to meet performance requirements.

Normalization is an incremental process. In other words, each Normal Form layer adds to whatever Normal Forms have already been applied. For example, 2nd Normal Form can only be applied to tables in 1st Normal Form, and 3rd Normal Form only applied to tables in 2nd Normal Form, and so on. Each Normal Form is a refinement of the previous Normal Form.

### Benefits of Normalization Effectively 
minimizing redundancy is another way of describing removal of duplication. The effect of removing duplication is as follows:
- Physical space needed to store data is reduced.
- Data becomes better organized.
- Normalization allows changes to small amounts of data (namely single records) to be made to one table at once. You don’t have to search through an entire database to change a single field value in a single record, just the table.


### Potential Normalization Hazards
While normalization is essential for minimizing data redundancy and maintaining integrity, taking it too far can create serious hazards, particularly in terms of performance and usability. Over-normalization leads to excessive granularity—meaning data is split into too many tiny tables—which forces complex queries with numerous joins, severely slowing down high-concurrency OLTP systems and hurting their transactional performance. Additionally, in data warehouses, this over-granularity makes the table structure so technically fragmented that non-technical end-users find it impossible to interpret or query directly. Thus, the key is to apply normalization judiciously, balancing the benefits of redundancy reduction against the practical needs of performance and user accessibility, rather than blindly pursuing the highest normal form in every situation.

Keep the following in mind:
- Physical space is not nearly as big a concern as it used to be, because disk space is one of the cheapest cost factors to consider(unless, of course, when dealing with a truly huge data warehouse).
- Too much minimization of redundancy implies too much granularity and too many tables. Too many tables can lead to extremely huge SQL join queries. The more tables in a SQL join query, the slower queries execute. Performance can be so drastically affected as to make applications completely useless.
- Better organization of data with extreme amounts of redundancy minimization can actually result in more complexity, particularly if end-users are exposed to database model structure.'



