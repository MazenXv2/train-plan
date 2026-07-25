---
{"dg-publish":true,"permalink":"/books-summery/beginning-database-design/4/datatypes/","dg-note-properties":{}}
---


Datatypes can be divided into three separate sections:
- Simple datatypes—These are datatypes applying a pattern or value limitation on a single value such as a number.
- Complex datatypes—These include any datatypes bridging the gap between object and relational databases, including items such as binary objects and collection arrays. Specifics on complex datatypes are not strictly necessary for this book as they are more object-oriented than relational in nature.
- Specialized datatypes —These are present in more advanced relational databases catering to inherently structured data such as XML documents, spatial data, multimedia objects and even dynamically definable datatypes.

### Simple Datatypes
1. Strings: A string is a sequence of one or more characters.
	- Fixed-length strings—A fixed-length string will always store the specified length declared for the datatype. The value is padded with spaces even when the actual string value is less than the length of the datatype length. For example, the value NY in a CHAR(3) vari able would be stored as NY plus a space character. Fixed-length strings are generally only used for short length strings because a variable-length string (discussed next) requires storage of both value and length.
	- Variable-length strings —A variable-length string allows storage into a datatype as the actual length of the string, as long as a maximum limit is not exceeded. The length of the string is variable because when storing a string of length less than the width specified by the datatype, the string is not padded (as is the case for fixed-length strings). Only the actual string value is stored. Storing the string XXX into a variable length string datatype of ten characters in length stores the three characters only, and not three characters padded by seven spaces. Different databases use different naming conventions for variable-length string datatypes. VARCHAR(n) or TEXT(n) are common naming formats for variable-length strings.

2. Numbers—Numeric datatypes are often the most numerous field datatypes in many database tables.
	- Integers—An integer is a whole number such that no decimal digits are included. Some databases allow more detailed specification using small integers and long integers, as well and standard-sized integer datatypes.
	- Fixed-length decimals—A fixed-length decimal is a number, including a decimal point, where the digits on the right side of the decimal are restricted to a specified number of digits. For example, a DECIMAL(5,2) datatype will allow the values 4.1 and 4.12, but not 4.123.
	- Floating points—A floating-point number is just as the name implies, where the decimal point “floats freely” anywhere within the number. In other words, the decimal point can appear anywhere in the number. Floating-point values can have any number of digits both before and after the decimal point, even none on either side. Values such as 1000, 1000.12345, and 0.8843343223 are valid floating-point numbers. Floating point values are likely to be less efficient in storage and retrieval than fixed-length decimals and integers because they are less predictable in terms of record length and otherwise.

3. Dates and times—Dates can be stored as simple dates or dates including timestamp information. In actuality, simple dates are often stored as a Julian date or some other similar numbering system. A Julian date is a time in seconds from a specified start date (such as January 1, 1960). When simple date values are set or retrieved in the database, they are subjected to a default formatting process spitting out to, for example, a dd/mm/yyyy format excluding seconds (depending on default database formatting settings, of course). A timestamp datatype displays both date and time information regardless of any default date formatting executing in the database (sometimes stored as a special timestamp datatype).

### Complex Datatypes
1. Binary objects: 
	1. The Core Problem: Size Mismatch: Databases are optimized for handling small, uniform pieces of data—typically **strings and numbers**.
		- **Standard table record size:** Usually fits within **2 KB** (often called a _page_ or _block_). This is the fundamental unit of storage the database reads and writes to disk.
		    
		- **Modern file size:** A single small graphic (like a JPEG or PNG for a website) is easily **larger than 2 KB**. A sound file, video, or high-res image is exponentially larger.
	
	2. Why This Breaks Database Performance : To understand why this is a disaster for performance, you have to understand how databases read data:
	
		- Databases use a **"cache" or "buffer pool"** in memory. They read entire 2 KB blocks from disk into this cache at once.
		    
		- The database assumes that if you request one record in that block, you are likely to need the _other_ records in that same block soon (this is called _locality of reference_).
		
		**The Disaster Scenario:**  
		Imagine a standard table with 100 records, each storing a 1 MB product image directly inside the table row.
		
		- To read just **one** record, the database must load a 1 MB image into its cache.
		    
		- Because the cache is only, say, 100 MB, it can only hold 100 images at a time, instead of millions of small text records.
		    
		- **Result:** The cache fills up instantly with massive graphics, throwing out all the small, frequently accessed text data (like customer names or order IDs). Performance grinds to a halt because the database is constantly swapping massive files in and out of memory and reading huge chunks from slow disk drives.

	3. The Solution: Physical Separation (Binary Objects): To fix this, database creators introduced **Binary Objects**. The key concept is **physical separation**.
		- The **table record** stores only the regular small data (e.g., `ProductID = 101`, `ProductName = "Shirt"`) and a **pointer** (a reference key) to the binary file.
		
		- The **Binary Object** (BLOB) stores the massive graphic (or sound file, video, XML document) in a **completely separate storage area** outside the table's main block structure.
		    
	4. The Advantages of This Separation
		- **Table performance is protected:** When the database queries the table for `ProductID` and `ProductName`, it only reads tiny 2 KB blocks. Its cache stays efficient and fast.
		    
		- **Binary objects are streamed separately:** When the application actually _needs_ the graphic, it retrieves it directly from the BLOB storage area. It doesn't pollute the core table cache.
		    
		- **Flexibility:** You can store _anything_ in binary format—graphics, sound, video, large text (like XML or JSON documents), or compressed files.

2. Reference pointers —In the C programming language, a reference pointer is a variable containing an address on disk or in memory of whatever the programmer wants to point at.
3. Collection arrays —Some relational databases allow creation of what an object database would call a collection. A collection is a set of values repeated structurally (values are not necessarily the same) where the array is contained within another object, and can only be referenced from that object. In the case of a relational database, the containment factor is the collection being a field in the table. Collection arrays can have storage structures defined in alternative locations to table fields as for binary objects, but do not have to be as such. Collection arrays, much like program arrays, can be either fixed length or dynamic. A dynamic array is a variable-length array, and is actually a pointer. When using a fixed-length array, the programmer must specify the length of the array before using it.
4. User-defined types—Some relational databases allow programmable or even on-the-fly creation of user-defined types. A user-defined type allows the creation of new types. Creation of a new type means that user-defined datatypes can be created by programmers, even using other user-defined types. It follows that fields can be created in tables where those fields have user-defined datatypes.