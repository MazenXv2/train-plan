---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/5/3rd-normal-form-3-nf/","dg-note-properties":{}}
---

### 3NF the Academic Way

- The table must be in 2NF.
- Eliminate transitive dependencies. A transitive dependency is where a field is indirectly determined by the primary key because that field is functionally dependent on a second field, where that second field is dependent on the primary key.
- Create a new table to contain any separated fields.



### 3NF the Easy Way

3NF is an odd one and can often cause confusion. In basic terms, every field in a table that is not a key field must be directly dependent on the primary key. There are number of different ways to look at 3NF, and this section goes through them one by one.

Figure 4-23 shows one of the easiest interpretations of 3NF where a many-to-many relationship presents the possibility that more than one record will be returned using a query joining both tables.

![Pasted image 20260722214832.png](/img/user/Photos/Pasted%20image%2020260722214832.png)


Figure 4-24 illustrates the many-to-many relationship between employees and tasks in the original 2NF structure, reflecting that employees perform multiple tasks in their daily routines. In this design, querying for the employee Columbia will always return the three tasks assigned to her, while querying for a specific task will always return all employees (like the two shown for the third task) assigned to it. The serious flaw emerges when you need to reference or store attributes for a _single, unique pairing_—such as Columbia’s performance specifically on the third task—because there is no primary key that identifies that exact employee-task combination. Without the new `ASSIGNMENT` table introduced by the 3NF transformation in Figure 4-23 to explicitly represent each employee-task pair as a distinct record, it is impossible to locate, update, or manage an individual assignment.

The 3NF transformation (Figure 4-23) resolves this by creating the `ASSIGNMENT` table, which turns the implicit many-to-many relationship into two independent one-to-many relationships. This new table gives each specific employee-task pairing its own unique primary key, effectively making "an assignment" a tangible, queryable entity in the database.


![Pasted image 20260722215255.png](/img/user/Photos/Pasted%20image%2020260722215255.png)



Another way to look at 3NF is as displayed in Figure 4-25, where fields common to more than one table can be moved to a new table, as shown by the creation of the FOREIGN_EXCHANGE table. At first, this looks like a 2NF transformation because fields not dependent on the primary key are removed to the new table; however, currencies should be conceived as being dependent upon location. Both CUSTOMER and SUPPLIER have addresses and, thus, there are transitive dependencies between currencies, through addresses (location), ultimately to customers and suppliers. Customers and suppliers use specific currencies depending on what country they are located in. Figure 4-25 shows a 3NF transformation allowing removal of common information from the CUSTOMER and SUPPLIER tables for two reasons: 

❑ Currency coding and rate information does not depend on CUSTOMER and SUPPLIER primary keys, even though which currency they use does depend on who the customer or supplier are, based on the country in which they do business.

❑ The CURRENCY and EXCHANGE_RATE fields in the pre-transformation tables are transitively dependent on CUSTOMER and SUPPLIER primary keys because they depend on the CURRENCY_CODE, which in turn does depends on addresses.

If we keep `CURRENCY_CODE` and `EXCHANGE_RATE` in the `CUSTOMER` and `SUPPLIER` tables, we encounter two major practical problems:

- **Update Anomaly**: If the exchange rate for the US Dollar changes, you have to update every single `CUSTOMER` and every single `SUPPLIER` who uses USD. If you miss just one customer, your financial reports will be inconsistent.
    
- **Insertion Anomaly**: You cannot add a new currency (e.g., "Euro") to the system until you have a customer or supplier who actually uses it. The currency must wait for a child record to exist.

Solve: 

**1. Update Anomaly:** Before 3NF, changing an exchange rate meant updating thousands of rows across `CUSTOMER` and `SUPPLIER` tables—risking inconsistency if any row was missed—whereas after 3NF, you update a single row in the central `FOREIGN_EXCHANGE` table, and every linked customer/supplier automatically sees the new rate instantly.

**2. Insertion Anomaly:** Before 3NF, adding a new currency required creating a dummy customer/supplier record just to hold the currency data—cluttering the database with garbage—whereas after 3NF, you simply insert the currency directly into the standalone `FOREIGN_EXCHANGE` table anytime, ready for future customers or suppliers to reference.

![Pasted image 20260722223256.png](/img/user/Photos/Pasted%20image%2020260722223256.png)


`The transformation in Figure 4-25 could be conceived as being two 2NF transformations because a many-to-one relationship is creating a more static table by creating the FOREIGN_EXCHANGE table.`


Obviously, the 3NF transformation shown in Figure 4-25 decreases the size of the database in general because repeated copies of CURRENCY and EXCHANGE_RATE fields have been normalized into the FOREIGN_EXCHANGE table and completely removed from the CUSTOMER and SUPPLIER tables. No data example is necessary in this case because the diagram in Figure 4-25 is self-explanatory.


Another commonly encountered version of 3NF is as shown in Figure 4-26. In this case, there is a very clear transitive dependency from CITY to DEPARTMENT and on to the EMPLOYEE primary key field.

![Pasted image 20260722223430.png](/img/user/Photos/Pasted%20image%2020260722223430.png)


`A transitive dependency occurs where one field depends on another, which in turn depends on a third field —the third field typically being the primary key. A state of transitive dependency can also be inter preted as a field not being entirely dependent on the primary key.`


In Figure 4-26, a transitive dependency exists because it is assumed that each employee is assigned to a particular department. Each department within a company is exclusively based in one specific city. In other words, any company in the database does not have single departments spread across more than a single city. As stated in Figure 4-26, this type of normalization might be getting a little over zealous in terms of creating too many tables, possibly resulting in slow queries having to join too many tables.

Another very typical 3NF candidate is as shown in Figure 4-27, where a calculated value is stored in a table. Also, the calculated value results from values in other fields within the same table. In this situation, the calculated field is actually non-fully dependent on the primary key (transitively dependent) and thus does not necessarily require a new table. Calculated fields are simply removed.

![Pasted image 20260722223644.png](/img/user/Photos/Pasted%20image%2020260722223644.png)

`There is usually a good reason for including calculated fields — usually performance denormalization. (Denormalization is explained as a concept in a later chapter.) In a data warehouse, calculated fields are sometimes stored in materialized views. Data warehouse database modeling is also covered in a later chapter.`


Try It Out : 3rd Normal Form

Figure 4-28 shows four tables:

1. Assume that any particular department within the company is located in only one city. Thus, assume that a city is always dependent upon which department a sales order occurred within. 
2. Put the SALE_ORDER and STOCK_ITEM tables into 3NF. 
3. Remove some calculated fields and create a new table. 
4. Remove the appropriate fields from an original table to a new table. 
5. Create a primary key in the new table. 
6. Create a many-to-one relationship between the original table and the new table, defining and placing a foreign key appropriately


![Pasted image 20260722223823.png](/img/user/Photos/Pasted%20image%2020260722223823.png)

