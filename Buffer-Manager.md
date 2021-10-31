#  BUFFER MANAGER
The buffer manager manages buffer organization for caching on-disk pages in memory.

By using buffer manager, upper layer's operation result loaded from buffer or stored in buffer.

By using disk space manager, upper layer's request but not in buffer pool, loaded from disk and stored in buffer block.
During page eviction by LRU policy, the victim page stored in disk by DSM.

------

##  BUFFER MANAGER API

1. int init_buffer(int num_buf = DEFAULT_BUFFER_SIZE)

- initialize buffer manager.

Allocate the buffer pool and init LRU linked list with the given number of entries.
If success, return 0. Otherwise, return non zero value.

Point corresponding frame and connect neighbor control block with prev and next block number or set -1 if not existed.

Also, initialize hash table for finding page with table and page id.

- parameters
  - num_buf - number of buffer pool entries

- return value - status code(0 is ok)
- exceptions - (none)
---
2. pagenum_t buffer_alloc_page(int64_t table_id)

- Allocate a page

Get header page block
If success, return 0. Otherwise, return non zero value.

Only existed key value record can be deleted. Otherwise, API return non zero value.

Check violating the B+ tree properties and modify tree structure to ensure occupancy invariant if it needed.

- parameters
  - table_id - table id of the opened database

- return value - page number
- exceptions
  - if error occurred in Buffer manager module or Sub Layer API, print error message and return -1
---
3. void buffer_free_page(int64_t table_id, pagenum_t pagenum)

- Free a page

Find the record containing input key.
If found matching key, store matched value string in ret_val and matched size in val_size.
If success, return 0. Otherwise, return non zero value.
The caller should allocate memory for a record structure.

Only existed key value record can be found. Otherwise, API return non zero value.

- parameters
  - table_id - table id of the opened database
  - pagenum - page number to be freed

- return value - (none)
- exceptions
  - if error occurred in buffer manager module or Sub Layer API, throw error message
---
4. void buffer_read_page(int64_t table_id, pagenum_t pagenum, page_t* dest, bool readonly)

- read a page from buffer

Find the buffer block containing input table id and page number.
If found matching block, copy page content to dest.
Upper layer uses dest to access content.
Only to be written page get pinned to reduce disk io. (to be written page means there is access soon and we prevent this page to be evicted)
Upper layer sets readonly flag to distinguish this property.

- parameters
  - table_id - table id of the opened database
  - pagenum - page number to read
  - dest - dest to write
  - readonly - read only flag (whether to pin)
- return value - (none)
- exceptions
  - if corresponding page already has pinned, throw "double write access".
  - if error occurred in buffer manager module or Sub Layer API, throw error message
---
5. void buffer_write_page(int64_t table_id, pagenum_t pagenum, const page_t* src)

- write a page to buffer

Find the buffer block containing input table id and page number.
If found matching block, copy src page content to the block content.
Upper layer provides src page.
Only pinned block allowed to be written.
After writing page, set dirty property on and unpin it.

- parameters
  - table_id - table id of the opened database
  - pagenum - page number to read
  - src - source page to write to buffer
- return value - (none)
- exceptions
  - if corresponding page is unpinned, throw "invalid write access".
  - if error occurred in buffer manager module or Sub Layer API, throw error message
---
6. void buffer_close_table_file()

- Flush all and destroy

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
  - set on if it need flush 
  - identify content's changes
- (reserved) : byte range [41-43]
  - for aligned struct
- is_pinned : byte range [44-47]
  - identify this buffer is-use
  - set eviction free when such page to be written

3. PAGE ID
Page Identification Code to use for searching the block in the list. This consists of table id and page number.
  - std::pair<int64_t, pagenum_t>
    - combine two property with pair

---
## BM module(BM namespace)
use for Buffer Manager Layer ONLY!!(DON'T USE IN OTHER LAYERS)

inner structure and function used in Buffer Manager

- struct ctrl_blk
  - buffer control block structure
  - specific spec of CONTROL BLOCK
  - frame_t* frame_ptr
  - int64_t table_id
  - pagenum_t pagenum
  - blknum_t lru_prv_blk_number
  - blknum_t lru_nxt_blk_number
  - bool is_dirty
  - uint32_t is_pinned
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

- std::unordered_map<page_id, blknum_t, BM::hash_pair> hash_table
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
