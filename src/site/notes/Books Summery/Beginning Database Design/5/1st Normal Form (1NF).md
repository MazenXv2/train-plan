---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/5/1st-normal-form-1-nf/","dg-note-properties":{}}
---

### 1NF the Academic Way

- Eliminates repeating groups.
- Defines primary keys.
- All records must be identified uniquely with a primary key. A primary key is unique and thus no duplicate values are allowed.
- All fields other than the primary key must depend on the primary key, either directly or indirectly.
- All fields must contain a single value.
- All values in each field must be of the same datatype.
- Create a new table to move the repeating groups from the original table.

![Pasted image 20260722161941.png](/img/user/Photos/Pasted%20image%2020260722161941.png)

Figure 4-8 simply shows the detail of the AUTHORSBOOKS table shown in Figure 4-7, demonstrating that leaving a table with no Normal Forms applied at all is completely silly. In fact, by definition, 1NF is actually a requirement of a relational database being relational.

![Pasted image 20260722162017.png](/img/user/Photos/Pasted%20image%2020260722162017.png)

To alleviate any potential confusion, Figure 4-9 shows how comma-delimited lists are used as another common method of displaying 0th Normal Form data, including repeated groups. The data shown in Figure 4-9 is identical to the data shown in Figure 4-8, except with a slightly different structural representation.

![Pasted image 20260722162039.png](/img/user/Photos/Pasted%20image%2020260722162039.png)

Figure 4-10 shows the application of 1NF, removing repeating fields by creating a new table where the original and new table are linked in a master-detail, one-to-many relationship.

![Pasted image 20260722162054.png](/img/user/Photos/Pasted%20image%2020260722162054.png)

In Figure 4-11, primary keys are created on both tables where the detail table has a composite primary key. The composite primary key contains the master table primary key field as the prefix field of its primary key. Therefore, the prefix field AUTHOR on the BOOK table is the foreign key pointing back to the master table AUTHOR.

![Pasted image 20260722162158.png](/img/user/Photos/Pasted%20image%2020260722162158.png)

Figure 4-12 shows what the data looks like in the altered AUTHOR table and the new BOOK table, previously the AUTHORSBOOKS table. Notice how the introduction of the relationship between the two tables allows any number of books for each author to be catered for.

![Pasted image 20260722162331.png](/img/user/Photos/Pasted%20image%2020260722162331.png)

Looking at Figure 4-12, it is apparent that applying 1NF to the AUTHORSBOOKS in Figure 4-7 has not actually saved any physical space. It has, however, saved on unused metadata slots for numerous TITLE_nn and PAGES_nn fields. Thus, each field across a record contains different information where titles and pages are not repeated a fixed number of times for each record. Also, there is no restriction on the number of books. Perhaps most importantly, the data is better organized in 1NF and it now is actually a relational database model.


Try It Out:  Figure 4-13 shows a 0th Normal Form table: 
1. Put the SALES table shown in Figure 4-13 into 1NF. 
2. Create a new table with the appropriate fields. 
3. Remove the appropriate fields from the original table. 
4. Create primary keys in the original and new tables. 
5. Create the one-to-many relationship between the original and new tables by defining the foreign key in the new table.
   

![Pasted image 20260722162832.png](/img/user/Photos/Pasted%20image%2020260722162832.png)



