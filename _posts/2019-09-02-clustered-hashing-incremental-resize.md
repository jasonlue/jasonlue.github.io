---
layout: post
title:  "Clustered Hashing: Incremental Resizing"
date:   2019-09-02 17:02:14 -0700
categories: algo
---

Previous post doesn't touch resize of the hash table. Usually resize of the hash table is straight-forward. A new table is created; Insert all items in the orginal table to the new table; And finally destroy the old table.

This is an expensive O(N) operation compared to lookup, insert and remove. Is there a faster way than O(n)? There is not. However, the O(N) operation in the middle of O(1) operations cause system hiccups. For a normally application, this is acceptable. There are some realtime systems where near-constant operating time is important. In these systems constant inflow of data can only be buffered for so long or buffer overrun and data loss may happen.

For this reason, an incremental resizing mechanism is designed to only move a small chunk of items from old table to new table on every insert. There are a few potential issues with this mechanism

- Uses double amount of memory during transition period.
- Lookup performance is detrimented. You need to decide which table to lookup an item. Or you have to look up in both table.
- When the table is big, the transitory period is not so transitory anymore, especially when the move from old to new depends on insert and after resizing insertion becomes less frequent. In some cases transition may stay forever.



## Fibonacci Hashing


## Resize

# Incremental Resizing




