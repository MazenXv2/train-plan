---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/4/fields-and-columns-and-attributes/","dg-note-properties":{}}
---


The terms field, column, and attribute all mean the same thing. They are all terms used to describe a field in a table. A field applies structure and definition to a chunk of data within each repeated record. Data is not actually repeated on every record, but the structure of fields is applied to each record. So, data on each record can be different, both for the record as a whole, and for each field value. Note the use of the term “can be” rather than “is,” implying that there can be duplication across both fields and records, depending on requirements and constraints.

A constraint constrains (restricts) a value. For example, in Figure 3-5 the second box showing NOT NULL for the first three fields specifies that the ISBN, PUBLISHER_ID, and PUBLICATION_ID fields can never contain NULL values in any record.

