# Analysis of in-memory B+ tree
## 1.  node data structure and modification
---
```
-leaf node-
pointers [2*D + 1] (point to records)
keys [2*D]
parent
is_leaf
num_keys
next (not used in find, insert, delete)
```
```
-internal node-
pointers [2*D + 1] (point to node)
keys [2*D]
parent
is_leaf
num_keys
next (not used in find, insert, delete)
```
Note. in bpt code, order means different from lecture's meaning.
Let D means order of the tree in lecture.
Then order variable in bpt code equals 2*D + 1
> order = 2*D + 1

So, order equals max fan-out in lecture.
 
**From now on, order of the tree means D (in lecture meaning) not order variable in code.**

Then, Occupancy Invariant in this tree is as follows
> D <= (the number of entries) <= 2*D

In leaf node, it stores record pointers array and keys array.
It stores up to 2*D keys and record pointers and right leaf pointer.
Right leaf node's pointer is stored in last pointer in array.
If the node is rightmost node in the tree, then that pointer value is NULL(0).

In internal node, it stores node pointers array and keys array.
It stores up to 2\*D keys and 2\*D + 1 record pointers.
Records whose key value is lower than key[0] are stored in pointer[0]'s node.
For all i(0<=i<(number of the key)), Records whose key value is greater than and equal to key[i] and is lower than key[i+1] are stored in pointer[i+1]'s node.
Records whose key value is greater than and equal to last key are stored in last pointer's node.
It is same as Key Invariant. Also, the number of pointers in internal node is greater than the number of keys by 1.
> Let, [P_0, K_0, P_1, K_1, ..., K_n, P_n+1]
>
> records(K_i <= key value < K_(i+1)) is stored in P_(i+1)
>
> (number of pointers) = (number of keys) + 1

parent, is_leaf, num_keys means same in both type of nodes. next doesn't use in our project's api, so we can ignore this field.

By paginated architecture in disk-based B+ tree, all member variables in node structure stored in certain field in page.
We can store parent, is_leaf, num_keys in same field in page header in both type of page.

Because leaf page is slotted page, we store slot(key + size + in-page offset) instead of keys arrays at front part of page body contiguously. Records are stored at end of page reversely instead of using pointers array. Right leaf page pointer(Special pointer) stored in page header. As we use slotted page architecture, we have to store additional information in page header such as amount of free space.

In internal page, we store key and page number(instead of pointer) in page body alternately instead of using two arrays. In page body, we store these values structured like [K_0, P_1, K_1, ..., K_n, P_n+1]. We can store P_0(Special pointer) in page header.

In such way, we can store dynamic node structure in static page space.
---






