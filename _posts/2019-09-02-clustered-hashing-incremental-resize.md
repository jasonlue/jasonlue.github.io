---
layout: post
title:  "Clustered Hashing: Incremental Resizing"
date:   2019-09-02 17:02:14 -0700
categories: algo
---

Previous post doesn't touch resize of the hash table. Usually resize of the hash table is straight-forward. A new table is created; Insert all items in the orginal table to the new table; And finally destroy the old table.

This is an expensive O(N) operation compared to lookup, insert and remove. Is there a faster way than O(n)? There is not. However, the O(N) operation in the middle of O(1) operations causes system hiccups. For a normally application, this is acceptable. There are some realtime systems where near-constant operating time is important. In these systems constant inflow of data can only be buffered for so long or buffer overrun and data loss may happen.

For this reason, an incremental resizing mechanism is designed to only move a small chunk of items from old table to new table on every insert. There are a few potential issues with this mechanism

- Uses double amount of memory during transition period.
- Lookup performance is detrimented. Lookup needs to decide which table to operate on. Or it has to look up in 2 tables.
- When the table is big, the transitory period is not so transitory anymore, especially when the move from old to new depends on insert and after resizing insertion becomes less frequent. In some cases transition may stay forever.

## Resize

The no-pointer property of open table gives Cluster Hashing a really neat way to size up: It can actually size up in place. Because it doesn't require readjustment of pointers, it can reallocate larger size with the existing table as first part of the table. We can easily achieve this with realloc() C function call. It the table has some free memory after it, it simply extends the memory space. If it doesn't, relalloc() allocates a larger memory and copy the existing table to the new memory as the first part. This should be a very fast operation.



![Sizeup](/img/hashing/cluster-sizeup.dot.png)

## Remap

After table sizes up, all items become old and need to be relocated to positions based on new table_size. To remove the item from the table and then insert it back again is called remapping. We know that from position 0 to previous table_size are old items right after resize. We mark this as remap range [0,remap_end]. New inserts may happen in this range or after it, ie. in  range (remap_end, table_size). We know that range (remap_end, table_size) contains only items mapping to the new table size. Range [0,remap_end] contains both old and new items.

We remap the items from bottom of the remap range until the top. As incremental resizing indicates, we remap a small set of items for every new inserts. Remap range [0,remap_end] usually shrinks after each remap. However, Remap may also cause the range to grow due to relocation of the 

How do we know if an item is an old item or not? We need to know that to remap the old item to the new position.

We don't have to know. For each item of key K, we recalculates its bucket based on new table size: 
``bucket=M(H(K),table_size)`` and based on its position we can calculate its actual bucket: ``bucket=position-distance``. If two buckets are the same, we don't have to remap it. This is the same for all new items and some old items. In fact, when we design a smart map function, only half of the old items need to be remapped. This is a big boost in performance compared to other incremental resizing mechanisms.


```c++
void Remap()
{
    int left = DICT_REMAP_ENTRIES; //max entries to remap at a time.
    //Remap(remap_end) may change remap_end on return if Remap adjusted items expands remap range.
    while(remap_end >= 0 && left > 0)
    {   //< if we actually Remap(remap_end) successfully, remap_end may be changed. check it again.
        if(!table.Empty(remap_end) && Remap(remap_end))
            left--;
        else
            remap_end--;
    }
}

bool Remap(int position)
{
    ASSERT(position >= 0 && position < Capacity());
    int current = Bucket(position);
    int expected = Bucket(table[position].hash, log2_buckets);
    if( current == expected)
        return false;
    Entry e = Remove(position);
    e.distance = 0;
    Insert(e, expected);
    return true;
}
```

In the diagram below, left table shows the state right after the size up. dark grey area is the overflow area. remap_end is set the the last position before table resizes. The old table_size = 10, the new one is 20. During first remapping, we go from bottom (13) up. Position 13,12,11 are empty. No adjustment. Position 10 is 47. new bucket is 47%20=7, the same as before. No change. Position 0 is 37. bucket = 37 % 20 = 17. new bucket is 17 while current bucket is 7. So take 47 out by calling remove(9). Removal of 9 causes position 10 to shift up. So 47 is relocated to position 9. insert(37) put item 37 to bucket 17.

![cluster-remap-37](/img/hashing/cluster-remap-37.dot.png)

## Lookup

Post Road to Clustered Hashing explains the remap mechanism. A step parameter is applied during mapping from hash to bucket. In clustered Hashing table the distance from its bucket is actually the step.

![hash-table-basic](/img/hashing/remap.dot.png)

Now after resize, new items and old items are mixed together. To find the new item's bucket, we apply current table_size (S<sub>0</sub>) to its hash. To find the old item's bucket, we apply its hash to the table_size when the item is inserted(S<sub>-i</sub>). Now the question is, how do we know the table_size when its inserted? 

We could record it somewhere, but then we need to map the key to it. This still remains a problem. Instead, we don't have to know. If we record all the previous table sizes, we can try them all out from current table size (S<sub>0</sub>) to  previous table sizes (S<sub>-i</sub>). In essense, we expand the mapping function again as shown below:

![hash-table-basic](/img/hashing/cluster-remap.dot.png)

Actually, we can be smarter than trying all sizes out. Because we know that mix of items only happens during remapping process (to map old items to new table sizes). We clear the table size queue when we finish remapping. The table size queue only contains just a few, most of time only one, sizes when remapping is not fast enough to complete. So the lookup of old items and non-existing items takes at most sizeof(queue) tries if the usual lookup doesn't find the item. 

If we only grow the table, We only need to record the start of queue because we can infer all the sizes thereafter. 

We can even do better than that with regards to old items if we are willing to break an implicit rule during hash table lookup. When we look up an item in the hash table, we assume that lookup doesn't modify the table. 

From simple Open Addressing Hashing to Clustered Hashing, we've broken a few rules to boot the performance:

- insert can modify existing item's positions
- remove can modify existing item's positions
- look can modify existing item's positions.

Here's the optimized algorithm: When we successfully looked up an item successfully in more than one tries, we remap the item to a new positions according to current table size. It sacrifices a bit of time on the first lookup to make the sequential lookups on the same item faster. This is also a good complement to remap when inserting new items. If inserting new items becomes less frequent but lookup becomes more frequent, this improvement make a big difference.

## Fibonacci Hashing

To get the bucket from key, we use 

    bucket=M(H(key),table_size)

Up to now we haven't touched hash function and map function. Hash functions are extensively studied but we seem to stick with a very simple map function:

    M(hash,table_size) = hash % table_size.

There are a few issues with this map function:
- table_size need to be a prime to make it really effective. And the increasingly large primes are hard, and expensive to find. When we compromise with just an odd large integer, the mapping function may perform worse. It results in more collisions.
- hash % table_size looks very simple, but is expensive to calculate. It uses division, thus a loop is involved.
- the bucket mapping depends heavily on the hash function. A bad hash function, for example, an integer identity function, may cause a lot of collisions.

Fibonacci Hashing is a map function from hash to a range of (0, table_size). It solves all three issues above and add one more benefit in the case of incremental resizing.

- table_size is always 2^N.
- bucket = M(hash,N) = (hash * 2^64 / phi) << (64-N) >> (64-N), phi = Golden Ratio = 1.618033987...

2^64/phi = 11400714819323198485



## References

- Sebastian Sylvan, [Robin Hood Hashing should be your default Hash Table implementation](https://www.sebastiansylvan.com/post/robin-hood-hashing-should-be-your-default-hash-table-implementation/)
- [Malte Skarupke, Fibonacci Hashing: The Optimization that the WOrld Forgot](https://probablydance.com/2018/06/16/fibonacci-hashing-the-optimization-that-the-world-forgot-or-a-better-alternative-to-integer-modulo/)
