### Read all rows from FunDB

We can store our data into data file, and we can get them by `hexdump`, but we still cannot read them through `SQL`. After this article, we can `INSERT` and `SELECT` data from `FunDB` storage engine. 

```SQL
SELECT * FROM t1;
```

This time, we'll return all rows like below, in the future we may merge `val` as a List of integers.

```SQL
mysql> SELECT * FROM t1;
+----+--------+
| id | val    |
+----+--------+
|  3 |   1002 |
|  1 |   1002 |
|  1 |  13458 |
|  1 |   1024 |
|  1 | 102400 |
|  1 |    256 |
|  1 |    255 |
|  1 |    666 |
|  2 |   1020 |
|  2 | 102345 |
+----+--------+
```

We may or may not impelment the output like below (This's not that important from my perspective. :))

|id |val  |
--- | --- |
|1|1002,13458,1024,102400,256,255,666|
|2|1020,102345|
|3|1002|

### Code snippets to implement `SELECT` all rows

You may refer to official document [Implementing Basic Table Scanning](https://dev.mysql.com/doc/internals/en/implementing-basic-table-scanning.html).

>The following shows the method calls made during a nine-row table scan of the CSV engine:
```
ha_tina::store_lock
ha_tina::external_lock
ha_tina::info
ha_tina::rnd_init
ha_tina::extra - ENUM HA_EXTRA_CACHE   Cache record in HA_rrnd()
ha_tina::rnd_next
ha_tina::rnd_next
ha_tina::rnd_next
ha_tina::rnd_next
ha_tina::rnd_next
ha_tina::rnd_next
ha_tina::rnd_next
ha_tina::rnd_next
ha_tina::rnd_next
ha_tina::extra - ENUM HA_EXTRA_NO_CACHE   End caching of records (def)
ha_tina::external_lock
ha_tina::extra - ENUM HA_EXTRA_RESET   Reset database to after open
```

At this point, `rnd_init` and `rnd_next` are what we need to take care of. `rnd_int` is similar as `vector<int>::iterator it = v.begin()`, it initializes an iterator to iterate through all rows in the table. And `rnd_next` is similar as `it++` which move iterator to next record.

### Data structures for FunDB storage and record iteration

Below data structures are for metadata file.

```C++
#define FUNDB_RESERVED_STRING "FunDB@2022"
#define FDB_EXT ".fdb"
#define FMD_EXT ".fmd"

typedef struct metadata_header {
    char reserved[12];
    uint32_t count;
} metadata_header;

typedef struct metadata_data {
    uint32_t id;
    uint32_t offset;
    uint32_t length;
} metadata_data;

```

And we need a dedicate class to manipulate data for a FunDB table, here we go

```C++
/**
 * @brief This is memory buffer representative of row data
 * 
 */
typedef struct fundb_data_buffer {
    uint32_t id;
    uint32_t count;
    uint32_t data[1024];
    struct fundb_data_buffer* next;
} fundb_data_buffer;

class fundb_table {
private:
    fundb_data_buffer* data;
    uint32_t rows;
    string tableName;
public:
    typedef row_iterator iterator;
    const fundb_table::iterator last;
    fundb_table(const string& table): data(nullptr), last(nullptr), rows(0), tableName(table) {}
    bool add(uint32_t id, uint32_t val);
    int remove(uint32_t id, uint32_t val);

    void open();

    const fundb_table::iterator& begin();
    const fundb_table::iterator& end();

    int create();

    int drop();

    void close();
};
```

From `fundb_data_buffer` declaration, we have a limit for the List size of `val` field in FunDB. We can hold at most `1024` integers for a specific `id`.

And of course an iterator to go through all rows is required.

```C++
class row_iterator{
private:
    fundb_data_buffer* data;
    uint32_t idx;
public:
    row_iterator(): data(nullptr), idx(0) {}
    row_iterator(fundb_data_buffer* ptr) : data(ptr), idx(0) {}
    tuple<uint32_t, uint32_t> operator*() {
        if (data == nullptr || idx >= data->count) {
            return std::make_tuple(-1, -1);
        }

        return std::make_tuple(data->id, data->data[idx]);
    }

    row_iterator& operator++(int) {
        if (++idx >= data->count) {
            data = data->next;
            idx = 0;
        }

        return *this;
    }

    bool operator==(const row_iterator& rit) {
        if (data == rit.data && idx == rit.idx) {
            return true;
        }

        return false;
    }
};
```

The `iterator` is simple and designed for our specific scenario. It's responsible to iterate all value for an `id`, and iterate all `id` in the table. With this iterator we can have below output.

```SQL
mysql> SELECT * FROM t1;
+----+--------+
| id | val    |
+----+--------+
|  3 |   1002 |
|  1 |   1002 |
|  1 |  13458 |
|  1 |   1024 |
|  1 | 102400 |
|  1 |    256 |
|  1 |    255 |
|  1 |    666 |
|  2 |   1020 |
|  2 | 102345 |
+----+--------+
```

### Implementation of rnd_init and rnd_next
With `fundb_table` and `fundb_tablel::itrator`, we can implement our rnd_init and rnd_next like below.

```C++
int ha_fun::rnd_init(bool scan) {
  DBUG_ENTER("ha_fun::rnd_init");
  stats.records = 0;
  records_is_known = false;

  share->it = share->fun_table->begin();

  DBUG_RETURN(0);
}

int ha_fun::rnd_next(uchar *buf) {
  DBUG_ENTER("ha_fun::rnd_next");

  if (HA_ERR_END_OF_FILE == find_current_row(buf) )
     DBUG_RETURN(HA_ERR_END_OF_FILE);

  stats.records++;
  DBUG_RETURN(0);
}

int ha_fun::find_current_row(uchar *buf) {
  DBUG_ENTER("ha_fun::find_current_row");
  
  if (share->it == share->fun_table->end()) {
    DBUG_RETURN(HA_ERR_END_OF_FILE);
  }

  my_bitmap_map *org_bitmap;
  org_bitmap = dbug_tmp_use_all_columns(table, table->write_set);
  std::tuple<uint32_t, uint32_t> row = *share->it;
  uint32_t id = std::get<0>(row);
  uint32_t val = std::get<1>(row);

  String buffer(16);

  for (Field **field = table->field; *field; field++) {
    buffer.length(0);
    if (strcmp("id", (*field)->field_name) == 0) {
      buffer.append_longlong(id);
      (*field)->store(buffer.c_ptr(), buffer.length(), (*field)->charset(), CHECK_FIELD_WARN);
    } else {
      buffer.append_longlong(val);
      (*field)->store(buffer.c_ptr(), buffer.length(), (*field)->charset(), CHECK_FIELD_WARN);
    }
  }

  share->it++;
  dbug_tmp_restore_column_map(table->write_set, org_bitmap);
  DBUG_RETURN(0);
}
```

Above code snippet is not difficult, a few things I gonna mention:
 1. First thing is `rnd_next` is something opposite of `write_row`. In `write_row`, we get value from `*field` (which is MySQL row format) and store them into files on the disk. In `rnd_next` we need to set value to `*field` so that MySQL can return them back to client.
 2. We'd tell MySQL query engine that we have got all records we have by returnning `HA_ERR_END_OF_FILE`.

```C++
  if (share->it == share->fun_table->end()) {
    DBUG_RETURN(HA_ERR_END_OF_FILE);
  }
```