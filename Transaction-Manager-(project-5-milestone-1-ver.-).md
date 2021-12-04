#  TRANSACTION MANAGER

The transaction manager manages transaction's lock list in transaction table and transaction log of multiple transaction.

By using transaction manager, multiple threads can begin, commit or abort(rollback) corresponding transaction. 

As we use strict two-phase locking pessimistic concurrency control to enforce conflict serializability,
releasing lock pass should be occurred when corresponding transaction is to commit or be aborted. 

In transaction manager, several API are provided for lock manager such as accessing transaction's last lock or appending lock object in transaction list.

In transaction manager, we use transaction manager latch to enforce transactions use transaction manager API once at time for atomicity of transaction table.

To find transaction object in transaction manager, transaction manager use hash table that maps trx_id to transaction object. As hash table expected to find such object in constant time if existed, we can access trx table fastly.

In transaction object, transaction manager maintains transaction lock list with linked list structure and transaction log list.

To successfully abort transaction and rollback all effects, transaction manager uses transaction log list of modification in current transaction.

To avoid deadlock problem between several manager-level latch, DBMS needs to break circular wait condition by acquiring lock in sequential order.

When several layer latches needed in one API, it should acquire multiple latch by following sequence.

Lock Manager Latch -> Transaction Manager Latch -> Buffer Manager Latch -> (Page Latch)

------

##  TRANSACTION MANAGER API

1. int init_trx_manager(void)

- initialize transaction manager.

Initialize transaction hash table for finding transaction entry and transaction manager latch.
Reset global transaction id pool to 0 for labeling first transaction to 1.

If success, return 0. Otherwise, return non zero value.

- parameters - (none)

- return value - status code(0 is ok)

- exceptions - (none)

---
2. int trx_begin_txn(void)

- Allocate a transaction structure and initialize it

This API requires transaction manager Latch.

Get a new transaction id and make a new entry in transaction hash table.

New transaction entry has empty log list and empty lock list.

- parameters - (none)

- return value - a unique transaction id (>= 1) if success, otherwise return 0

- exceptions - (none)

---
3. int trx_commit_txn(int trx_id)

- Commit the transaction

***This API requires both transaction manager Latch and lock manager Latch.***

In this API, it uses the lock manager API that requires explicitly acquiring lock manager latch.

Clean up the transaction and log list with the given transaction id and release all lock in transaction lock list (Shrinking phase of strict 2PL).

In invalid transaction(e.g. aborted transaction) case, just return 0.

- parameters
  - trx_id - id of transaction to be committed

- return value - the completed transaction id if success, otherwise return 0

- exceptions - (none)

---
4. int trx_abort_txn(int trx_id)

- Abort the transaction and rollback effects

***This API requires both transaction manager Latch and lock manager Latch.***

In this API, it uses the lock manager API that requires explicitly acquiring lock manager latch.

Rollback all effects made by given transaction by using transaction log list. Undo update operation in given transaction in reverse order by calling file and index manager API(update record without locking) because this transaction already acquired all locks for rollback operation.

Clean up the transaction and log list with the given transaction id and release all lock in transaction lock list (Shrinking phase of strict 2PL).

In invalid transaction(e.g. aborted transaction) case, just return 0.

- parameters
  - trx_id - id of transaction to be aborted

- return value - the aborted transaction id if success, otherwise return 0

- exceptions - (none)

---
5. int trx_append_lock_in_trx_list(int trx_id, lock_t* lock_obj)

- Append given lock object to given transaction lock list in transaction table

This API requires transaction manager Latch.

In invalid transaction(e.g. aborted transaction) case, just return 0.

- parameters
  - trx_id - transaction id of the transaction lock list where insertion occurred
  - lock_obj - lock object to be appended

- return value - the transaction id if success, otherwise return 0

- exceptions - (none)

---
6. int trx_remove_lock_in_trx_list(int trx_id, lock_t* lock_obj)

- Remove given lock object to given transaction lock list in transaction table

This API requires transaction manager Latch.

In invalid transaction(e.g. aborted transaction) case, just return 0.

- parameters
  - trx_id - transaction id of the transaction lock list where insertion occurred
  - lock_obj - lock object to be appended

- return value - the transaction id if success, otherwise return 0

- exceptions - (none)

---
7. int trx_add_log(int64_t table_id, pagenum_t page_id, int64_t key, uint32_t slot_number, char *new_values, uint16_t new_val_size, char *old_values, uint16_t old_val_size, int trx_id)

- Append update log to transaction log list

This API requires transaction manager Latch.

Make new update log object to transaction log list which means record[key]'s value changed from old_values to new_values.

In invalid transaction(e.g. aborted transaction) case, just return 0.

- parameters
  - table_id - table where update operation occurred
  - page_id - page number where target record reside
  - key - key of target record
  - slot_number - slot number where target record reside
  - new_values - new value that changed to
  - new_val_size - length of new value string
  - old_values - old value that changed from
  - old_val_size - length ofold value string
  - trx_id - id of transaction that makes this update operation

- return value - the transaction id if success, otherwise return 0

- exceptions - (none)

---
8. lock_t* trx_get_last_lock_in_trx_list(int trx_id)

- Get last lock object in given transaction lock list

This API requires transaction manager Latch.

Get tail lock object in given transaction lock list.

In invalid transaction(e.g. aborted transaction) case, just return 0.

- parameters
  - trx_id - transaction id of the transaction target lock list

- return value - lock's pointer if success, otherwise return null

- exceptions - (none)

---
9. int trx_is_this_trx_valid(int trx_id)

- Check given transaction id is valid

This API requires transaction manager Latch.

Check given transaction is alive or invalid (e.g. aborted or committed)
by using transaction table.

If error occurred, just return -1.

- parameters
  - trx_id - transaction id to be checked

- return value - status code(1 is alive, 0 is dead(invalid), -1 is error case)

- exceptions - (none)

---
10. void close_trx_manager()

- Rollback all uncommitted result and destroy transaction table

***This API requires both transaction manager Latch and lock manager Latch.***

In this API, it uses the lock manager API that requires explicitly acquiring lock manager latch.

Abort all uncommitted transaction in transaction table.

In each transaction, rollback all results, release all locks and remove log.

After aborting, delete transaction table.

- parameters - (none)

- return value - (none)

- exceptions - (none)

---
## TRANSACTION OBJECT FORMAT
1. TRANSACTION (trx_t)

Transaction object is the entry of transaction table that contains transaction information.
- std::vector<trx_log_t*> trx_log
  - log list of update(modification) in current transaction
- lock_t* nxt_lock_in_trx
  - next(head) lock of transaction lock list
- lock_t* last_lock_in_trx
  - last(tail) lock of transaction lock list
- int trx_id
  - identifier of entry

2. TRANSACTION LOG (trx_log_t)

Transaction log object that contains update information.
- char *old_value
  - point to old value string
- char *new_value
  - point to new value string
- int64_t table_id
  - target table
- pagenum_t page_id
  - target page number
- int64_t key
  - target record key
- uint32_t slot_number
  - target slot number
- uint16_t old_size
  - old value string length
- uint16_t new_size
  - new value string length

---
## TM module(TM namespace)
use for Transaction Manager Layer ONLY!!(DON'T USE IN OTHER LAYERS)

inner structure and function used in Transaction Manager

- pthread_mutex_t transaction_manager_latch = PTHREAD_MUTEX_INITIALIZER
  - global transaction manager latch

- std::unordered_map<int, trx_t> trx_table
  - hash table that mapping transaction id to transaction object in the table
  - search key is transaction id
  - corresponding value is transaction object

- bool is_trx_valid(int trx_id)
  - check given transaction id is valid by finding corresponding entry in table

- lock_t* find_first_lock_in_trx_table(int trx_id)
  - get first lock in lock list of given trx_id's transaction table entry
  - return lock object pointer or null if not found

- lock_t* find_last_lock_in_trx_table(int trx_id)
  - get last lock in lock list of given trx_id's transaction table entry
  - return lock object pointer or null if not found

- int make_new_txn_id()
  - get new id for new transaction in transaction begin
  - return new id

- void insert_new_trx_in_table(int trx_id)
  - insert new transaction entry in transaction table at the given id position

- void erase_trx_in_table(int trx_id)
  - delete transaction entry in transaction table

void append_lock_in_table(int trx_id, lock_t* lock_obj)
  - append lock object in transaction lock list of given trx_id's transaction table entry

- TM::trx_log_t* make_trx_log(int64_t table_id, int64_t key, char *new_values, uint16_t new_val_size, char *old_values, uint16_t old_val_size)
  - make and initialize trx_log object using given information
  - return new trx_log object's pointer

- void append_log_in_list(int trx_id, TM::trx_log_t* log_obj)
  - append given trx_log object into given transaction's log list

- void remove_trx_log(int trx_id)
  - destroy given transaction's log

- void release_all_locks_in_trx(int trx_id)
  - release all locks in trx lock list of given trx_id's trx table entry

- void rollback_trx_log(int trx_id)
  - rollback all effects made by given transaction by using log list in corresponding transaction table entry