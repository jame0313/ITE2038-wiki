#  Changes in project 5 milestone 2

There are some changes in several managers to implement implicit locking and lock compression.

There can be new API or changes in the previous API or some changes in struct.

For instance, bitmap needed for lock compression and transaction id in slot needed for implicit lock.

------

##  CHANGES IN File and Index Manager
1. Add transaction id field in page slot in leaf page structure 

To implement implicit exclusive lock, record's slot needs space for writing transaction id that acquired exclusive lock without explicitly inserting a lock object.

Using trx_id field in slot, transaction can check existence of implicit lock or set implicit lock by using new API in Index manager.

---
2. int idx_get_trx_id_in_slot(int64_t table_id, pagenum_t page_id, uint32_t slot_number)

- Get transaction id in given slot for implicit locking

Read the target page by using buffer direct read & write API and return transaction id in the slot.

This information can be used in lock and transaction manager for checking transaction id in the slot.

---
3. void idx_set_trx_id_in_slot(int64_t table_id, pagenum_t page_id, uint32_t slot_number, int trx_id)

- Set transaction id in given slot for implicit locking

Write the given transaction id in target slot by using buffer direct read & write API

Compare old transaction id with given parameter to set dirty flag in the page.

This API can be used in lock and transaction manager for setting implicit lock or release implicit lock.

------

## CHANGES IN Transaction Manager
1. Add some field in transaction log structure

To use new index manager api, add some field(e.g. page_id, slot_number) to specify accurate location in page.

---
2. void TM::release_all_implicit_lock(int trx_id)

- Release all implicit lock in slot

In each log entry, release implicit lock by setting trx_id field in slot to 0 by calling index manager API.

***This is unnecessary inner function as lock manager can check transaction id by looking up transaction table (figure out given id is active)***

But as I don't implement avoiding wrap-around transaction id in project 5 (It will need in further project), this function used temporary for minor exception handling.

---
3. changes in append_lock_in_table

In converting implicit lock to explicit lock, it can be occurred that last lock in transaction lock list is sleeping state whereas lock to be converted is acquired state.

In current implementation of deadlock detection, Lock Manager traversal wait-for graph by using last lock only in transaction lock list for fast traversal.

To make deadlock detection's assumption (Only last lock must be sleeping where owner transaction is sleeping), append_lock_in_table inner function append given lock at the front instead of end of the list.

This changes can affect to trx_append_lock_in_trx_list API.

------

## CHANGES IN Lock Manager
1. Add bitmap in lock_t structure

To implement lock compression, lock object needs some space to express multiple same type of locks in single object.

Each bit represents corresponding slot number.

By using this bitmap, lock manager can check compatible lock, deadlock, and successor lock to wake up in record-wise.

---
2. Use bitmap in deadlock detection, waking conflict lock up, and finding compatible lock

In judging whether these two locks are conflicting or not, use bitwise and to represent intersection that indicates confliction in addition to comparing key value.

In this deadlock detection and waking successor algorithm, it only considers first conflicting X lock or a series of S locks. As there is compressed lock now, it checks record-wise in parallel by using bitmap instead of bool flag.


---
3. lock_t* find_same_trx_lock_in_lock_list(lock_head_t* lock_head, int trx_id)

- Find lock object already acquired by given transaction for lock compression

New inner function to search same transaction's S lock object in current page lock list.

This output object can be target lock object to be combined with acquiring lock object.

---
4. void convert_implicit_lock_to_explicit_lock(int64_t table_id, pagenum_t page_id, int64_t key, uint32_t slot_number, int trx_id)

- Convert implicit lock to explicit lock

New inner function to make new explicit lock object for implicit acquired lock and append into page lock list and transaction list.

When there is alive implicit lock and there is trying-to-acquire conflicting lock, this lock should convert implicit lock to explicit lock by calling this inner function.

---
5. void try_to_lock_compression(lock_t* lock_obj)

- Try to do lock compression

New inner function to do lock compression where there is lock belonged to same transaction.

By using find_same_trx_lock_in_lock_list inner function, it can find target lock to be combined.

In the found case, combine two lock by using bitmap and remove later lock(current lock).

---
6. changes in lock_acquire API

To implement implicit lock, it need to check whether there is implicit lock and that implicit lock is valid.

When there is no conflicting lock in page lock list, it check implicit lock (transaction id in slot) by using index manager API. If there is alive transaction id in slot, convert this lock into explicit by calling convert_implicit_lock_to_explicit_lock and call lock_acquire again to try to acquire again(make own lock object and sleep).

When there is no conflicting lock even in implicit lock, do lock compression by calling try_to_lock_compression or do implicit locking by using index manager API(set transaction id in slot) depending on lock mode.