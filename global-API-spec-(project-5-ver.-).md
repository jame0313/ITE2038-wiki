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
---
7. int db_find(int64_t table_id, int64_t key, char *ret_val, uint16_t *val_size, int trx_id)

- Find a record.

Read a value in the table with a matching key for the transaction having transaction id
If found matching key, store matched value string in ret_val and matched size in val_size.
If success, return 0. Otherwise, return non zero value.
The caller should allocate memory for a record structure.

Only existed key value record can be found. Otherwise, API return non zero value.

return 0 (SUCCESS): operation is successfully done, and the transaction can continue the next operation.
return non zero (FAILED): operation is failed (e.g., deadlock detected), and the transaction should be aborted.

- parameters
  - table_id - table id of the opened database
  - key - record key to find
  - value - destination memory of record value
  - val_size - record value's length
  - trx_id - id of transaction which called this api 

- return value - status code(0 is ok)
- exceptions
  - if error occurred in File and Index manager module or Sub layer API, print error message and return -1
---
8. int db_update(int64_t table_id, int64_t key, char *values, uint16_t new_val_size, uint16_t *old_val_size, int trx_id)

- Update a record.

Find target record and modify the values for the transaction having transaction id
If found matching key, update the value of the record to values string with its new_val_size and store its size in old_val_size
If success, return 0. Otherwise, return non zero value.
The caller should allocate memory for a old_val_size

Only existed key value record can be found. Otherwise, API return non zero value.

return 0 (SUCCESS): operation is successfully done, and the transaction can continue the next operation.
return non zero (FAILED): operation is failed (e.g., deadlock detected), and the transaction should be aborted.

- parameters
  - table_id - table id of the opened database
  - key - record key to find
  - values - new value string to be applied into record
  - new_val_size - new value string's length
  - old_val_size - existed record value's length
  - trx_id - id of transaction which called this api 

- return value - status code(0 is ok)
- exceptions
  - if error occurred in File and Index manager module or Sub layer API, print error message and return -1
---
9. int trx_begin(void)

- Begin new transaction

Allocate a transaction structure and initialize it
Return a unique transaction id (>= 1) if success, otherwise return 0.

- parameters - (none)

- return value - new transaction id, otherwise return 0
- exceptions - (none)
---
10. int trx_commit(int trx_id)

- Commit and end the transaction

Clean up the transaction with the given trx_id (transaction id) and its related information that has been used in your lock manager.

This is shrinking phase of strict 2PL.

Return the completed transaction id if success, otherwise return 0.

- parameters
  - trx_id - id of transaction to be committed

- return value - completed trx id if success, otherwise return 0
- exceptions - (none)