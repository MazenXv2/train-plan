---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/2/understanding-database-model-design/","dg-note-properties":{}}
---


Database design is so important because all applications written against that database model design are completely dependent on the structure of that underlying database. If the database model must be altered at a later stage, everything constructed based on the database model probably must be changed and perhaps even completely rewritten. That’s all the applications—and I mean all of them! That can get very expensive and time consuming. Design the database model in the same way that you would design an application—using tools, flowcharts, pretty pictures, Entity Relationship Diagrams (ERDs), and anything else that might help to ensure that what you intend to build is not only what you need, but also will actually work, and preferably work without ever breaking.

A computer system, and database model that ultimately turns into a complete dud as a result of poor planning and design can cost a company more money than it is prepared to spend and perhaps more than a company is even able to lose.

Design is the process of ensuring that it all works without actually building it. Design is a little like testing something on paper before spending thousands of hours building it in possibly the wrong way.

Design is needed to ensure that it works before spending humungous amounts of money finding out that it doesn’t. The idea is to fix as many teething problems and errors in the design. Fixing the design is much easier than fixing a finished product. A design on paper costs a fraction of what building and implementing the real thing would cost. Waste a small amount of money in planning, rather than lose more than can be afforded when it’s too late to fix it.

## Defining the Objectives

Defining objectives is probably the single most important task done in planning any project, be it a skyscraper or a database model. You could, of course, just start anywhere and dive right into the project with your eyes shut. But that is not planning. The more you plan what you are going to do, the more likely the final result will fit your requirements.

Defining the objectives is the basic step of defining how you are going to get from A to B.

##### There are, of course, a number of points to guide the establishment of design objectives for a proper relational database model design:

1. Aim for a well-structured database model: A well-structured database model is simple, easy to read, and easy to comprehend. If your company has a database model made up of 50 pieces of A4-sized paper taped to an entire wall, and links between tables taking 20 minutes to trace, you have a problem. 

2. Data integrity: Integrity is a set of rules in a database model, ensuring that data is not lost within the database, and that data is only destroyed when it should be.

3. Support both planned queries and ad-hoc or unplanned queries: Planned queries are pre-written, routine reports (like daily sales summaries) that run efficiently because the database knows what to expect, while ad-hoc queries are spontaneous, unpredictable questions (like "show me all customers who bought X and Y in the last 3 hours") that users invent on the fly. In high-traffic OLTP databases, these unplanned queries are dangerous because they can scan millions of records, eat up system resources, and slow down or lock out the critical real-time transactions. Therefore, the fewer ad-hoc queries the better; in extreme cases, they must be banned entirely from the OLTP system and redirected to a separate data warehouse platform, which is specifically optimized for heavy, flexible analysis without jeopardizing daily operations.

4. Ad-hoc queries can cause serious performance issues. Customer-facing applications that require millisecond response times (which depend solely on a high-performance OLTP database) do not get along well with ad-hoc queries. Don’t risk losing your customers and wind up with no business to speak of. Do not allow anyone to do anything ad-hoc in an application-controlled OLTP database.

5. database design must align with **business objectives**, not blindly follow normalization rules. While highly normalized structures are great for OLTP systems (to ensure data integrity during fast transactions), they don't necessarily reflect how a business actually operates. In contrast, **denormalized** structures (like those used in data warehouses) often look more like the business itself—making them better suited for management-level **ad-hoc queries** and strategic analysis. The key takeaway is that subjecting a customer-facing OLTP database to ad-hoc activity can be disastrous for performance and customer satisfaction, so the level of normalization should be chosen based on the database's **purpose**: strict normalization for transactional systems, and looser, more business-aligned structures for analytical systems where flexibility is more important than raw speed.

6. database must be fast and efficient enough to handle whatever type of change is thrown at it, whether it's a customer updating their own phone number in an OLTP system (a tiny, single-record change) or a nightly batch process that recalculates millions of sales totals in a data warehouse (a massive, high-speed operation). The key point is that performance isn't a one-size-fits-all concept—an OLTP system needs millisecond response times for individual actions, while a data warehouse needs raw throughput to process huge volumes of data quickly. Ultimately, the design, indexing, and hardware must be tuned specifically for the **type and scale of changes** that database is expected to perform.

7. Future growth must always be a serious consideration: **databases can experience explosive, unpredictable growth**, especially in internet-facing OLTP systems where a viral ad or a lucky social media post can suddenly bring millions of new users overnight. Unlike data warehouses, where growth follows a predictable schedule (e.g., daily or weekly data loads), OLTP growth can spike without warning, and if the database design can't handle the surge in transactions, the system will slow down or crash—causing frustrated customers to abandon your site just as quickly as they discovered it, potentially destroying the business overnight.

8. Minimize dependence between applications and database model structures if you expect change. This makes it easier to change and enhance both database model and application code in the future.


##### positive results from using good database model design objectives are as follows:

1. From an operational perspective, the most important objective is fulfilling the needs of applications.
2. Queries should be relatively easy to code without producing errors because of lack of data integrity or poor table design.
3. The easier applications can be built, the better. In general, the less co-dependence between database model and application, the better. In tightly controlled OLTP application environments where no ad-hoc activity is permitted, this is easy.


## Looking at Methods of Database Design

##### There are various methodologies available for designing database models. Each of these different approaches consists of a number of steps. The following sequence of steps to database model design:

- Requirements analysis—Collect information about the nature of the data, features required, and any specialized needs such as expected output responses. This step covers what is needed, so simply analyze it and write it down. Talk to the customer and company employees to get a better idea of exactly what they need.
- Conceptual design—This is where you get to use the fancy graphical tools and draw the pretty pictures—Entity Relationship Diagrams (ERDs). This step includes creation of tables, fields within those tables, and relationships between the tables.
- Logical design—Create database language commands to generate table definitions. Some tools used for creating ERDs allow generation of data definition language (DDL) scripting; however, they are likely to generate generic scripts. Be sure that you check anything generated before exe cutting in any specific database engine.
- Physical design—Adjust database language commands to alter the database model for the underlying physical attributes of tables. For example, you might want to store large binary objects in separate, underlying files to that of standard relational record-field data.
- Tuning phase—This step includes items such as appropriate indexing, further normalization, or even denormalization, security features, and anything else not covered by the previous steps.

These separate steps are interchangeable, repeatable, iterative, and really anything-able, according to various different approaches used for different database engines and different database designer personal preferences. Some designers may even put some of these steps into single steps and divide others up into more detailed sets of subset steps. In other words, this is all open to interpretation. The only thing I do insist that should be universal is that you draw the ERDs and build tables well before you build metadata table creation code, placing visual design prior to physical implementation