---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/4/validation-and-null-values-and-constraints/","dg-note-properties":{}}
---

Relational databases allow constraints, which restrict values that are allowed to be stored in table fields:

1. NOT NULL—This is the simplest of field level constraints, making sure that a value must always be entered into a field when a record is added or changed.
2. Validation check—Similar to a NOT NULL constraint, a validation checking type of constraint restricts values in fields when a record is added or changed in a table. A check validation constraint can be as simple as making sure a field allowing only M for Male or F for Female, will only ever contain those two possible values. Otherwise, check validation constraints can become fairly complex in some relational databases, perhaps allowing inclusion of user written functions running SQL scripting.