# Disk Space Manager Test

Test the correctness of API by using Google unit test framework

---
## FileInitialization
When a file is newly created, a file of 10MiB is created and The number of pages corresponding to 10MiB should be created. Check the "Number of pages" entry in the header page.
- check file size is 10MiB and header page's the number of pages entry
- test for file_open_database_file and file_close_database_file
---
## PageManagement
Allocate two pages by calling file_alloc_page() twice, and free one of them by calling file_free_page(). After that, iterate the free page list by yourself and check if only the freed page is inside the free list.
- check free page list works properly while performing alloc and free
- test for file_alloc_page and file_free_page
---
## PageIO
After allocating a new page, write any 4096 bytes of value (e.g., " aaaa …") to the page by calling the file_write_page () function. After reading the page again by calling the file_read_page () function, check whether the read page and the written content are the same.
- compare buffer and in-disk content by memcmp
- compare buffer and reopened in-disk content
- test for file_write_page and file_read_page
---
## ErrorHandling
make various error situation and check the error msg is thrown
- test for duplicated open, unvalid fd, and out of bound page number
---
## FileGrowTest
When new pages are allocated by calling file_alloc_page (), new pages must be allocated as much as the current DB size. check doubling the current file space when there is no free page and caller requests it.
- allocate all free page in the list and check the file size doesn't grow yet and the number of page doesn't increase yet
- allocate one more to make file grow and check file size growth and doubling the number of page
- free all allocated page and check the number of free page also almost doubled
- free file list order after free all pages is reverse of original order before allocation