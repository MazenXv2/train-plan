---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/5/beyond-3rd-normal-form-3-nf/boyce-codd-normal-form-bcnf/","dg-note-properties":{}}
---

- A table must be in 3NF.
- A table can have only one candidate key.

A candidate key has potential for being a table’s primary key. A table is not allowed more than one primary key because referential integrity requires it as such. It would be impossible to check foreign keys against more than one primary key. Referential integrity would be automatically invalid, unenforceable, and, thus, there would be no relational database model.

