#  FILE AND INDEX MANAGER
The file and index manager manages record and file organization by
using disk-based b+ tree (b+ tree index) to expertize access method.

By using disk space manager, file and index manager's operation result loaded from file or stored in file.

In database, record is consisted of key and value. The value is variable-length field. The record can be inserted, deleted, found by using table ID and key.

The database file consists of a set of 4KiB pages.
Each page is roughly divided into four types: Header Page, Free Page, Leaf page, Internal page

------

##  FILE AND INDEX MANAGER API
1. int64_t open_table(char *pathname)

- Open the database file.

Open existing data file using 'pathname' or create one if not existed.
If a new file needs to be created, the default file size should be 10 MiB
If success, return the unique table id, which represents the own table in this database.
Otherwise, return negative value.

Call file_open_table_file(pathname) to use Disk Space Manager API.

- parameters
  - pathname - pathname to database file

- return value - table id of the opened database file
- exceptions
  - if error occurred in Disk Space Manager API, print error message and return -1
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
  - if error occurred in File and Index manager module or Disk Space Manager API, print error message and return -1
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
  - if error occurred in File and Index manager module or Disk Space Manager API, print error message and return -1
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
  - if error occurred in File and Index manager module or Disk Space Manager API, print error message and return -1
---
5. int init_db()

- Initialize DBMS

Initialize database management system.
If success, return 0. Otherwise, return non zero value.

Before starting DBMS, each layer initialize their status if needed.
In Disk Space Manager and File and Index Manager, there is nothing to do for now. 

- parameters - (none)

- return value - status code(0 is ok)
- exceptions - (none)
---
6. int shutdown_db()

- Shutdown DBMS

Shutdown your database management system.
If success, return 0. Otherwise, return non zero value.

Before shutdown DBMS, each layer finish and save their status if needed.
In Disk Space Manager, close all file by calling file_close_table_file API.

- parameters - (none)
- return value - status code(0 is ok)
- exceptions
  - if error occurred in Disk Space Manager API, print error message and return -1
---
## PAGE FORMAT IN FIM
1. Header Page

Header page is the first page (offset 0-4095) of a database file and contains important metadata to manage b+ tree.
- Free page number: byte range [0-7]
  - It points to the first free page (head of free page list)
  - 0, if there is no free page left.
- Number of pages: byte range [8-15]
  - It tells how many pages are paginated in this database file. (Count the header page itself as well.)
- Number of pages: byte range [16-23]
  - It points the root page within the data file.
  - 0, if there is no root page.
- reserved: byte range [24-4095]

2. Free Page

Free pages are all linked, and page allocation is done by getting a free page from the free page list.
- Next Free Page Number : byte range [0-7]
  - points the next free page.
  - 0, if end of the free page list.

3. Page Header

Page Header is the first 128 bytes of a database page and contains important metadata to manage b+ tree both in internal page and leaf page.

- Parent page Number: byte range [0-7]
  - It points to the position of parent page
  - 0, if it is the root page.
- Is Leaf: byte range [8-11]
  - 0, if it is internal page
  - 1, if it is leaf page
- Number of keys: byte range [12-15]
  - the number of keys within this page
- reserved: byte range [26-128]

4. Leaf Page

Leaf page is constructed in slotted page layout. It contains fixed length key and variable length value(50bytes ~ 112bytes). First 128 bytes will be used as a page header.

Each slot consisted of key(8bytes), size(2bytes), offset(2bytes). The slot is sorted by key value. Keys are inserted from the beginning of a page body and corresponding values are inserted from the end. The corresponding value is in byte range: [offset, offset+size)

Right sibling page number and amount of free space field is stored at the end of page header. If this page is rightmost, the right sibling page number is 0.

- Page header: byte range [0-127]
- Amount of Free space: byte range [112-119]
  - amount of free space in the page
- Right Sibling Page Number: byte range [120-127]
  - points the right sibling page
  - 0, if this page is rightmost page
- Slot: byte range [128-]
  - store packed slot list in beginning of page body
- Value: byte range [-4095]
  - store packed value list in end of page body

5. Internal Page

Internal Page contains fixed length key and fidex length page number. First 128 bytes will be used as a page header.

Each pair consisted of key(8bytes), page number(8bytes). The default order is 124. So Branching factor is 249.

Leftmost page number is stored at the end of page header.

- Page header: byte range [0-127]
- Leftmost Page Number: byte range [120-127]
  - points the leftmost page number
  - 0, if this page is rightmost page
- Key and Page Number: byte range [128-4095]
  - store packed key and the corresponding page number

---
## FIM module(FIM namespace)
use for File and Index Manager Layer ONLY!!(DON'T USE IN OTHER LAYERS)

inner structure and function used in File and Index Manager

- struct header_page_t
  - header page(first page) structure
  - pagenum_t free_page_number
    - point to the first free page(head of free page list) or indicate no free page if 0
  - uint64_t number_of_pages
    - the number of pages paginated in DB file
  - pagenum_t root_page_number
    - pointing the root page within the data file or indicate no root page if 0
  - uint8_t \_\_reserved\_\_[PAGE_SIZE - 24]

- struct page_header_t
  - page header structure in leaf, internal page
  - pagenum_t parent_page_number
    - point to parent page or indicate root if 0
  - uint32_t is_leaf
    - 0 if internal page, 1 if leaf page
  - uint32_t number_of_keys
    - number of keys within page

- struct page_slot_t
  - leaf page's slot structure
  - int64_t key
  - uint16_t size
  - uint16_t offset
    - in-page offset, point begin of value
    - value byte range: [offset, offset+size)

- struct keypagenum_pair_t
  - (key, page number(pointer)) pair structure
  - int64_t key
  - pagenum_t page_number

- struct leaf_page_t
  - leaf page structure
  - FIM::page_header_t page_header
  - uint8_t \_\_reserved\_\_[96]
  - uint64_t amount_of_free_space
    - free space in slot and value space
  - pagenum_t right_sibling_page_number
    - point to right sibling page or indicate rightmost leaf page if 0
  - FIM::page_slot_t slot[64]
    - slot list
  - uint8_t value_space[3200]
    - value space to store record value

- struct internal_page_t
  - internal page structure
  - FIM::page_header_t page_header
  - uint8_t \_\_reserved\_\_[104]
  - pagenum_t leftmost_page_number
    - point to leftmost page
  - FIM::keypagenum_pair_t key_and_page[248]
    - key and page number

- union _fim_page_t
  - union all type of page to reinterpret shared bitfield by using each type's member variable
  - page_t _raw_page
  - FIM::header_page_t _header_page
  - FIM::internal_page_t _internal_pag
  - FIM::leaf_page_t _leaf_page

 - pagenum_t make_page(int64_t table_id) 
  - get page from DSM and return new page number
 - int change_root_page(int64_t table_id, pagenum_t root_page_number, bool del_tree_flag = false)
  - change root page number in header page
  - you can set root page number to 0 when del_tree_flag is on
  - return 0 if success or -1 if fail

 - pagenum_t find_leaf_page(int64_t table_id, int64_t key)
  - find the leaf page in which given key is likely to be
  - return 0 if there is no tree
  - throw msg in looping situation (doesn't has key but not leaf)
   
 - int find_record(int64_t table_id, int64_t key, char *ret_val = NULL, uint16_t* val_size = NULL)
  - find the record value with given key
  - save record valud in ret_val(caller must provide it) and set size in val_size
  - you can get existence state by using key only and setting ret_val and val_size null
  - return 0 if success or -1 if fail
   
 - int insert_record(int64_t table_id, int64_t key, char *value, uint16_t val_size)
  - insert master function
  - insert record in tree
  - return 0 if success or -1 if failed
  - if given key is already in tree, return -1
   
 - pagenum_t init_new_tree(int64_t table_id, int64_t key, char *value, uint16_t val_size)
  - make root page and put first record
  - set initial state in root
  - return new root page number
    
 - void insert_into_leaf_page(pagenum_t leaf_page_number, int64_t table_id, int64_t key, char *value, uint16_t val_size)
  - insert record in given leaf page
  - push key in sorted order and push value in right next free space (packed)
   
 - int insert_into_leaf_page_after_splitting(pagenum_t leaf_page_number, int64_t table_id, int64_t key, char *value, uint16_t val_size)
  - make new pages and split records in leaf page and new record into two pages evenly
  - Set the first record that is equal to or greater than 50% of the total size
  - on the Page Body as the point to split, and then move that record, and all records
  - after that to the new leaf page.
  - set new leaf page at right of old leaf page
  - need push new key(new leaf's first key) and new leaf page in parent's page
  - return insert_into_parent_page call(next phase)
   
 - int insert_into_parent_page(pagenum_t left_page_number, int64_t table_id, int64_t key, pagenum_t right_page_number)
  - insert key and right page number in parent page
  - return 0 if success
   
 - pagenum_t insert_into_new_root_page(pagenum_t left_page_number, int64_t table_id, int64_t key, pagenum_t right_page_number)
  - make new root page and set initial state
  - put left page #, key, right page # in new root page
  - connect two given pages to new root page as parent
  - return new root page number
   
 - void insert_into_page(pagenum_t page_number, pagenum_t left_page_number, int64_t table_id, int64_t key, pagenum_t right_page_number)
  - insert key and right page number in given page
  - push key and page # in sorted order
   
 - int insert_into_page_after_splitting(pagenum_t page_number, pagenum_t left_page_number, int64_t table_id, int64_t key, pagenum_t right_page_number)
  - make new pages
  - split key and page number in page and new data (key and right page #) into two pages evenly
  - old page has DEFAULT_ORDER keys and new page has DEFAULT_ORDER keys as a result
  - set new page at right of old page
  - need push middle key and new page in parent's page
  - return insert_into_parent_page call 
   
 - int delete_record(int64_t table_id, int64_t key)
  - delete master function
  - delete key and corresponding value in tree
  - return 0 if success or -1 if fail
  - if there is no such key, return -1
   
 - int delete_entry(pagenum_t page_number, int64_t table_id, int64_t key)
  - delete key(and corresponding data(page number or value)) and in page
  - and make tree obey key occupancy invariant
  - return 0 if success or -1 if failed
   
 - void remove_entry_from_page(pagenum_t page_number, uint64_t table_id, int64_t key)
  - remove key and data in page
  - data is value(leaf page) or page number(internal page)
  - fill gap caused by deleting key
   
 - pagenum_t adjust_root_page(pagenum_t root_page_number, int64_t table_id)
  - deal with root page changes
  - use child as root or delete tree when root page is empty
  - return existed root page number or 0 if entire tree is deleted
  - return new root page number if root page is changed
   
 - int merge_pages(pagenum_t page_number, int64_t table_id, int64_t middle_key, pagenum_t neighbor_page_number, bool is_leftmost) 
  - move all page's contents to neighbor page
  - to merge two pages into one page
  - need delete right page number in parent page
  - return delete_entry call
  
 - void redistribute_pages(pagenum_t page_number, int64_t table_id, int64_t middle_key, pagenum_t neighbor_page_number, bool is_leftmost)
  - move some page's contents to neighbor page
  - pull a record from neighbor page until its free space becomes
  - smaller than the threshold 