#  DISK SPACE MANANGER
The disk space manager manages space on disk by
mapping pages to locations on disk,
loading and saving pages between disk and memory.

The database file consists of a set of 4KiB pages.
Each page is roughly divided into three types: Header Page, Free Page, Allocated Page

------

##  DISK SPACE MANANGER API
1. int64_t file_open_table_file (const char * pathname)

- Open the database file.
It opens an existing database file using ‘pathname’ or create a new file if absent.
If a new file needs to be created, the default file size should be 10 MiB
Then it returns the file descriptor of the opened database file.
All other 5 commands below should be handled after open data file.

use DB_FILE_LIST for check duplicated open
and use realpath to distinguish isomorphic file path

when creating DB file, init header page and free page list by linking free page sequentially.

- parameters
  - pathname - pathname to database file

- return value - table id(just same as file descriptor for now)
- exceptions
  - if duplicated open occurred in same file, throw "this file has been already opened" msg
  - if open failed, throw "file_open_database_file failed" msg
  - if read/write system call failed, throw "(function_name) system call failed!"
---
2. (deprecated) pagenum_t file_alloc_page (int64_t table_id)

- Allocate a page.
It returns a new page number from the free page list.
If the free page list is empty, then it should grow the database file twice and return a free page number.

pop free page from list and maintain list linked

- parameters
  - table_id - table id of the opened database file

- return value - new page number
- exceptions
  - if given table_id not opened by file_open_database_file, throw "unvalid tabld id" msg
  - if read/write system call failed, throw "(function_name) system call failed!"
---
3. (deprecated) void file_free_page(int64_t table_id, pagenum_t pagenum)

- Free a page
It informs the disk space manager of returning the page with 'page_number' for freeing it to the free page

put the given page in the beginning of the list (LIFO)

- parameters
  - table_id - table id of the opened database file
  - pagenum - page number for free

- return value - (none)
- exceptions
  - if given table_id not opened by file_open_database_file, throw "unvalid tabld id" msg
  - if given pagenum is unvalid, throw "pagenum is out of bound in file_free_page" msg
  - if given pagenum is 0(header page), throw "free header page" msg
  - if read/write system call failed, throw "(function_name) system call failed!"
---
4. void file_read_page(int64_t table_id, pagenum_t pagenum, page_t* dest)

- Read a page.
It fetches the disk page corresponding to 'page_number' to the in memory buffer (i.e. dest)

- parameters
  - table_id - table id of the opened database file
  - pagenum - page number for read
  - dest - Pointer to the destination page where the content is to be copied

- return value - (none)
- exceptions
  - if given table_id not opened by file_open_database_file, throw "unvalid tabld id" msg
  - (deprecated) if given pagenum is unvalid, throw "pagenum is out of bound in file_read_page" msg
  - (new) if given pagenum is unvalid, just end function(upper layer handle this)
  - if read/write system call failed, throw "(function_name) system call failed!"
---
5. void file_write_page(int64_t table_id, pagenum_t pagenum, const page_t* src)

- Write a page.
It writes the in memory page content in the buffer (i.e. 'src') to the disk page pointed by page_number

Disk synced right after write operation by calling fsync

- parameters
  - table_id - table id of the opened database file
  - pagenum - page number for write
  - src - Pointer to the source page of data to be copied

- return value - (none)
- exceptions
  - if given table_id not opened by file_open_database_file, throw "unvalid tabld id" msg
  - (deprecated) if given pagenum is unvalid, throw "pagenum is out of bound in file_write_page" msg
  - (new) if given pagenum is unvalid, just write it (upper layer handle this)
  - if read/write system call failed, throw "(function_name) system call failed!"
---
6. void file_close_table_file()

- Close the database file.

Close all file and free all path string in in DSM::DB_FILE_LIST
and clear DSM::DB_FILE_LIST

- parameters - (none)
- return value - (none)
- exceptions
  - if read/write system call failed, throw "(function_name) system call failed!"
  - if close system call failed, throw "close db file failed"
  - it may throw other exceptions in free operation or SET and MAP container method.
---
## PAGE FORMAT IN DSM
1. Header Page

Header page is the first page (offset 0-4095) of a database file and contains important metadata to manage a list of free pages and file space.
- Free page number: byte range [0-7]
  - It points to the first free page (head of free page list)
  - 0, if there is no free page left.
- Number of pages: byte range [8-15]
  - It tells how many pages are paginated in this database file. (Count the header page itself as well.)
- reserved: byte range [16-4095]

2. Free Page

Free pages are all linked, and page allocation is done by getting a free page from the free page list.
- Next Free Page Number : byte range [0-7]
  - points the next free page.
  - 0, if end of the free page list.

3. Allocated Page

Allocated page are allocated by caller and they're maintained by upper layer.

---
## DSM module(DSM namespace)
use for Disk Space Manager Layer ONLY!!(DON'T USE IN OTHER LAYERS)

inner structure and function used in Disk Space Manager

- struct table_info
  - store table(file) information
  - int64_t table_id
  - char *path
    - path to database file
  - int fd
    - file descriptor to the database file

- table_info DB_FILE_LIST[MAX_DB_FILE_NUMBER];
  - maintain realpath and file descriptor list opened by open api call for each table id
  - consisted of (table_id, realpath, fd)

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

- union _dsm_page_t
  - union all type of page to reinterpret shared bitfield by using each type's member variable
  - page_t _raw_page;
  - DSM::header_page_t _header_page;
  - DSM::free_page_t _free_page;

- bool is_file_opened(int fd)
  - check given file descriptor is valid
  - validate fd opened and not closed by DSM before by checking fd in the DB_FILE_LIST
 
- bool is_path_opened(const char* path)
  - check given path is opened
  - check this path opened and not closed by DSM before by checking path in the DB_FILE_LIST

- bool is_pagenum_valid(int fd, pagenum_t pagenum)
  - check given pagenum is valid in fd (boundary check)

- void init_header_page(page_t* pg, pagenum_t nxt_page_number,  uint64_t number_of_pages);
  - init given page to header page format with given parameters

- void init_free_page(page_t* pg, pagenum_t nxt_page_number)
  - init given page to free page format with given parameters

- void store_page_to_file(int fd, pagenum_t pagenum, const page_t* src)
  - inner function to store page to file
  - write page and sync

- void load_page_from_file(int fd, pagenum_t pagenum, page_t* dest)
  - inner function to load page from file
  - read page

- int get_file_descriptor(int64_t table_id)
  - get file descriptor corresponding to given table id from DB_FILE_LIST
  - if given table id not existed return -1








