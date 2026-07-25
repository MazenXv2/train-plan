---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/5/2nd-normal-form-2-nf/","dg-note-properties":{}}
---


### 2NF the Academic Way

- The table must be in 1NF.
- All non-key values must be fully functionally dependent on the primary key. In other words, non-key fields not completely and individually dependent on the primary key are not allowed.
- Partial dependencies must be removed. A partial dependency is a special type of functional dependency that exists when a field is fully dependent on a part of a composite primary key.
- Create a new table to separate the partially dependent part of the primary key and its dependent fields.

### 2NF the Easy Way

2NF performs a seemingly similar function to that of 1NF, but creates a table where repeating values rather than repeating fields are removed to a new table. The result is a many-to-one relationship rather than a one-to-many relationship, created between the original and the new tables. The new table gets a primary key consisting of a single field. Typically, 2NF creates many-to-one relationships between dynamic and static data, removing static data from transactional tables into new tables. In Figure 4-15, the BOOK table is in 1NF after separation of repeating group books from the authors. The publisher and subject information are relatively static compared with books. In this situation, the BOOK table represents transactional or dynamic information where books are published, reprinted, discontinued, produced in different formats, among other options. All the while, publishers and subjects remain relatively the same—static.


![Pasted image 20260722200031.png](/img/user/Photos/Pasted%20image%2020260722200031.png)


Figure 4-16 shows the initial stage of the application of 2NF, removing static publisher and subject information from the more dynamic BOOK transaction table. 

In Figure 4-17, many-to-one relationships are established between dynamic and static tables, namely BOOK to PUBLISHER and BOOK to SUBJECT tables. When applying 1NF, one-to-many relationships are established between master and detail tables, as shown in Figure 4-10 and Figure 4-11. Application of 2NF establishes the same relationship, except it is easier to understand the slight difference by calling that relationship a many-to-one relationship. Mathematically, a one-to-many relationship is identical to a many-to-one relationship; however, 1NFs and 2NFs are completely different because the one-to-many relationship is established for a completely different reason.

![Pasted image 20260722200421.png](/img/user/Photos/Pasted%20image%2020260722200421.png)

![Pasted image 20260722200444.png](/img/user/Photos/Pasted%20image%2020260722200444.png)


In Figure 4-18, primary keys are created on both the PUBLISHER and SUBJECT tables to uniquely identify individual publishers and subjects within their two respective tables. Identifying relationships as BOOK related to PUBLISHER and BOOK related to SUBJECT causes the publisher and subject primary key values to be included in the composite primary key of the BOOK table.

![Pasted image 20260722203117.png](/img/user/Photos/Pasted%20image%2020260722203117.png)


The design flaw in Figure 4‑18 lies in treating the relationship between a dynamic table 
(like `BOOK`) and its static lookup tables (like `SUBJECT` and `PUBLISHER`) as identifying, which incorrectly forces the static tables' primary keys into the `BOOK` table's composite primary key. This logic is fundamentally wrong because static data exists completely independently—for instance, you can add the "Science Fiction" subject to the `SUBJECT` table without having a single book in that genre, and a publisher can be listed before any of their books are entered. Since the existence of a child record (Book) does not depend on the existence of its parent (Subject or Publisher), the relationship must be non-identifying instead, as shown in Figure 4‑19. By making this change, the publisher and subject keys become simple foreign keys, allowing the `BOOK` table to use its own standalone primary key (e.g., `book_id`) rather than being forced into an overly restrictive composite key that falsely implies dependency on its lookups.


![Pasted image 20260722203619.png](/img/user/Photos/Pasted%20image%2020260722203619.png)


The design flaw in Figure 4‑18 lies in treating the relationship between a dynamic table (like `BOOK`) and its static lookup tables (like `SUBJECT` and `PUBLISHER`) as identifying, which incorrectly forces the static tables' primary keys into the `BOOK` table's composite primary key. This logic is fundamentally wrong because static data exists completely independently—for instance, you can add the "Science Fiction" subject to the `SUBJECT` table without having a single book in that genre, and a publisher can be listed before any of their books are entered. Since the existence of a child record (Book) does not depend on the existence of its parent (Subject or Publisher), the relationship must be non-identifying instead, as shown in Figure 4‑19. **For example, imagine a book is initially cataloged under "Science Fiction" but later turns out to be "Fantasy." In the flawed identifying design, changing the subject would mean actually altering the book's composite primary key—a notoriously expensive and hazardous operation that would require cascading that change to every order, review, or inventory record that references this book. In the corrected non-identifying design, however, you simply update the `subject_id` foreign key column in the `BOOK` table, leaving the book's stable, standalone primary key (`book_id`) untouched, making the change instantaneous and risk-free.** By making this change, the publisher and subject keys become simple foreign keys, allowing the `BOOK` table to use its own standalone primary key (e.g., `book_id`) rather than being forced into an overly restrictive composite key that falsely implies dependency on its lookups.


![Pasted image 20260722205113.png](/img/user/Photos/Pasted%20image%2020260722205113.png)


Figure 4-20 shows what the data looks like in the altered BOOK table with the new PUBLISHER and SUB JECT tables shown as well. Multiple fields of publisher and subject field information previously duplicated on the BOOK table (as shown in Figure 4-15) is now separated into the two new PUBLISHER and SUBJECT tables, with duplicate publishers and subjects removed from the new tables.


![Pasted image 20260722205259.png](/img/user/Photos/Pasted%20image%2020260722205259.png)


It is readily apparent from Figure 4-20 that placing the BOOK table into 2NF has physically saved space. Duplication has been removed, as shown by there now being only a single SUBJECT record and far fewer PUBLISHER records. Once again, data has become better organized by the application of 2NF to the BOOK table.



Try It Out : 2nd Normal Form

Figure 4-21 shows two tables in 1NF. Put the SALE_ORDER and SALE_ORDER_ITEM tables shown in Figure 4-21 into 2NF:

1. Create two new tables with the appropriate fields. 
2. Remove the appropriate fields from the original tables. 
3. Create primary keys in the new tables. 
4. Create the many-to-one relationships between the original tables and the new tables, defining and placing foreign keys appropriately.


![Pasted image 20260722205813.png](/img/user/Photos/Pasted%20image%2020260722205813.png)

