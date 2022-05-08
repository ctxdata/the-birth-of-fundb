## Store rows into data file

### Create a data file when a FUN Storage Engine table is created

Though, memory storage engine might be a better way as our next step for a dummy storage engine. I'd try a difficult and further step to store rows into data file.

So, the first step is to create a data file when a table is created by `CREATE TABLE` statement.
```SQL
CREATE TABLE t1(id INT NOT NULL, val BIGINT NOT NULL) ENGINE = FUN;
```

Refer to MySQL Internals doc [Creating Tables](https://dev.mysql.com/doc/internals/en/creating-tables.html)

Once MySQL Server receives `CREATE TABLE` command from users, MySQL will parse and check validity of the statement and then invoke storage engine's `create` method

We need to add some code to create a data file to store rows for this new table
```C++
int ha_fun::create(const char *name, TABLE *table_arg, HA_CREATE_INFO *,
                       dd::Table *) {
  File create_file;
  char name_buff[FN_REFLEN];
  
  DBUG_TRACE;
  DBUG_ENTER("ha_fun::create");

  /*
    check columns, we don't allow any column to be nullable
  */
  for (Field **field = table_arg->s->field; *field; field++) {
    if ((*field)->is_nullable()) {
      my_error(ER_CHECK_NOT_IMPLEMENTED, MYF(0), "nullable columns");
      DBUG_RETURN(HA_ERR_UNSUPPORTED);
    }
  }

  if ((create_file= my_create(fn_format(name_buff, name, "", ".fdb",
        MY_REPLACE_EXT|MY_UNPACK_FILENAME),0,
        O_RDWR | O_TRUNC,MYF(MY_WME))) < 0)
  DBUG_RETURN(-1);

  my_close(create_file, MYF(0));
  DBUG_RETURN(0);
}
```

With this code, now `t1.fdb` file will be created after the CREATE TABLE as mentioned is executed.
```bash
mysql-8.0.26 % ls -l ./data/FUNDB/t1* 
-rw-r-----@ 1 **  staff     0 May  8 18:18 ./data/FUNDB/t1.fdb
-rw-r-----  1 **  staff  2774 May  8 18:18 ./data/FUNDB/t1_377.sdi
```

### Append rows into data file
Once we have a data file for each table, we'd write rows into data file for each INSERT operation.

```SQL
mysql> INSERT INTO t1 VALUES(1, 100000);
Query OK, 1 row affected (4.22 sec)
```

We expect "1,100000" to be written as a single line in our data file.
```bash
mysql-8.0.26 % cat ./data/fundb/t1.fdb
1,100000

mysql-8.0.26 % hexdump ./data/fundb/t1.fdb
0000000 2c31 3031 3030 3030 000a               
0000009

mysql-8.0.26 % hexdump -c ./data/fundb/t1.fdb
0000000   1   ,   1   0   0   0   0   0  \n                            
0000009
```

According to [Adding Support for INSERT to a Storage Engine](https://dev.mysql.com/doc/internals/en/support-for-insert.html)
All `INSERT` operations are handled by `handler::write_row` method, our implementation looks like below:
```C++
int ha_fun::write_row(uchar *buf) {
  DBUG_ENTER("ha_fun::write_row");
  
  ha_statistic_increment(&System_status_var::ha_write_count);

  char attribute_buffer[1024];
  String attribute(attribute_buffer, sizeof(attribute_buffer), &my_charset_bin);

  my_bitmap_map *org_bitmap = dbug_tmp_use_all_columns(table, table->read_set);
  buffer.length(0);

  for (Field **field = table->field; *field; field++) {
    (*field)->val_str(&attribute, &attribute);
    buffer.append(attribute);
    buffer.append(',');
  }

  // replace the extra ',' into '\n' to make sure each row is stored within a single line
  buffer[buffer.length()-1] = '\n';
  dbug_tmp_restore_column_map(table->read_set, org_bitmap);

  if (!share->fd_write_opened)
    if (init_fd_writer()) 
      DBUG_RETURN(-1);

  if (mysql_file_write(share->fundb_write_fd,
                       pointer_cast<const uchar *>(buffer.ptr()), buffer.length(),
                       MYF(MY_WME | MY_NABP)))
    DBUG_RETURN(-1);

  DBUG_RETURN(0);
}
```

We'll convert all fields value into a String by joining with a comma, so we reserve a buffer `attribute_buffer` to store row data.

`table->field` holds all fields and value, we can iterate to concat all fields into a String.
```C++
  for (Field **field = table->field; *field; field++) {
    (*field)->val_str(&attribute, &attribute);
    buffer.append(attribute);
    buffer.append(',');
  }
```

And then check whether the data file is open and ready for writing, open the data file.
```C++
  if (!share->fd_write_opened)
    if (init_fd_writer()) 
      DBUG_RETURN(-1);
```

We open the data file with `O_APPEND` mode, so new rows will be appended at the end of the data file
```C++
int ha_fun::init_fd_writer() {
  DBUG_TRACE;

  if ((share->fundb_write_fd =
           mysql_file_open(fundb_key_file_data, share->data_file_name,
                           O_RDWR | O_APPEND, MYF(MY_WME))) == -1) {
    DBUG_PRINT("info", ("Could not open tina file writes"));
    share->crashed = true;
    return my_errno() ? my_errno() : -1;
  }
  share->fd_write_opened = true;

  return 0;
}
```

Once the data file is opened for append, we'll write the `buffer` String of row data into data file

```C++
  if (mysql_file_write(share->fundb_write_fd,
                       pointer_cast<const uchar *>(buffer.ptr()), buffer.length(),
                       MYF(MY_WME | MY_NABP)))
    DBUG_RETURN(-1);
```

Now, we can save rows into our *.fdb data file.

Next time, we'll try to read data from data file.
