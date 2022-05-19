## Prepare development environment

### Setup CMake in your environment
Refer to MySQL Documentation, [Install MySQL from source](https://dev.mysql.com/doc/refman/8.0/en/source-installation.html), 
and [Build MySQL with CMake](https://dev.mysql.com/doc/internals/en/cmake.html) <br/>

### Build source code
After the source code is ready and build tools are ready, use below command to initialize makefile (Suppose code is located at ~/mysql-8.0)
```bash
% cd ~/mysql-8.0
% mkdir bld && cd bld
% cmake .. -DCMAKE_INSTTALL_PREFIX=~/mysql-8.0 -DWITH_DEBUG=1 -DFORCE_INSOURCE_BUILD=1 -DMYSQL_DATADIR=../data -DWITH_BOOST=../libboost/boost_1_73_0
```
After this step, we can build MySQL with `make` command now
```bash
% make -j 4
```

### Start self-built MySQL Server
After the compilation, there would be a `bin` shortcut folder with all binaries including `mysqld`, `mysql` etc. You may initialize MySQL database and then you can simply start MySQL server process
```bash
% ./mysqld --initialize --user=mysql
% ./mysqld -u mysql &
```
We can connect to this MySQL Server with `mysql` tool.
```bash
% ./mysql -u root -p
```

Now, we can use interact with MySQL as usual, you may need to update your password for user `mysql` after your first login.
```SQL
ALTER USER USER() IDENTIFIED BY 'auth_string';
```

## Create custom storage engine
After MySQL is working as expected, we can create custom storage engine source files accroding to [Create Storage Engine source files](https://dev.mysql.com/doc/internals/en/custom-engine-source-files.html)

```bash
% cd ~/mysql-8.0/storage && mkdir fundb
% sed -e s/EXAMPLE/FUN/g -e s/example/fun/g -e s/Example/Fun/g ./example/ha_example.h > ./fundb/ha_fun.h
% sed -e s/EXAMPLE/FUN/g -e s/example/fun/g -e s/Example/Fun/g ./example/ha_example.cc > ./fundb/ha_fun.cc
% sed -e s/EXAMPLE/FUN/g -e s/example/fun/g -e s/Example/Fun/g ./example/CMakeFiles.txt > ./fundb/CMakeFile.txt
```

You can name your own custom storage engine with any name.

Execute `cmake` command to configure makefile again, and then execute `make -j 4`. For details, refer to [Storage Engine options](https://dev.mysql.com/doc/refman/8.0/en/source-configuration-options.html#option_cmake_storage_engine_options) <br/>
You can build your storage engine in dynamic library and [INSTALL PLUGIN](https://dev.mysql.com/doc/refman/8.0/en/install-plugin.html). Or you can build the storage engine in static library like what I did with `-DWITH_FUN_STORAGE_ENGINE=1`
```bash
% cmake .. -DCMAKE_INSTTALL_PREFIX=/Users/maike/mysql-8.0/tmp -DWITH_DEBUG=1 -DFORCE_INSOURCE_BUILD=1 -DMYSQL_DATADIR=../data2 -DWITH_BOOST=../libboost/boost_1_73_0 -DWITH_FUN_STORAGE_ENGINE=1
```

Now, we can execute `show engines;` in `mysql` command prompt, it will display our storage engine.
``` bash
mysql> show engines;
```

| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
--- | --- | --- | --- | --- | --- |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| **FUN**            | YES     | **Fun storage engine**                                         | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |

Wow, though we did very few things, we can create table with `FUN` storage engine now. A big step for me. :)

Next, I'll create a table with FUN storage engine, and play with it...

Next: [Play with our dummy storage engine](./play.md)
