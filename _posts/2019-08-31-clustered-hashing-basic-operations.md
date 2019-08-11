---
layout: post
title:  "Clustered Hashing: Basic Operations"
date:   2019-08-15 17:02:14 -0700
categories: algo
---

## Introduction

I've recently been tasked to optimize the memory usage of a real time application (zeek.org). This application captures network packet and drives a lot of scripts which uses dictionaries. Under certain stress tests it has created a million dictioniaries in a matter of minutes. All of sudden how efficient the dictionary uses memory matters a lot to the overall application memory consumption.

Unsurprisingly the dictionary is implemented as chain-linked hash table. Why chain hash table is so popular? Let's step back a little to start from the motivation to have a hash table and see what leads us to chain hash table.

## Speed up Lookup

### Unsorted List
For a list of N items, the worst time used to look up a specific item is O(N). You basically look at them one by one. You cannot do worse than that. The items don't have to be in order, so insert happens at the end as append so insert takes O(1). Remove takes the same as lookup O(N), because you need to find it first in O(N). To remove it, you move the last item to occupy the spot and shrink list by one. The items don't have to use more space than necessary to store them.

`Lookup: O(N)`  
`Insert: O(1)`  
`Remove: O(N)`  
`Space : O(N)`  

Now we want to do better in lookup. Considering in computer science (and in every aspects of real life), there's no free lunch. We have to see what can we sacrifice to speed up the lookup.

### Sorted List

If we sacrifice time used in insert a bit, we can keep the list in order. You can use binary search for lookup in O(lgN) time. For Insert, we first need to find the right position, same as lookup in O(lgN), then we need to shift all items by one to the right to make room for the new item, and that's O(N). Remove is similar to Insert, you need to find the item first in O(lgN) time, and then you need to shift left all items on the right of the position by one.

`Lookup: O(lgN)`  
`Insert: O(lgN) + O(N) = O(N)`  
`Remove: O(lgN) + O(N) = O(N)`  
`Space : O(N)`  

Wait, you'd think, insert and remove can be as fast as O(lgN) if we use single link list. But can you do binary search in an sorted link list? You simply have to look one by one. This solution is worst the basic unsorted one in every aspect. And don't forget, by using link list, you use more space, as pointers occupy space, not small, 8 bytes each. We will revisit this later.

So keep list in order improves lookup greatly, but with heavy sacrifice on insert and remove. Is this a bad idea at all? Not necessarily. If you have a dictionary you insert once, never remove it, then most of the time is lookup. It is still a sensible solution.

### Binary Search Tree (BST)

The next idea is to sacrice space. If adding one pointer on the item  doesn't work, How about adding two? If each item contains two pointers, we can maintain it as a binary search tree. The idea of BST is to keep items sorted, but in a special binary tree form. Lookup can perform binary search just like sorted list, but insert and remove are almost as fast as lookup. After finding the item or position, you don't need to shift items back and forth. You simply change pointers to insert and remove, which takes constant time. Assume the binary tree is balanced because a variation of the BST, red black tree, can keep the tree balanced with a small price (`lgN` swaps).

`Lookup: O(lgN)`  
`Insert: O(lgN)`  
`Remove: O(lgN)`  
`Space : O(N)`  

This is good, isn't it? Not so fast.  As each item now requires two pointers, and each pointer is 8 bytes in 64bit CPU system, each item consumes 16 extra bytes. Suppose average item size is S, we have D=(8+8+S)/S. The space is actually O(DN). Theorectically O(DN) = O(N), but in reality you still uses D times memory than necessary. For an integer of 4 bytes, D=5.

Doesn't indirection of pointers consume computer time? For each item, you have to follow its pointers to another item. Suppose read item takes x amount of time, and load extra pointer take y amount of time, lookup actually takes O(lg((x+y)/xN) = O(lg(CN)) = O(lgC+lgN), given C=(x+y)/x. When x is small, such as read an 4-byte integer, load 8-byte takes at least same amount of time. So you could easily double the lookup time. It can be even worse... When items approach certain large counts, random pointers cause cache misses: the item pointed to is not in the current cache and CPU needs to reload it from the RAM. This is very costly. We will revisit it later. So the acutally binary search tree performance is:

`Lookup: O(lgCN) = O(lgC + lgN)`  
`Insert: O(lgCN) = O(lgC + lgN)`  
`Remove: O(lgCN) = O(lgC + lgN)`  
`Space : O(DN)`  

Sacrifice in insert and remove to the extreme make lookup as fast as O(lgN). To go further, we only have space to sacrifice. If we leave spaces among items on purpose, can we do better on lookup? Yes we can! Here comes in picture: hash table.

### Hash Table

For a item with key, we calculate an integer called hash through a hash function H: `hash = H(key)`. We allocate a table with table_size a few times larger than item count. We then map hash to the index of the table called bucket through another map function M: `bucket = M(hash, table_size)`. We then directly store item on position bucket, as shown below:

![hash-table-basic](/img/hash-table-basic.dot.png)

To lookup item, we do the same: `hash=H(key), bucket=M(hash, table_size)` table[bucket] is the item. Insert and remove are the same. find the bucket, then put to position or remove from the position (special mark). so,

`Lookup: O(1)`  
`Insert: O(1)`  
`Remove: O(1)`  
`Space : O(DN)`  

1/D is called load factor, which is usually 30-50%. So D is 2x - 3x N. For a small item such as integer, the space is even smaller than BST!

Sacrificing the space a few times makes lookup, insert and remove to be as fast as possible and ultimate goal is achieved?

Not so fast.

What if two items with K1 and K2, after function H, reaches the same hash?
`H(K1) == H(K2)`? With the same hash, both items find the same bucket. Follow the previous logic item 2 will overwrite item1 on insert. Even if two hashes are different, `H(K1) != H(K2)`, mapping hash to bucket can still result in same bucket: `M(H(K1)) == M(H(K2))`.

The conflict is unavoidable. Conflict resolution is the key to implement hash tables.

## Hash Table Conflict Resolution

### Chain Conflicts Together

Pointers seem to be the solution to many problems. Like the binary search tree, we use indirection to link certain items together. As shown below, instead of puting item direction in the table, we put a pointer in the table, which points to a single link list. Single link list holds all the items of the same bucket. In the following diagram:  
`H(key) = key`,  H() is identity function, just for illustration only
`M(hash,table_size)=hash%table_size`, M is frequently used simple mapping function: mod where `table_size=10`  

![Chained hash table](/img/hash-table-chain.dot.png)

For example,  
11: `M(H(11),10) = M(11,10) = 11%10 = 1` and  
21: `M(H(21),10) = M(21,10) = 21%10 = 1`  

Both end up in the same bucket so they are chained together.
With a good hash fucntion H and a reasonable table_size, sometimes with a good mapping function M, the average length(L) of the chain-link is small.

To lookup, we first map key to bucket, and then go through the single link list one by one. As pointer is involved, It takes O(cL) time. C=(pointre-read-time + item-read-time) / item-read-time. For item < 8 bytes, C>=2.

To insert, we first lookup to see if it already exists. If it does, then return or overwrite the item on spot. It takes O(CL). If it doesn't, we add it to the head in constant time. It also takes O(CL).

To remove, we first lookup to see if it exists. If it doesn't, then return. It takes O(CL). If it does, remove it by chaining the previous item to the next item. (Single Link List Algorithm) It also takes O(CL).

For each item with size S, it uses one extra pointer, 8 bytes. It also uses the whole extra table to store heads of single link list. Table_Size = DN, D is usually 2-3. For each item we use `S+8+8D` bytes. let E=(S+8+8D)/S, we use O(EN) spaces. 


`Lookup: O(CL)`  
`Insert: O(CL)`  
`Remove: O(CL)`  
`Space : O(EN)`  

As L is usually < 2, this is reasonable fast.

This seems to be the perfect solution, what could be the possible downside?

The space could be. For an integer item, `S=4, D=2 => E=7`; for a typical 8-byte item (items are pointers), `S=8, D=2 => E=4`. We typically use 4x necessary memory.

constant C is also a potential cause of concern. In normal cases, C=2 is very reasonable. But when N approaches to 1 million, cache misses happen more frequently, C could even double to 4.

So what's the reasonable next step? Well, Obviously, While maintainng the  performance, to remove pointers.

### Open Addressing Hash Table

Withoug chaining conflicts together, the only option is to find another bucket. The process of finding another empty spot is called probing. To insert an item while its bucket is taken, there are two ways. 

One way is to add a step parameter in mapping function `M(hash, table_size)` : `M'(hash, table_size, i)`, where i starts from 0 forward. G' usually increase with i. Keep increasing i until it reaches the end of table or empty spot. This adds uncertainty to the hash table. Anther simpler probing is called linear probing. It basically find the next position availabe from its bucket.

Below is a example of open addressing hashing table with linear probing of the same elements in chained hash table. A open addring hash table is often slightly larger than the advertized table size to have a overflow region at the end to accomodate keys that falls into the last bucket in the table.

Also noted is that the actual content of the table depends on the order of items inserted. Insert the same elements in another order renders very different picture.

![open addreing hash](/img/hash-table-open-addressing-linear-probing.dot.png)


Another way is to add step parameter to hash function H: `hash=H(key)` to `hash=H'(key,i)` where i starts from 0 forward. This is called rehashing. It rehashes until an empty position is found.

While saving space, open address hash table adds some uncertainties. At any point, you don't easily know where the item is stored. It doesn't seem to have clear invariants (that property of the data structure that's always true).  In chain-linked hash table, you know every item in the list, the bucket it maps to from key `M(H(key))` is the index its head is in. Before and after every operation you can put assertions on this property to make sure the operation is correctly performed. How do you verify that for an open address hash table? After insertion you can verify that lookup returns the correctly item and after deletion the lookup of the item fails. This is probably the best you can do. This is concerning for debugging and testing. You probabaly wouldn't count on it to launch a rocket.

Even if the open address hash table is corectly implemented, the other uncertainty during runtime is also a major drawback: There is no upbound for how many steps you have to rehash or remap before you find the item. The same item may be found in one try, or it may be found in 1000+ tries. Even worse, lookup of an item that's not in the table takes most of time.

If rehash/remap eventually leads to these two uncertainties, what can we do to improve? Just as the thought process along the way, to make lookup more consistent, we have to think what we can sacrifice. From unsorted list to sorted list, we sacrifice the performance of `insert` for a better performance of lookup. From sorted list to binary search tree, we sacrifice space to improve `insert` and `delete`. From BST to chain-linked hash table we sacrifice more space to improve on `lookup`, `insert` and `remove`. From chain-linked hash table to open address hash table we try to recoup some space back while maintain performance of `lookup`, `insert` and `remove`. Now to improve the uncertainty of `lookup` as well as `insert` and `remove`, is there anything left that we can sacrifice?

As it turns out, there is. To improve the worst case `lookup`, we can sacrifice best case `lookup`. 

### Robinhood Hashing

When we insert an item into hash table, we look for an empty position to put it in. Ideally the position is at its bucket. But if not, we keep looking for other positions via differnet strategies. Doing so we actually follow a hidden rule that when an item is in the table, we don't change its position anymore. Following this rule we can get a really bad position when the table is pretty full and an empty position is far away.




### Clustered Open Addressing Hashing



## Lessons Learned

**"There can be no progress, no achievements, without sacrifice, and a main's worldly success will be in the measure that he sacrifices." -James Allen, As a Man Thinketh**


## References
Sebastian Sylvan, [Robin Hood Hashing should be your default Hash Table implementation](https://www.sebastiansylvan.com/post/robin-hood-hashing-should-be-your-default-hash-table-implementation/)
