---
layout: post
title:  "Clustered Hashing: Modify On Iteration"
date:   2019-09-09 17:02:14 -0700
categories: algo
---

Iteration on the hash table is usually a constant operation. It doesn't modify the table. Since it's constant, the implementation is straight-forward. It's even simpler for open addressing hash tables: just go through every index of the table, skip empty items.

Some rare scenarios require hash tables to be able to modify itself while in iteration. For example, in a hash table of integers, you want to insert a even number for every odd number. 

With a constant iteration, you iterate all the keys. For each odd key that an even key is to be inserted, you insert it to an external container such as vector. Then you go through the vector and insert them all to the hash table.

```c++
    map<int,int> mp;
    vector<pair<int,int>> v;
    for( auto& [k,v]: mp)
    {
        if(k%2)
            v.push_back(make_pair(k+1,k+1));
    }
    for( auto i: v)
    {
        mp[i.first] = i.second;
    }
```

However, this is only good for a one-pass operation. If the condition is set so that the newly inserted item may also qualify, a more complex algorithm is needed aside hash table. Allowing insert on iteration can solve this problem without help of another container.

```c++
    map<int,int> mp;
    for( auto& [k,v]: mp)
    {
        if(k%2)
            mp[k+1] = k+1;
    }
```

The project I worked on (zeek.org) requires support of modification on iteration. Below is how it is implemented on Clustered Hashing.

## Basic idea

For each iteration, we maintain an iteration cookie to keep state of the iteration. The hash table keeps a list of iteration cookies for adjustment on modification.

```c++

```



## insert on iteration

## remove on iteration

## lookup on iteration

## References

- [Clustered Hashing]({% link _posts/2019-08-19-clustered-hashing.md %})
- [Clustered Hashing]({% link _posts/2019-08-26-clustered-hashing-basic-operations.md %})
- [Clustered Hashing Incremental Resize]({% link _posts/2019-09-02-clustered-hashing-incremental-resize.md %})
- [Clustered Hashing Modify On Iteration]({% link _posts/2019-09-09-clustered-hashing-modify-on-iteration.md %})
- Sebastian Sylvan, [Robin Hood Hashing should be your default Hash Table implementation](https://www.sebastiansylvan.com/post/robin-hood-hashing-should-be-your-default-hash-table-implementation/)
- [Malte Skarupke, Fibonacci Hashing: The Optimization that the WOrld Forgot](https://probablydance.com/2018/06/16/fibonacci-hashing-the-optimization-that-the-world-forgot-or-a-better-alternative-to-integer-modulo/)
