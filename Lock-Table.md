#  LOCK TABLE
Lock table module manages lock object of multiple threads.
This architecture is simplified version of record lock for future concurrency control.

By using Latch(mutex) and hash table, only one thread should be able to access the lock table at a time.
By using per-record lock list, lock table allow only one thread to access the record item at a time ordered by request time.

In application(test code) side, each thread should try to access shared resource by using Lock Table API.
It requests lock before access and releases lock after using corresponding resource.

Use table id and record id to access lock table.

In lock table, it uses pthread API for concurrency control.

------

##  LOCK TABLE API

1. int init_lock_table()

- initialize lock table and mutex

Initialize mutex attribute, mutex variable, and hash_table.
The mutex's attribute is set to PTHREAD_MUTEX_DEFAULT defined in pthread.

- parameters - (none)
- return value - status code(0 is ok)
- exceptions - (none)
---
2. lock_t* lock_acquire(int table_id, int64_t key)

- Allocate and append a new lock object to the lock list of the record having key

Depending on wether there is a predecessor or not in corresponding record,
this returns the address of the new lock object immediately or sleep until predecessor releases it.

Get head lock object(head of lock list) from hash table using given table_id and key and make new lock object.

If there is no head object in hash table, make new head object and connect with new lock object and append it in hash table.

If there is no predecessor in the list, just connect new object with head object and return it's address immediately.
If not, the new object append in the end of the list and wait for the turn.

- parameters
  - table_id - table id of the opened database
  - key - record id of the opened database

- return value - lock object's address or NULL if error occured
- exceptions
  - if error occurred in pthread API, throw error message
---
3. int lock_release(lock_t* lock_obj)

- Remove the lock_obj from the lock list

Free lock_obj and remove it from the list.

If there is successor in the list, connect it with prev object(head object) and wake it up by using flag and conditional variable.

- parameters
  - lock_obj - lock object to be released

- return value - status code(0 is ok)
- exceptions
  - if error occurred in pthread API, throw error message
---

## LOCK TABLE MODULE

inner structure and function used in Lock table

- struct lock_t
  - lock object structure
  - lock_t* prev
    - previous pointer in lock list or nullptr if not existed
  - lock_t* nxt
    - next pointer in lock list or nullptr if not existed
  - lock_t* sentinel
    - sentinel(head) pointer
  - pthread_cond_t cond
    - conditional variable used in waiting in lock table
  - bool flag
    - flag for conditional variable

- struct hash_pair
  - inner structure for hashing std::pair object
  - hash algorithm used in boost lib using std::hash
  - source from https://www.boost.org/doc/libs/1_64_0/boost/functional/hash/hash.hpp
  - size_t operator()(const std::pair<T1, T2>& p) const
    - hash pair object by rearrange two hash value from each object's hash value.

- page_id
  - std::pair<int64_t, int64_t>
  - combine two property with pair
 
- std::unordered_map<page_id, lock_t*, hash_pair> hash_table
  - hash table that mapping page id to corresponding head lock object in lock table
  - search key is page_id({table_id, key})
  - corresponding value is lock obejct's address
  - use hash_pair to hash pair object

- pthread_mutex_t mutex
  - shared lock table latch (mutex) object for critical section in API

- pthread_mutexattr_t mattr
  - mutex attribute for initializing mutex

- lock_t* find_lock_in_hash_table(int table_id, int64_t key)
  - find lock object in hash table where it's key and table id is same with given parameter
  - return corresponding lock object address or null if not found