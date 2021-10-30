# API INSTRUCTION

To distinguish File and Index Manager API and global API(such as init_db and shutdown_db), copy File and Index  Manager API to api file. In global API, function calls corresponding layer api call.

-----
## API Functions
1. int64_t open_table(char *pathname)

- Open the database file.

Open existing data file using 'pathname' or create one if not existed.
If a new file needs to be created, the default file size should be 10 MiB
If success, return the unique table id, which represents the own table in this database.
Otherwise, return negative value.

- parameters
  - pathname - pathname to database file

- return value - table id of the opened database file
- exceptions
  - if error occurred in Sub Layer API, print error message and return -1
---
2. int db_insert(int64_t table_id, int64_t key, char *value, uint16_t val_size)

- Insert a record.

Insert input key and value record with its size to data file at the right place.
If success, return 0. Otherwise, return non zero value.

Duplicated key is not allowed. Only non-existed key value record can be inserted. Otherwise, API return non zero value.

Check violating the B+ tree properties and modify tree structure to ensure occupancy invariant if it needed.

- parameters
  - table_id - table id of the opened database
  - key - record key
  - value - record value
  - val_size - record value's length

- return value - status code(0 is ok)
- exceptions
  - if error occurred in File and Index manager module or Sub Layer API, print error message and return -1
---
3. int db_delete(int64_t table_id, int64_t key)

- Delete a record

Find the matching record and delete it if found.
If success, return 0. Otherwise, return non zero value.

Only existed key value record can be deleted. Otherwise, API return non zero value.

Check violating the B+ tree properties and modify tree structure to ensure occupancy invariant if it needed.

- parameters
  - table_id - table id of the opened database
  - key - record key

- return value - status code(0 is ok)
- exceptions
  - if error occurred in File and Index manager module or Sub Layer API, print error message and return -1
---
4. int db_find(int64_t table_id, int64_t key, char *ret_val, uint16_t *val_size);

- Find a record.

Find the record containing input key.
If found matching key, store matched value string in ret_val and matched size in val_size.
If success, return 0. Otherwise, return non zero value.
The caller should allocate memory for a record structure.

Only existed key value record can be found. Otherwise, API return non zero value.

- parameters
  - table_id - table id of the opened database
  - key - record key to find
  - value - destination memory of record value
  - val_size - record value's length

- return value - status code(0 is ok)
- exceptions
  - if error occurred in File and Index manager module or Sub layer API, print error message and return -1
---
5. int init_db(num_buf)

- Initialize DBMS

Initialize database management system.
If success, return 0. Otherwise, return non zero value.

Before starting DBMS, each layer initialize their status if needed.
In Buffer Manager, call init_buffer(allocate the buffer pool). 

- parameters
  - num_buf - size of buffer (num of buffer page)

- return value - status code(0 is ok)
- exceptions - (none)
---
6. int shutdown_db()

- Shutdown DBMS

Shutdown your database management system.
If success, return 0. Otherwise, return non zero value.

Before shutdown DBMS, each layer finish and save their status if needed.
In Buffer Manager, flush all dirty page by calling buffer_close_table_file API.
In Disk Space Manager, close all file by calling file_close_table_file API.

- parameters - (none)
- return value - status code(0 is ok)
- exceptions
  - if error occurred in Sub Layer API, print error message and return -1
