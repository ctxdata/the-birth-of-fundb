## Play with our dummy storage engine
After [previous step](./env.md), now we can play with our FUN storage engine in MySQL Server. 

### Create a Table to paly with

```SQL
mysql> CREATE TABLE t1(id INT PRIMARY KEY, comments BIGINT NOT NULL) ENGINE = FUN;
```

Oops, we got an error which doesn't allow us to create a primary key. <br/>
`ERROR 1069 (42000): Too many keys specified; max 0 keys allowed`

I guess there might be some flags I need to specify in my storage engine, however I'm a newbie to MySQL Internals. I tried to figure out in my own `stupid` way. <br/>
I think `Too many keys specified;` should be a specific error message, so I grep this message in source code by below command

```Bash
mysql-8.0.26 % grep -r "ER_TOO_MANY_KEYS" .
Binary file ./bld2/library_output_directory/libserver_unittest_library.dylib matches
./bld2/include/mysqld_errmsg.h:#define ER_TOO_MANY_KEYS_MSG "Too many keys specified; max %d keys allowed"
./bld2/include/mysqld_ername.h:{ "ER_TOO_MANY_KEYS", 1069, "Too many keys specified; max %d keys allowed","42000", "S1009", 57 },
./bld2/include/mysqld_error.h:#define ER_TOO_MANY_KEYS 1069

...
```

So I guess `ER_TOO_MANY_KEYS` is the error code for this specific situation, I grep this error code in source code:
```Bash
mysql-8.0.26 % grep -r "ER_TOO_MANY_KEYS" -n ./sql
./sql/sql_table.cc:7935:    my_error(ER_TOO_MANY_KEYS, MYF(0), file->max_keys());
```

Now, we can review source code directly or use a debugger to debug, the error is thrown at this code snippet

```c++
  if (*key_count > file->max_keys()) {
    my_error(ER_TOO_MANY_KEYS, MYF(0), file->max_keys());
    return true;
  }
```

This code snippet is quite simple and straightfoward, we can easily identify `handler::max_keys()` -> `handler::max_supported_keys()` -> `ha_fun::max_supported_keys()`.
```c++
uint max_supported_keys() const override { return 1; }
```

Stop `mysqld` and rebuild source code by `make -j 4`, at meantime I'm quite proud that I fixed my first problem. However...

```SQL
mysql> CREATE TABLE t1(id INT PRIMARY KEY, comments BIGINT NOT NULL) ENGINE = FUN;
ERROR 1070 (42000): Too many key parts specified; max 0 parts allowed
```
Why it didn't work? How poor my eyesight is! It does change from error 1069 to 1070, but error messages look too similar.. :(

Let me try my old tricks, `handler::max_key_parts()` -> `handler::max_supported_key_parts()` -> `ha_fun::max_supported_key_parts()`, done! Wait!! Occasionally...I read the comment of this method `There is no need to implement ..._key_... methods if your engine doesn't support indexes.`. Support index?? No!No!!No!!! At least not now. 
    
So, I think the best way for now is to avoid using `key` in our tables with our dummy storage engine. By this way, we can continue to play with it. But I believe we'll be able to support index in FunDB sooner or later. :) I guess so

```SQL
mysql> CREATE TABLE t1(id INT NOT NULL, comments BIGINT NOT NULL) ENGINE = FUN;
Query OK, 0 rows affected (3.63 sec
```

Cool!!! A big step, isn't it?

### Query data from our table

```SQL
mysql> SELECT * FROM t1;
Empty set (0.02 sec)
```

Am I expecting to fetch data from this table? I must be insane now... :)

### Insert data into table

I cannot wait to insert some data into my table, here we go.
```SQL
mysql> INSERT INTO t1 VALUES(1, 100);
```

`ERROR 1662 (HY000): Cannot execute statement: impossible to write to binary log since BINLOG_FORMAT = ROW and at least one table uses a storage engine limited to statement-based logging.`

Let's check the binlog format
```SQL
mysql> show variables like 'binlog_format';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.01 sec)
```

Hmmm, it seems our storage engine doesn't support or we don't declare we support row format for binlog. There could be many ways to fix this issue:

* Change binlog format for current session
```SQL
mysql> SET SESSION binlog_format = 'MIXED';
Query OK, 0 rows affected (0.00 sec)
```

* Start MySQL Server with mixed binlog format
```Bash
./mysqld -u root --binlog-format=MIXED &
```

* Declare that our storage engine is compatible with binlog both ROW and STMT format
```c++
  ulonglong table_flags() const override {
    return ( HA_BINLOG_ROW_CAPABLE | HA_BINLOG_STMT_CAPABLE );
  }
```

Each of them will make `insert` working for our storage engine. 
```SQL
mysql> INSERT INTO t1 VALUES(1, 100);
Query OK, 1 row affected (0.00 sec)

mysql> SELECT * FROM t1;
Empty set (0.00 sec)
```

`INSERT` succeeded but we still cannot fetch data from our storage engine, because we didn't save the data. 

So I guess next step we'll create a data file and store rows into data file in `write_row`.
