#  BUFFER MANAGER
The buffer manager manages buffer organization for caching on-disk pages in memory.

By using buffer manager, upper layer's operation result loaded from buffer or stored in buffer.

By using disk space manager, upper layer's request but not in buffer pool, loaded from disk and stored in buffer block.
During page eviction by LRU policy, the victim page stored in disk by DSM.

In buffer manager, two arrays are used to manage buffer block. One is frame_list(real image to on-disk page), the other is ctrl_blk_list(frame block pointer and some information). This makes each frame and each control block saved compacted.

To find page in buffer, buffer manager use hash table that maps page_id(table id and page number) to ctrl block. As hash table expected to find such block in constant time if existed, we can access buffer fastly.

To implement transaction concept, buffer manager uses buffer manager latch and page latch instead of pin variable.

For faster and parallel access, other layer can access buffer cache directly by calling direct api.

The eviction process caused by full buffer follows LRU policy.

------

##  BUFFER MANAGER API

1. int init_buffer(int num_buf = DEFAULT_BUFFER_SIZE)

- initialize buffer manager.

Allocate the buffer pool and init LRU linked list with the given number of entries.

Also initialize buffer manager latch and page latch.

In ctrl_blk_list, each element points corresponding frame and connects neighbor control block with prev and next block number or set -1 if not existed.

This makes buffer array filled with dummy page. (To use eviction process only in allocating block, we initialize LRU list full with dummy page. As we only flush dirty page, this dummy page doesn't flush to disk at all and doesn't occupy hash table.)

Also, initialize hash table for finding page with table id and page id.

If success, return 0. Otherwise, return non zero value.

**num_buf** should be enough to use upper layer freely. num_buf should be greater and equal to maximum number of pinned page This means upper layer can use up to num_buf pages simultaneously. In our File And Index Manager(project 3 ver.) use up to **3** pages simultaneously(i.e. FIM::insert_into_new_root_page).

- parameters
  - num_buf - number of buffer pool entries

- return value - status code(0 is ok)

- exceptions - (none)
---
2. pagenum_t buffer_alloc_page(int64_t table_id)

- Allocate a new page and load it to buffer

Allocate a new page by calling DSM api and load new page to buffer.
Set this page pin on because we expected that this API(buffer_alloc_page) was called to write something to new page soon in upper layer.

- parameters
  - table_id - table id of the opened database

- return value - new page number
- exceptions
  - if error occurred in buffer manager module or Sub Layer API, throw error message
---
3. void buffer_free_page(int64_t table_id, pagenum_t pagenum)

- Free a page and discard it from buffer

Free a given page by calling DSM api and discard given page from buffer.
Since un-dirty and not-accessed page anymore by upper layer is considered as dummy page and not flushed at all, set this page's dirty bit off not to flush to disk.

- parameters
  - table_id - table id of the opened database
  - pagenum - page number to be freed

- return value - (none)

- exceptions
  - if error occurred in buffer manager module or Sub Layer API, throw error message
---
4. void buffer_read_page(int64_t table_id, pagenum_t pagenum, page_t* dest, int lock_policy)

- read a page from buffer

This API requires buffer manager Latch.

Find the buffer block containing given table id and page number.
If found matching block, copy page content to dest.
Upper layer should provide dest to access content via that pages.

By using lock policy flag, it locks page in three ways.

- In BUFFER_WRITE_LOCK_MODE, use write lock(exclusive lock) in the writing situation.
- In BUFFER_READ_LOCK_MODE, use read lock(shared lock) in the reading situation.
- In BUFFER_NO_LOCK_MODE, only check there is no exclusive lock and use no lock in the temporary reading situation.

When there is conflicting lock in page latch, wait until it can acquire corresponding lock.

Only to be written page use BUFFER_WRITE_LOCK_MODE to reduce disk io. (to be written page means there is access soon and we can prevent this page to be evicted by pinning)

As we copy buffer contents to upper layer, we can evict such page freely when this is BUFFER_NO_LOCK_MODE (no write api call soon).
Skipping pinning read-only page is equivalent that this page will be unpinned right after return.

Upper layer sets lock policy flag to distinguish this property.

- parameters
  - table_id - table id of the opened database
  - pagenum - page number to read
  - dest - dest to write
  - lock_policy - lock policy flag (type of lock)

- return value - (none)

- exceptions
  - if there is error in pthread, throw "pthread error occurred".
  - if error occurred in buffer manager module or Sub Layer API, throw error message
---
5. page_t* buffer_direct_read_page(int64_t table_id, pagenum_t pagenum)

- read a page from buffer from buffer directly

This API requires buffer manager Latch.

Find the buffer block containing given table id and page number.
If found matching block, just return cache pointer directly.

Before accessing current block, it tries to acquire shared page lock.

When there is conflicting lock in page latch, wait until it can acquire corresponding lock.

If another thread accesses cache directly concurrently, there can be inconsistent read. So upper layer should use this API carefully.

- parameters
  - table_id - table id of the opened database
  - pagenum - page number to read

- return value - (none)

- exceptions
  - if there is error in pthread, throw "pthread error occurred".
  - if error occurred in buffer manager module or Sub Layer API, throw error message
---
6. void buffer_write_page(int64_t table_id, pagenum_t pagenum, const page_t* src)

- write a page to buffer and release page latch

This API requires buffer manager Latch.

Find the buffer block labeled with given table id and page number.
If matching block existed, copy src page's content to the corresponding block frame.
Upper layer should provide src page.
Only the locked block allowed to be written.
After writing page, set dirty bit on and unlock it.

In the case that src is NULL, this means there is no changes in page. So it only releases page latch(no dirty set and no copy).

To wake up successor, broadcast to other threads by using page conditional variable.

- parameters
  - table_id - table id of the opened database
  - pagenum - page number to read
  - src - source page to write to buffer or NULL if nothing changed

- return value - (none)

- exceptions
  - if there is error in pthread, throw "pthread error occurred".
  - if corresponding page is unlocked, throw "invalid write api call".
  - if error occurred in buffer manager module or Sub Layer API, throw error message
---
7. void buffer_direct_write_page(int64_t table_id, pagenum_t pagenum, bool is_dirty)

- read a page from buffer from buffer directly

This API requires buffer manager Latch.

Find the buffer block containing given table id and page number.
If found matching block, just set dirty bit and release page latch.

Only the locked block allowed to be written.

To wake up successor, broadcast to other threads by using page conditional variable.

- parameters
  - table_id - table id of the opened database
  - pagenum - page number to read

- return value - (none)

- exceptions
  - if there is error in pthread, throw "pthread error occurred".
  - if corresponding page is unlocked, throw "invalid write api call".
  - if error occurred in buffer manager module or Sub Layer API, throw error message
---
8. void buffer_close_table_file()

- Flush all and destroy

This API requires buffer manager Latch.

Scan all block in buffer list.
Flush only dirty page to disk file.
Free and clear buffer pool and hash table.

- parameters - (none)
- return value - (none)
- exceptions
  - if error occurred in buffer manager module or Sub Layer API, throw error message
---
## BUFFER FORMAT IN BM
1. FRAME
Frame is the image of on-disk page in memory and contains contents of target page.

2. CONTROL BLOCK
Control Block contains frame and some info used in buffer manager and LRU policy.
- frame_ptr : byte range [0-7]
  - points to the corresponding real frame.
- table_id : byte range [8-15]
- pagenum : byte range [16-23]
- lru_prv_blk_number : byte range [24-31]
  - points to the previous block number in LRU linked list.
  - -1, if not existed (front of list).
- lru_nxt_blk_number : byte range [32-39]
  - points to the next block number in LRU linked list.
  - -1, if not existed (end of list).
- is_dirty : byte range [40-40]
  - set on if it needs flush 
  - indicate whether contents change or not
- pthread_rwlock_t page_latch = PTHREAD_RWLOCK_INITIALIZER
  - identify this buffer is-use
  - set eviction free that prevents eviction by LRU policy
  - limit access of thread by implementing transaction policy
- pthread_cond_t cond = PTHREAD_COND_INITIALIZER
  - conditional var for waiting for releasing

3. PAGE ID
Page Identification Code to use for searching the block from hash table. This consists of table id and page number.
  - std::pair<int64_t, pagenum_t>
    - combine two property with pair

---
## BM module(BM namespace)
use for Buffer Manager Layer ONLY!!(DON'T USE IN OTHER LAYERS)

inner structure and function used in Buffer Manager

- struct ctrl_blk
  - buffer control block structure
  - specific implementation of CONTROL BLOCK
  - frame_t* frame_ptr
  - int64_t table_id
  - pagenum_t pagenum
  - blknum_t lru_prv_blk_number
  - blknum_t lru_nxt_blk_number
  - bool is_dirty
  - pthread_rwlock_t page_latch = PTHREAD_RWLOCK_INITIALIZER
  - pthread_cond_t cond = PTHREAD_COND_INITIALIZER

- struct hash_pair
  - inner structure for hashing std::pair object
  - hash algorithm used in boost lib using std::hash
  - source from https://www.boost.org/doc/libs/1_64_0/boost/functional/hash/hash.hpp
  - size_t operator()(const std::pair<T1, T2>& p) const
    - hash pair object by rearrange two hash value from each object's hash value.

- struct header_page_t
  - header page(first page) structure
  - pagenum_t free_page_number
    - point to the first free page(head of free page list) or indicate no free page if 0
  - uint64_t number_of_pages
    - the number of pages paginated in DB file
  - uint8_t \_\_reserved\_\_[PAGE_SIZE - 16]

- struct free_page_t
  - free page structure
  - pagenum_t nxt_free_page_number;
    - point to the next free page or indicate end of the free page list if 0
  - uint8_t \_\_reserved\_\_[PAGE_SIZE - 8];

- union _bm_page_t
  - union all type of page to reinterpret shared bitfield by using each type's member variable
  - page_t _raw_page;
  - BM::header_page_t _header_page;
  - BM::free_page_t _free_page;

- frame_t *frame_list
  - frame(page) array

- BM::ctrl_blk *ctrl_blk_list
  - ctrl block(frame_ptr + info) array

- __gnu_pbds::gp_hash_table<page_id, blknum_t, BM::hash_pair> hash_table
  - hash table that mapping page id to corresponding ctrl block in the list
  - search key is page_id({table_id, pagenum})
  - corresponding value is block number
  - use BM::hash_pair to hash pair object

- size_t BUFFER_SIZE
  - length of buffer array

- blknum_t ctrl_blk_list_front
  - LRU list front pointer
  - point LRU block number

- blknum_t ctrl_blk_list_back
  - LRU list back pointer
  - point MRU block number
- pthread_mutex_t buffer_manager_latch
  - buffer manager latch

- void flush_frame_to_file(blknum_t blknum) 
  - get corresponding block in the list
  - flush such page to disk by calling DSM write api

- blknum_t find_ctrl_blk_in_hash_table(int64_t table_id, pagenum_t pagenum)
  - find ctrl block number in hash table where it's pagenum and table id is same with given parameter
  - return ctrl block number or -1 if not found

- blknum_t find_victim_blk_from_buffer()
  - find victim block for eviction by following the LRU policy
  - return ctrl block number or -1 if not found(i.e. all pinned)

- void move_blk_to_end(blknum_t blknum)
  - move given block to end of LRU list by reconnecting some block's pointer caused by page access
  - maintain LRU policy list by moving recently accessed page to the end of list 
  - modify list front and list back block number if changed
  - there is case that no operation needed (such block is already in back of list)
   
- ctrl_blk* get_ctrl_blk_from_buffer(int64_t table_id, pagenum_t pagenum)
  - core function of buffer manager
  - get ctrl block from buffer
  - find block in buffer or get from disk and try to evict victim page
  - if victim page is dirty, flush it to disk
  - if eviction occurred, update hash table and initalize block info(pagenum, table_id, etc.)
  - move such ctrl block to the end of list 
  - return control block pointer or throw msg if it can't evict
