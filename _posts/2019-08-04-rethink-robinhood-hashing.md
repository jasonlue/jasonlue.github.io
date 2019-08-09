---
layout: post
title:  "Flattened Chain-link: A Special Case of Robinhood Hashing"
date:   2019-08-15 17:02:14 -0700
categories: algo
---

## Introduction

I've recently been tasked to optimize the memory usage of a real time application (zeek.org). This application captures network packet and drives a lot of scripts which uses dictionaries. Under certain stress tests it has created a million dictioniaries in a matter of minutes. All of sudden how efficient the dictionary uses memory matters a lot to the overall application memory consumption.

Unsurprisingly the dictionary is implemented as chain-linked hash table. Why chain-lined hash table is so popular? Let's step back a little to start from the motivation to have a hash table and see what leads us to chain-lined hash table.

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

Wait, you'd think, insert and remove can be as fast as O(lgN) if we use single link list. But can you do binary search in an ordered link list? You simply have to look one by one. This solution is worst the basic unordered one in every aspect. And don't forget, by using link list, you use more space, as pointers occupy space, not small, 8 bytes each. We will revisit this later.

So keep list in order improves lookup greatly, but with heavy sacrifice on insert and remove. Is this a bad idea at all? Not necessarily. If you have a dictionary you insert once, never remove it, then most of the time is lookup. It is still a sensible solution.

### Binary Search Tree (BST)

The next idea is to sacrice space. If for each item we keep two pointers, we come up with a binary search tree. The idea of BST is keeping items ordered, but in a special binary tree form, so that insert and remove are almost as fast as lookup. After finding the item or position, you don't need to shift items back and forth. You simply change pointers to insert and remove, which takes constant time. Assume the binary tree is balanced because a variation of the BST, red black tree, can keep the tree balanced with a small price (`lgN` swaps).

`Lookup: O(lgN)`
`Insert: O(lgN)`
`Remove: O(lgN)`
`Space : O(N)`

This is good, isn't it? Not so fast.  As each item now requires two pointers, and each pointer is 8 bytes in 64bit CPU system, each item consumes 16 extra bytes. Suppose average item size is s, we have D=(8+8+s)/s. The space is actually O(DN). Theorectically O(DN) = O(N), but in reality you still uses D times memory than necessary. For an integer of 4 bytes, D=5.

Doesn't indirection of pointers consume computer time? For each item, you have to follow its pointers to another item. Suppose read item takes x amount of time, and load extra pointer take y amount of time, lookup actually takes O(lg((x+y)/xN) = O(lg(CN)) = O(lgC+lgN), given C=(x+y)/x. When x is small, such as read an 4-byte integer, load 8-byte takes at least same amount of time. So you could easily double the lookup time. It can be even worse... When items approach certain large counts, random pointers cause cache misses: the item pointed to is not in the current cache and CPU needs to reload it from the RAM. This is very costly. We will revisit it later. So the acutally binary search tree performance is:

`Lookup: O(lgCN) = O(lgC + lgN)`
`Insert: O(lgCN) = O(lgC + lgN)`
`Remove: O(lgCN) = O(lgC + lgN)`
`Space : O(DN)`

Sacrifice in insert and remove to the extreme make lookup as fast as O(lgN). To go further, we only have space to sacrifice. If we leave spaces among items on purpose, can we do better on lookup? Yes we can! Here comes in picture: hash table.

### Hash Table

For a item with key, we calculate an integer called hash through a function 
F: `hash = F(key)`. We allocate a table with table_size a few times larger than item count. We then map hash to the index of the table called bucket through another function G: `bucket = G(hash, table_size)`. We then directly store item on position bucket, as shown below:

![hash-table-basic](/img/hash-table-basic.dot.png)

To lookup item, we do the same: `hash=F(key), bucket=G(hash, table_size)` table[bucket] is the item. Insert and remove are the same. find the bucket, then put to position or remove from the position (special mark). so,

`Lookup: O(1)`
`Insert: O(1)`
`Remove: O(1)`
`Space : O(DN)`

1/D is called load factor, which is usually 30-50%. So D is 2x - 3x N. For a small item such as integer, the space is even smaller than BST!

Sacrificing the space a few times makes lookup, insert and remove to be as fast as possible and ultimate goal is achieved?

Not so fast.

What if two items with K1 and K2, after function F, reaches the same hash?
`F(K1) == F(K2)`? With the same hash, both items find the same bucket. Follow the previous logic item 2 will overwrite item1 on insert. Even if two hashes are different, `F(K1) != F(K2)`, mapping hash to bucket can still result in same bucket: `G(F(K1)) == G(F(K2))`.

The conflict is unavoidable. Conflict resolution is the key to implement hash tables.

## Hash Table Conflict Resolution

### Link Conflicts Together

Pointers seem to be the solution to many problems. Like the binary search tree, we use indirection to link certain items together. As shown below, instead of puting item direction in the table, we put a pointer in the table, which points to a single link list. Single link list holds all the items of the same bucket. In the following diagram, assume F() is identity function and G is mod: `F(key) = key`, `G(hash,table_size)=hash%table_size`, and `table_size=10`

![Chain-linked hash table](/img/hash-table-chain.dot.png)

For example,  
11: `G(F(11)) = G(11,10) = 11%10 = 1` and 
21: `G(F(21)) = G(21,10) = 21%10 = 1`
Both end up in the same bucket so they are chained together.
With a good hash fucntion F and a reasonable table_size, sometimes with a good mapping function G, the average length(L) of the chain-link is small.

To lookup, we first map key to bucket, and then going through the single link list one by one. As pointer is involved, It takes O(cL) time. C=(pointre-read-time + item-read-time) / item-read-time. For item < 8 bytes, C>=2.

To insert, we first lookup to see if it already exists. If it does, then return or overwrite the item on spot. It takes O(CL). If it doesn't, we add it to the head in constant time. It also takes O(CL).

To remove, we first lookup to see if it exists. If it doesn't, then return. It takes O(CL). If it does, remove it by chaining the previous item to the next item. (Single Link List Algorithm) It also takes O(CL).

For each item with size S, it uses one extra pointer, 8 bytes. It also uses the whole extra table to store heads of single link list. Table_Size = DN, D is usually 2-3. For each item we use `S+8+8D` bytes. let E=(S+8+8D)/s, we use O(EN) spaces. 


`Lookup: O(CL)`
`Insert: O(CL)`
`Remove: O(CL)`
`Space : O(EN)`

As L is usually < 2, this is reasonable fast.

This seems to be the perfect solution, what could be the possible downside?

The space could be. For an integer item, `S=4, D=2 => E=7`; for a typical 8-byte item (to put pointers in), `S=8, D=2 => E=4`. We typically use 4x necessary memory.

constant C is also a potential cause of concern. In normal cases, C=2 is very reasonable. But when N approaches to 1 million, cache miss happens more frequently, C could even double to 4.

So what's the reasonable next step? Obviously, to remove pointers.

### Open Address Hash Table

Withoug chaining conflicts together, the only option is to find another bucket. The process of finding another empty spot is called probing. To insert an item while its bucket is taken, there are two ways. 

One way is to add a step parameter in mapping function `G(hash, table_size)` : `G'(hash, table_size, i)`, where i starts from 0 forward. G' usually increase with i. Keep increasing i until it reaches the end of table or empty spot. This adds uncertainty to the hash table. Anther simpler probing is called linear probing. It basically find the next position avaialbe from its bucket.

Another way is to add step parameter to hash function F: `hash=F(key)` to `hash=F'(key,i)` where i starts from 0 forward. This is called rehashing. It rehashes until an empty position is found.

While saving space, open address hash table adds a lot of uncertainties to the data structure. At any point, you don't easily know where you item is stored. This is a concern for debugging. Another uncertainty is most items are in good positions but some are in really bad poisitions. i could go beyond 1000 and it adds a lot of variance to the lookup. What happens if the first probing doesn't find a empty position at all and the table ends?

Among all the issues, huge variance is the biggest concern of all, especially in embedd and realtime systems. If all of sudden one lookup take 1000x average, the system is greatly interrupted and hiccups are expected.


