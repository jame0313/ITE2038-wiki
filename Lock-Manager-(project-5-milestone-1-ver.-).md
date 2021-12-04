#  LOCK MANAGER

The lock manager manages page lock list of record lock object of multiple threads.

By using lock manager, multiple thread's operation can be run concurrently by providing conflict serializable schedule.

As we use strict two-phase locking pessimistic concurrency control to enforce conflict serializability,
we can get lock when there is no conflicting lock only or have to wait for this situation.

Releasing lock should be occurred when corresponding transaction is to commit or be aborted. 

By using transaction manager, lock manager detects deadlock situation and abort new lock request.

In lock manager, we use lock manager latch to enforce transactions use lock manager API once at time for atomicity of lock table. 

To find page lock in lock manager, lock manager use hash table that maps page_id(table id and page number) to lock header object. As hash table expected to find such object in constant time if existed, we can access lock table fast.

In page lock list, lock manager maintains record lock objects with linked list structure.

In record lock object, there is two record level lock mode ( Shared(S)/Exclusive(X) ).
Shared lock should be acquired before accessing(reading) record's contents and
Exclusive lock should be acquired before updating(writing) record's contents.

To reduce duplicated lock, we allocate new lock object only if requesting transaction requests that kind of lock at first time. This means when same transaction request same record with compatible mode(lock mode is same or the acquired lock was exclusive lock and requesting lock is shared lock) lock manager just return previous lock object.

Different transaction case table

|  | S(shared) | X(exclusive) |
| - | - | - |
| S | compatible | conflicted |
| X | conflicted | conflicted |

Same transaction case table

| previous lock \ requested lock | S(shared) | X(exclusive) |
| - | - | - |
| S | compatible | not compatible |
| X | compatible | compatible |

------

##  LOCK MANAGER API

1. int init_lock_table(void)

- initialize lock manager.

Initialize lock hash table for finding page lock list and lock manager latch.

If success, return 0. Otherwise, return non zero value.

- parameters - (none)

- return value - status code(0 is ok)

- exceptions - (none)
---
2. int lock_acquire(int64_t table_id, pagenum_t page_id, int64_t key, uint32_t slot_number, int trx_id, int lock_mode)

- Allocate and append a new lock object of the record having the key to the corresponding page lock list of having page id

This API requires lock manager Latch.

By using lock table, find corresponding page lock list or make one if there is no such list.

Before finding conflicting lock, find compatible lock in the list. Compatible lock is previously acquired lock by same transaction which can share with requested type of lock. When there is compatible lock in the list, don't make new lock object and just return this compatible lock object.

The precise condition of compatible lock is that previous lock should be X lock or requested lock should be S lock. (This condition is described upon ***Same transaction case table***)

If there is no compatible lock, new lock object append to the end of page lock list and connect to transaction lock list in transaction table by using transaction manager API.

In this API, detect deadlock situation by checking existence of cycle in wait-for graph.
Find out-degree edge from current lock object in the graph and expand search pass like Depth-First Search(recursive function call).
In recursive call, use last lock object in transaction table by calling transaction manager API.
In wait-for graph, back transaction waits for front transaction when there is conflicting lock object.
Lock object can be conflicted with one X lock or several S locks. 

In this deadlock checking pass, return number of out-degree edges(number of conflicting lock object).
With this number, lock manager figure out existence of predecessor's conflicting lock object and wake up condition.

If there is a predecessor's conflicting lock object in the lock list, sleep until all predecessors releases its lock.
If there is no predecessor's conflicting lock object, return the address of the new lock object.

- parameters
  - table_id - table id of the opened database
  - page_id - page number of the corresponding record resides in
  - key - key of record
  - slot_number - slot number where target record reside in
  - trx_id - id of transaction that requests lock
  - lock_mode - type of lock (0(shared) or 1(exclusive))

- return value - 0 if success or -1 if error occurred (e.g. deadlock)

- exceptions - (none)

---
3. int lock_release(lock_t* lock_obj)

- Remove the lock_obj from the lock list

This API requires lock manager Latch.

Erase given lock object from corresponding page lock list and free object.

Find conflicting lock from current lock object (in-degree edge in wait-for graph) to wake the successor up.
If there is a successor’s lock waiting for the transaction releasing the lock, wake up the successor only if there is no conflicting lock object anymore after releasing this lock object.
In wait-for graph, back transaction waits for front transaction when there is conflicting lock object.
Lock object can be conflicted with one X lock or several S locks. 

- parameters
  - lock_obj - lock object pointer to be released

- return value - status code(0 is ok)

- exceptions - (none)

---
4. int lock_release_all(lock_t* lock_obj)

- Remove the all lock objects in transaction list from the lock list

This API requires lock manager Latch but ***NO locking lock manager latch***

***CALLER SHOULD LOCK LOCK MANAGER LATCH BEFORE CALLING THE API***

This is for avoiding deadlock between two manager(lock && transaction) latch.
As lock manager API uses transaction manager API that requires transaction manager latch, caller have to acquire lock manager latch first when both two manager latches needed.

Erase all lock object in transaction list from corresponding page lock list and free object.

This API is equivalent as several lock_release API call.

- parameters
  - lock_obj - first lock object pointer in the transaction list to be released

- return value - status code(0 is ok)

- exceptions - (none)

---
5. int lock_acquire_lock_manager_latch()

- acquire global lock manager latch

- parameters - (none)

- return value - status code(0 is ok)

- exceptions - (none)

---
6. int lock_release_lock_manager_latch()

- release global lock manager latch

- parameters - (none)

- return value - status code(0 is ok)

- exceptions - (none)

---
7. void close_lock_table()

- Destroy lock table

This API requires lock manager Latch.

Delete all lock header object and destroy lock hash table

- parameters - (none)

- return value - (none)

- exceptions - (none)

---
## LOCK OBJECT FORMAT (defined in page.h) AND OTHER FORMAT
1. LOCK (lock_t)

Lock object is the entry of page lock list that contains lock information.
- pthread_cond_t cond = PTHREAD_COND_INITIALIZER
  - conditional variable for waiting for conflicting lock release
- lock_t *prev_lock = nullptr
  - previous lock in page lock list
- lock_t *nxt_lock = nullptr
  - next lock in page lock list
- lock_t *prev_lock_in_trx = nullptr
  - prev lock in transaction list
- lock_t *nxt_lock_in_trx = nullptr
  - next lock in transaction list
- lock_head_t *sentinel = nullptr
  - lock header in page lock list
- int64_t record_id
  - record id that lock refer to
- int lock_mode = 0
  - lock mode (shared or exclusive)
- int owner_trx_id = 0
  - id of transaction which tries to acquire this lock 
- int waiting_num = 0
  - the number of conflicting lock (mark the lock is sleeping or not)

2. LOCK HEADER (lock_head_t)

Lock header in page lock list to access record-level lock entries
- int64_t table_id
- pagenum_t page_id
- lock_t *head = nullptr
  - first lock object in the list
- lock_t *tail = nullptr
  - last lock object in the list

3. PAGE ID
Page Identification Code to use for searching the lock list from hash table. This consists of table id and page number.
  - std::pair<int64_t, pagenum_t>
    - combine two property with pair

---
## LM module(LM namespace)
use for Lock Manager Layer ONLY!!(DON'T USE IN OTHER LAYERS)

inner structure and function used in Lock Manager

- pthread_mutex_t lock_manager_latch = PTHREAD_MUTEX_INITIALIZER
  - global lock manager latch

- std::unordered_map<page_id, lock_head_t*, LM::hash_pair> lock_table
  - hash table that mapping page id to page lock list in the table
  - search key is page_id({table_id, pagenum})
  - corresponding value is lock header's pointer
  - use LM::hash_pair to hash pair object

- lock_t* make_and_init_new_lock_object(int64_t table_id, pagenum_t page_id, int64_t key, uint32_t slot_number, int trx_id, int lock_mode)
  - make new lock object and initialize lock object with given parameter
  - return new lock object pointer

- int connect_with_lock_head(int64_t table_id, pagenum_t page_id, lock_t* lock_obj)
  - connect given lock object with lock header
  - if there is no corresponding lock header, make new one
  - return 0 if success, or non-zero if not

- lock_head_t* find_lock_head_in_table(int64_t table_id, pagenum_t pagenum)
  - find lock header object in lock table
  - where it's pagenum and table id is same with given parameter
  - return object's pointer or null if not found
    
- lock_head_t* insert_new_lock_head_in_table(int64_t table_id, pagenum_t pagenum)
  - insert lock header object into lock table at the position where it's pagenum and table id is same with given parameter
  - return new lock header object's pointer inserted

- void append_lock_in_lock_list(lock_t* lock_obj)
  - append lock in the lock list in object's sentinel
  - and also append in the trx lock list

- void remove_lock_from_lock_list(lock_t* lock_obj)
  - remove lock in the lock list in object's sentinel

- lock_t* find_compatible_lock_in_lock_list(lock_head_t* lock_head, lock_t* lock_obj)
  - find lock object that compatible with given lock object in page lock list defined by lock header
  - return object's pointer or null if not found 

- int detect_deadlock(lock_t* lock_obj, int source_trx_id, bool is_first = true)
  - detect deadlock will occur when given lock inserted into lock table
  - determine deadlock when parameter lock object is owned by source transaction (check cycle)
  - If there is no deadlock, return the number of conflicting lock
  - If there is deadlock, return -1
  - source_trx_id should be first lock object's owner trx id 
  - DO NOT SET is_first flag false at the first time (always return -1)

- void wake_conflicting_lock_in_lock_list(lock_t* lock_obj)
  - find and wake up the successor if it is waiting for the current transaction releasing the lock and no conflicting transaction left after releasing this lock

- int try_to_acquire_lock_object(int64_t table_id, pagenum_t page_id, int64_t key, int trx_id, int lock_mode)
  - Allocate and append a new lock object to the lock list of the record having the page id and the key.
  - If there is a predecessor's conflicting lock object in the lock list, sleep until the predecessor releases its lock.
  - If there is no predecessor's conflicting lock object, return 0
  - If an error occurs, return -1

- int release_acquired_lock(lock_t* lock_obj)
  - Remove the lock object from the lock list.
  - If there is a successor's lock waiting for the transaction releasing the lock, wake up the successor.
  - If success, return 0. Otherwise, return a non zero value.