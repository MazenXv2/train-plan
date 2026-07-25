---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/4/tables/","dg-note-properties":{}}
---

_"a table is a bucket into which data is poured."_
## The Bucket Metaphor
Imagine each table in a database as a physical **bucket**.

- You pour data (records/rows) into this bucket.
    
- Everything inside that specific bucket should belong together for a single, clear reason.
    
- If you start pouring unrelated items into the same bucket (e.g., apples, shoes, and paperwork), it becomes a messy, confusing pile.
    

In database terms, the "bucket" (table) must hold data that is **directly associated** with a single subject or entity


_"Data in a specific table is directly associated with all other items in that same table."_
## The Core Principle: Direct Association
This means that **every row** in a table represents one instance of that table's subject, and **every column** in that row describes that specific instance.

![Pasted image 20260713201811.png](/img/user/Books%20Summery/Beginning%20Database%20Design/4/Pasted%20image%2020260713201811.png)

![Pasted image 20260713201825.png](/img/user/Books%20Summery/Beginning%20Database%20Design/4/Pasted%20image%2020260713201825.png)


