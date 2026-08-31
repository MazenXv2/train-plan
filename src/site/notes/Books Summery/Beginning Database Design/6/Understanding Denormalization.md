---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/6/understanding-denormalization/","dg-note-properties":{}}
---

Denormalization is often (but not always) the opposite of normalization. (See Chapter 4 for a discussion of normalization.) Denormalization can be applied to a database model to create data warehouse or reporting only type tables. Denormalization is also sometimes required as a solution to reviving dying OLTP applications that exhibit dreadful performance. This can be a result of past profligate use of normalization in the development of database models and applications. Too much granularity in normalization can cause as many problems as it solves. So, denormalization often attempts to reverse granularity, created by over-application of Normal Forms during the normalization process.

### Denormalizing Beyond 3NF

Figure 6-1 shows reversal of the normalization processing applied in Figure 4-28, Figure 4-29, and Figure 4-30. Removing nullable fields to separate tables is a common method of saving space, particularly in databases with fixed record lengths. Many modern databases allow variable record lengths. If variable length records are allowed, removal of NULL valued fields is pointless because the space saved is either none, or completely negligible.

![Pasted image 20260801211755.png](/img/user/PDFs/Pasted%20image%2020260801211755.png)


Figure 6-2 shows reversal of normalization processing applied in Figure 4-31. Figure 6-2 shows a particularly upsetting application of BCNF where all candidate keys are separated into separate tables. A candidate keyis any field that potentially can be used as a primary key (unique identifier) for the original entity, in this case a customer (the CUSTOMERtable). Applying this type of normalization in a commercial environment would result in incredibly poor performance and is more of a mathematical nicety rather than a commercial necessity.

![Pasted image 20260801212252.png](/img/user/PDFs/Pasted%20image%2020260801212252.png)

Once again Figure 6-3 shows another application of the reversal of BCNF. Application of BCNF in Figure 4-32 and Figure 4-33 created three tables, each with unique combinations of unique values from the PROJECTStable on the left. Accessing of unique records in the PROJECTtable can be handled with application coding more effectively, without the downside of creating too much granularity in table structures.

![Pasted image 20260801212504.png](/img/user/Photos/Pasted%20image%2020260801212504.png)


Figure 4-34, Figure 4-35, Figure 4-36, and Figure 4-38 show a typical 4NF transformation where multiple valued lists in individual fields are separated out into separate tables. This type of denormalization is shown in Figure 6-4.

![Pasted image 20260801212753.png](/img/user/Photos/Pasted%20image%2020260801212753.png)



