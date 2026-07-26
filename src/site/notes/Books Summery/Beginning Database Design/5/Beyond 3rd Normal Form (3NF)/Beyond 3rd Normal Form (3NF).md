---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/5/beyond-3rd-normal-form-3-nf/beyond-3rd-normal-form-3-nf/","dg-note-properties":{}}
---

As stated earlier in this chapter, many modern relational database models do not extend beyond 3NF. Sometimes 3NF is not used at all. The reason why is because of the generation of too many tables and the resulting complex SQL code joins, with resulting terrible database response times.

### Why Go Beyond 3NF?

The objective of naming this section “Beyond 3rd Normal Form is to, in a small way, show the possible folly of using Normal Forms beyond 3NF. The biggest problems with going beyond 3NF are complexity and performance issues. Too much granularity actually introduces complexity, especially in a relational database. After all, a relational structure is not an object structure. Object structures become more simplistic as they are further reduced. Object database reduction is equivalent to the extremes of normalization in a relational database.

Extreme reduction in a relational database has the opposite effect to that of an object database where everything gets far to complex—even more complicated than is possible to manage. Extreme forms of reduction are not of benefit to the relational database model. Additionally, in a relational database the more normalization that is used then the greater the number of tables. The greater the number of tables, the larger SQL query joins become. The larger joins become the poorer database performance.

Extreme levels of granularity in relational database modeling are a form of mathematical perfection. These extremes rarely apply in fast-paced commercial environments. Commercial operations require that a job is done efficiently and cost effectively. Perfection in database model design is a side issue to that of making a profit.

### Beyond 3NF the Easy Way

In this section, you begin with the easy way, and not the academic way, as previously. Beyond 3NF are Boyce-Codd normal form (BCNF), 4NF, 5NF, and Domain Key Normal Form (DKNF). Yoiks! That’s just one or two Normal Forms to deal with. It always seems so inconceivable that relational database models can become so horribly complicated. After all, the essentials are by and large covered by 1NF and 2NF, with occasional need for 3NF transformations.

### One-to-One NULL Tables

Figure 4-30 shows removal of two often to be NULL valued fields from a table called EDITION, creating the new table called RANK. The result is a zero or one-to-one relationship between the RANK and EDITION tables. This implies that if a RANK record exists, then a corresponding EDITION record must exist as well. In the opposite case, however, an EDITION record can exist where a RANK record does not have to exist. This opposite case, accounts for an edition of a publication having no RANK and INGRAM_UNITS values. A recently published publication will rarely have any statistical information.

![Pasted image 20260726133949.png](/img/user/Photos/Pasted%20image%2020260726133949.png)


Figure 4-31 shows a data picture of the normalized structure shown at the lower-right of the diagram in Figure 4-30. What has effectively happened is that potentially NULL valued fields are moved into a new table, creating a one-to-one or zero relationship.

![Pasted image 20260726134055.png](/img/user/Photos/Pasted%20image%2020260726134055.png)


In the case of the tables shown in Figure 4-30 and Figure 4-31, it should be made apparent that the RANK and INGRAM_UNITS fields are not co-dependent. They do not depend on each other and are completely unrelated. It is quite possible that one field may be NULL valued and the other not. In the extreme, the data model could include two new tables, as shown in Figure 4-32. This level of normalization is completely absurd and seriously overzealous. In modern high-end relational database engines with variable record lengths, this is largely irrelevant. Once again disk space is cheap, plus increased numbers of tables leads to bigger SQL joins and poorer performance.

This level of normalization separating NULL valued fields into separate tables is simply going much too far, making too many little-bitty pieces and introducing too much granularity. This practice is not recommended; however, it is common. If normalization of this depth can be undone without destroying applications, do so—especially if there is a performance problem with SQL code joins.


![Pasted image 20260726134408.png](/img/user/Photos/Pasted%20image%2020260726134408.png)


### Beyond 3NF the Academic Way

- BCNF or Boyce-Codd Normal Form (BCNF)—Every determinant in a table is a candidate key. If there is only one candidate key, then 3NF and BCNF are one and the same.
- 4th Normal Form (4NF)—Eliminate multiple sets of multi-valued dependencies.
- 5th Normal Form (5NF)—Eliminate cyclic dependencies. 5NF is also known as Projection normal form (PJNF).
- Domain Key normal form (DKNF)—DKNF is the ultimate application of normalization and is more of a measurement of conceptual state as opposed to a transformation process in itself.


- [[Books Summery/Beginning Database Design/5/Beyond 3rd Normal Form (3NF)/Boyce-Codd Normal Form (BCNF)\|Boyce-Codd Normal Form (BCNF)]]
- 