---
layout: post
title:  "Clustered Hashing: Modify On Iteration"
date:   2019-09-09 17:02:14 -0700
categories: algo
---

Iteration on the hash table is usually a constant operation. It doesn't modify the table. At the same time, modifying the table during iteration is usually not supported as this is not a usual requirement.

## Iteration

To keep track of the state of each iteration, we maintain an iterator:

```c++
struct IterCookie
{
    Dictionary* d;
    int next; //track the position to be iterated next.
    vector<Dictionary::Entry> inserted; //items inserted before next iteration point.
    vector<Dictionoary::visited;//items at or after next but is already visited.
}


Value NextEntry(Key& k, IterCookie& it)
{
    if(!it.inserted.empty())
    {
        Value v = it.inserted.back().value;
        k = it.inserted.back().key;
        it.inserted.pop_back();
        return v;
    }
    
}
```


## insert on iteration

Inserting an item into position p, based on insert algorithm, it adjust the items in range [p,q]. There is an iteration in progress. The next iteration position is store in the iteration cookie c: c->next.



## remove on iteration



## lookup on iteration

## Remap on iteration

## References

- [Clustered Hashing]({% link _posts/2019-08-19-clustered-hashing.md %})
- [Clustered Hashing]({% link _posts/2019-08-26-clustered-hashing-basic-operations.md %})
- [Clustered Hashing Incremental Resize]({% link _posts/2019-09-02-clustered-hashing-incremental-resize.md %})
- [Clustered Hashing Modify On Iteration]({% link _posts/2019-09-09-clustered-hashing-modify-on-iteration.md %})
- Sebastian Sylvan, [Robin Hood Hashing should be your default Hash Table implementation](https://www.sebastiansylvan.com/post/robin-hood-hashing-should-be-your-default-hash-table-implementation/)
- [Malte Skarupke, Fibonacci Hashing: The Optimization that the WOrld Forgot](https://probablydance.com/2018/06/16/fibonacci-hashing-the-optimization-that-the-world-forgot-or-a-better-alternative-to-integer-modulo/)
