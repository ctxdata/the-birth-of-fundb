## Welcome to visit `The birth of FunDB`

### Step by step to create a MySQL custom storage engine

This is the place where I write down my ideas I come up with, issues I met and problems I solved during the period of writing the experiment MySQL Storage Engine.

I hope you can enjoy every word in `The birth of FunDB`, :)

### The main feature of this storage engine
I hope a talbe with FunDB will behave like Redis List value, the table will contain only two fields, id and value. 'id' is a typical INT AUTO INCREMENT PRIMARY KEY, while value is a List of Long which may present a stream of IDs for other tables.

|id |messages  |
--- | --- |
|1|12,34,118,874|
|2|443,8080|

So a SQL statement like `INSERT INTO tbl_user_messages VALUE(1, 8081)`, will be turn into an update to the row with `id=1`, and the logical representation of table will be

|id |messages  |
--- | --- |
|1|12,34,118,874,<b>8081</b>|
|2|443,8080|

### The plan
1. Prepare development environment
2. Create initial source code, and make sure MySQL loads the storage engine.
3. Try to create data file when creating a table (CREATE TABLE .. ENGINE = FUN)
4. Write row data into data file (INSERT INTO tbl VALUES(1, 1000))
5. TBD...

### Step by step to create MySQL custom storage engine
1.  [Prepare Development Environment](./env.md)
2.  [Play with our dummy storage engine](./play.md)
3.  
