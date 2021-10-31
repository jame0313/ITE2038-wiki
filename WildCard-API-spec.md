# WILDCARD API INSTRUCTION

For special purpose such as managing metadata like header page that can be used in multiple layer, it should need some method or protocol to access and synchronize it. In wildcard API, such wrapper function calls multiple layer api to meet special purpose. To avoid violating layered architecture, this function should use very carefully.

-----
## API Functions
1. void get_header_page_from_multiple_layer(int64_t table_id, page_t* dest)

- Get header page from multiple layer

try to get header page from upper layer to lower layer and copy header page to dest page.

- parameters
  - table_id - table id of the opened database
  - dest - dest page to store header page info

- return value - (none)
- exceptions
  - if error occurred in API, throw error msg
---
2. void set_header_page_from_multiple_layer(int64_t table_id, const page_t* src)

- Set header page from multiple layer

try to set header page from upper layer to lower layer and copy src page to header page in each layer for sync.

- parameters
  - table_id - table id of the opened database
  - src - source page that stored header page info to be copied
- return value - (none)
- exceptions
  - if error occurred in API, throw error msg
