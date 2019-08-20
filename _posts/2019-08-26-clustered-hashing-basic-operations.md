---
layout: post
title:  "Clustered Hashing: Basic Operations"
date:   2019-08-26 17:02:14 -0700
categories: algo
---

## Introduction

Clustered Hashing is the flattened version of Chained Hashing. Chained Hashing links items of the same bucket together by pointers. Clustered Hashing clusters items of the same bucket together directly in the hash table.

Heres the visual comparison of Chained and Clustered Hashing:

|Chained Hashing|Clustered Hashing|
|---------------|-----------------|
|![Chained](/img/hashing/chain.dot.png)|![Clustered](/img/hashing/cluster.dot.png) |

## Item/Entry Attributes

Each entry in the table has 3 attributes:

name    |description
--------|-----------------------------------------------------------
position|position, index in the table the item is in.
bucket  |bucket of the item, or the perfect position of the item. <br>`b = M(H(key),table_size)`, <br>where M is the Map function to map hash to the range of [0, table_size-1], H is the hash function of key.|
distance|distance of the item's actual position from its bucket. distance >= 0 <br>When the item is in its expected bucket position, the distance = 0. Likely the item is off from its bucket because its bucket positon is occupied by another item. <br>A special distance = -1 is used to identify an empty item.

Items in Clustered Hashing Table satisfies the following equation:
    
    position = bucket + distance

So we only need to maintain a distance attribute for the item and derive bucket from it by
    
    bucket = position - distance

```c++
int Bucket(int position)
{//don't call it on empty items.
    ASSERT( position >=0 && position <= Capacity() && table[position].distance >= 0);
    return position - table[position].distance;
}
```

You can also derive bucket from key and table_size

    bucket = M(H(key),table_size)

but this is expensive to calculate every time. So we only use it when it's first inserted.

## Structure of the table

The table has an advertised size of table_size, which is 10 in diagram below for easy illustration of hash and bucket calculation. It also has an overflow area after the end of table to accomodate clusters with bucket close to the table size. the overflow size is 4 in the diagram. 

    internal_table_size = table_size + overflow_size.
    overflow_size = log2(table_size) + C, where C is a constant. 

Basic Properties:
1. (B1) Items are arranged in clusters. Items of the same bucket belong to the same cluster. cluster 7 has 4 items 17,27,37,47 all of bucket 7. 
2. (B2) Cluster is located at or after its bucket. Clusters of items in bucket b is called cluster b. For example in the diagram, cluster 1 starts from postion 1, its bucket(1). Cluster 2 starts from position 3, after its bucket(2).
3. (B3) Clusters are arranged in the order of its bucket. For example, Cluster 1 is before cluster 2. 
4. (B4) Each cluster is located at its closest possible position to its bucket while maintaining cluster properties 1,2, and 3.

Derived Properties:
1. (D1) No gaps between bucket of cluster and its head. If it had, we could have shifted the cluster b up to reduce the distance of the cluster b. The shifted arrangement has closer distance from its bucket. This conflicts to Basic Property 4.
2. (D2) No gaps inside the cluster. If it had, we could have shifted the item of the cluster right after that gap up to improve its distance. This makes the arrangement conflict to Basic property 4.
3. (D3) Distance maintained in the item is non-negative. Each item in the cluster maintains the distance from its actual position to its perfect position (bucket). Since clusters reside at or after its perfect position, the distance is always non-negative. 
4. (D4) Distances of items in the same cluster are in continuous ascending order. Since items of the same cluster are contiguous, the distances are continuous. For exampe, cluster 2 has 3 items. The head of the cluster is 12, with distance of 1. It contains 3 items with distances of 1,2, and 3.

The operations are all based on these 4 basic and 4 derived properties.

![Clustered](/img/hashing/cluster.dot.png)

```c++
template<class Key, class Value> 
class Dictionary
{
    struct Entry
    {
        Key key;
        Value value;

        uint32 hash;
        short distance;
        short log2_buckets;//the size of table this entry belongs to.

        bool Empty(){return distance < 0;}
        void SetEmpty() {distance = -1;}
        bool Equal(const Entry& other){ return hash == other.hash && key == other.key;}
        bool operator==(const Entry& other){ return Equal(other); }
        bool operator!=(const Entry& other){ return !Equal(other); }
        Entry(Key key, Value& value): key(key),value(value), distance(0), log2_buckets(0){}
    };
    
    Entry* table;
    char log2_buckets; //2^log2_buckets is the table_size.
    int table_size;
    int internal_table_size;

    uint32 Hash(const Key& key) const;
    int BucketByHash(uint32 hash) const;
    int BucketByKey(Key& key) const
    {
        return BucketByHash(Hash(key));
    }
    int Buckets() { return 1<<log2_buckets;}
    int Capacity() { return (1<<log2_buckets) + log2_buckets;}
};

```

## Lookup

### Find the head and tail of the cluster b 

- Based on B2, we know the head of cluster is at or after b. So the search range starts from b.
- Based on B3, From b on 
    - we may first see partial or full of a cluster smaller than b0.
    - We then may see other full clusters b1, b2, ..., b-1.
    - we then may see the cluster of b.
    - no gaps between b and the tail of cluster b.
    - we then may see other full clusters b+1, b+2, ...

Solution: 
1. From b on, the first position with bucket of b is the head of the cluster b if it exists. 
2. From b on, the last position with bucket of b is the tail of the cluster b if it exists. 
3. When we reach cluster c > b (B3), or an empty position(D1&D2), or the end of the table. We know cluster b doesn't exist.

For example, head of cluster 2 is 3, the first position at or after 2 whose bucket is 2. The tail of cluster 2 is 5, the last position whose bucket is 2. 

If we don't find positions of bucket b before we reach the end of table, find empty space or the cluster greater than b, we know cluster b doesn't exist.

For example, when we look for head of cluster 4. Starting from position 4, we first see part of cluster 2. We then see empty space of position 6. If we had cluster 4, it must reside before empty space, ie 6. So we konw cluster 4 doesn't exist.

![Clustered](/img/hashing/cluster.dot.png)

```c++
int HeadOfCluster(int bucket)
{ 
    ASSERT(bucket>=0 && bucket < Buckets());
    int i = bucket;
    while( i < Capacity() && !table[i].Empty() && Bucket(i) < bucket)
        i++;
    if( i >= Capacity() || table[i].Empty() || Bucket(i) > bucket)
        return -1; //no such cluster.
    return i;
} 

int TailOfCluster(int bucket)
{ 
    ASSERT(bucket>=0 && bucket < Buckets());
    int i = bucket;
    if( table[i].Empty() || Bucket(i) > bucket)
        return -1; //no such cluster.
    while( i+1<Capacity() && !table[i+1].Empty() && Bucket(i+1) <= bucket)
        i++;//stop at the next larger bucket.
    if( Bucket(i) == bucket )
        return i;
    return -1;
} 
```

### Find the Head and Tail of the cluster position p belongs to.

This is straight-forward. As we know items of the cluster stay together, we only need to look up and down. The farthest position up with the same bucket of position is the head of my cluster and the farest position down with the same bucket of position is the tail of my cluster.

```c++
int HeadOfMyCluster(int position)
{ 
    ASSERT(position >=0 && position < Capacity() && !table[position].Empty() );
    int i = position;
    int bucket = Bucket(i);
    //[bucket,position] cannot have empty items. or it breaks the invariant.
    while( i>=bucket && Bucket(i) == bucket)
        i--;//stops just before head of cluster.
    return i == bucket ? i : i+1;
} 
```
### When the item at position is part of the cluster, find the Tail of my cluster

```c++
int TailOfMyCluster(int position)
{ 
    ASSERT(position >=0 && position < Capacity() && !table[position].Empty() );
    int i = position;
    int bucket = Bucket(i);
    while( i<Capacity() && Bucket(i) == bucket)
        i++;//stops just after tail of cluster.
    return i-1;
} 
```

### Lookup specific key

We find the head of the cluster b first. If it exists, we go through cluster b to look for specific key. If key is equal we have found it. When not found, the position it stops is the insert position had the key been inserted. 

For example, Lookup(22). we look continuous from position 2. We compare bucket to locate head of cluster 2, which is position 3. We then going through the cluster 2 and find 32 on position 4.

Lookup(26), we start from position 6. Because position 6 is empty, we know immediately cluster 6 doesn't exist. If we want to insert 26, position 6 is the position to insert.

Lookup(23), we start from position 3. Looking for head of the cluster 3 ends with position 6 as its empty. So we know cluster 3 doesn't exist. If we want to insert 23, position 6 is the position to insert.

```c++
Value* Lookup(Key& key)
{
    int position = LookupIndex(key);
    if( position < 0)
        return nullptr;
    return &table[position].value;
}

int LookupIndex(Key& key, int* insert_position = NULL)
{
    int bucket = BucketByKey(key);
    ASSERT(bucket>=0 && bucket < Buckets());
    //skip items with smaller buckets
    for(int i = b; i < Capacity() && !table[i].Empty() && Bucket(i) < bucket)
        i++;
    if( i >= Capacity() || table[i].Empty() || Bucket(i) != bucket)
        return -1; //not such cluster exists.
    //i is head of cluster, going through the cluster.
    for(; i < Capacity() && !table[i].Empty() && Bucket(i) == bucket){
        if(table[i].key == key)
            return i;
    }
    if(insert_position)
        *insert_position = i;
    return -1;
}

```

## Insert
When we insert an item into its cluster, we always append it to the tail of the cluster. So TailOfCluster(b)+1 is the insertion point of item of bucket b. It's also called the end of cluster b.

I1. To insert key K with bucket b. K has to be appended to the tail of the cluster.

Proof:
- B4 tells us all items are in their best possible positions whiling maintaining the properties of the Clustered Hashing. So before the head of the cluster b there is no position for key to insert. Had there been, head of the cluster b is not the best position. So the position to insert key is at or after the head of cluster b.
- D2 tells us there are no gaps inside cluster b. So we cannot put k into a gap inside the cluster.
- So to put k in any position inside the cluster, we have to swap out the item in that position. 
- B4 tells us all items are already in their best position. If we put key in and take out the item in that position, we face the same problem to insert the item in hand of the same cluster as the original key without improve any distances of the cluster. So it's waste of time.
- B1 tells us items of the same cluster are together. If we can't put the key before the head of the cluster, nor do we want to put it inside the cluster, the only option left is to put it at the end of the cluster, after tail of the cluster.

I2. To swap the item, we take out the head of the next cluster and append it to the tail of the same cluster.

Proof:
- When we insert an item, we append to the end of the cluster.
- B1 tells us items of the same bucket cluster together. If the end of the cluster b is the middle of the next cluster c, that means insertion of the item causes some items of cluster c before it. Now cluster b is mixed with the item of cluster c. This contradicts with B1. So the append position is the head of the next 

I1 and I2 forms the algorithm of insertion.

We lookup first. If key is already in the table, we update the value. 
If it's not, the end (tail of cluster + 1) of the cluster b is the insert position p of key.

If p is empty, we insert to that position p and done. 

If there's already an item in p, we take it out and put key in. We then insert the taken-out item to the end of its cluster and adjust its distance. It triggers chain effect and ends on the first condition. The worst case is the position reaches to the end of the table and there's no place to insert the item. This is when we size up the table and insert the item again.

Append to the end of its cluster. Swap out the item in that position if not empty. Continue until all items are adjusted.

```c++
void  Insert(Key& key, Value& value){
    int insert_position = 0;
    int position = LookupIndex(key, &insert_position);
    if( position > 0 )
    {
        table[i].value = value;
        return;
    }
    Entry entry(key, value);
    entry.distance = insert_position - BucketByKey(key);
    return Insert(entry, insert_position);
}

void Insert(Entry entry, int position)
{//insert entry into position. 
    while(true)
    {   //take out the head of cluster and append to the end of the cluster.
        if(position >= Capacity())
        {
            SizeUp(); //copied all the items to new table.
            Insert(entry.key, entry.value);
        }
        if(table[position].Empty())
        {   //the condition to end the loop.
            table[position] = entry;
            return;
        }        
        //position is always head of the cluster.
        ASSERT( position == HeadOfMyCluster(position) );
        //the to-be-swapped-out item appends to the end of its cluster.
        auto t = table[position];
        int next = TailOfMyCluster(position)+1;
        t.distance += next - position;
        //swap
        table[position] = entry;
        entry = t;
        //need to increase distance along the way.
        //move the head of my cluster to the end of my cluster.
        position = next; //append to the end of the current cluster.
    } 
}

```

### insert 10,11,12,17

The simplest case. All are inserted to their bucket position respectively without adjustment of other elements.

![insert 10,11,12,17](/img/hashing/cluster-insert-10-11-12-17.dot.png)

### insert 21

21 belongs to cluster 1. Cluster 1 has only one item (11). Head and tail of Cluster 1 is 1. 21 is appened to the end of cluster 1, which is at position 2.
12 is currently in position 2. So we take out item 12 and put 21 in.
12 is append to the end of cluster 2, which is at the position 3 before being taken out.
position 3 is empty. Adjustment ends.

![insert 21](/img/hashing/cluster-insert-21.dot.png)


### insert 27

![insert 27](/img/hashing/cluster-insert-27.dot.png)

### insert 32

![insert 32](/img/hashing/cluster-insert-32.dot.png)

### insert 37

![insert 37](/img/hashing/cluster-insert-37.dot.png)

### insert 47

![insert 47](/img/hashing/cluster-insert-47.dot.png)

### insert 50

![insert 50](/img/hashing/cluster-insert-50.dot.png)

### insert 61

![insert 61](/img/hashing/cluster-insert-61.dot.png)

### insert 73

![insert 73](/img/hashing/cluster-insert-73.dot.png)


## Remove


```c++
void Remove(Key& key)
{
    int position = LookupIndex(key);
    if( position < 0 )
        return; //not in the table.

    Remove(position);
}

void Remove(int position)
{   //fill the empty position with the tail of the cluster of position+1.
    while(true)
    {
        if( position == Capacity() - 1 || table[position+1].Empty() || table[position+1].distance == 0)
        {//no next cluster to fill, or next position is empty or next position is already in perfect bucket.
            table[position].SetEmpty();
            return;
        }
        int next = TailOfMyCluster(position+1);
        table[position] = table[next];
        table[position].distance -= next - position; //adjust distance.
        position = next;
    }
}
```



## References
Sebastian Sylvan, [Robin Hood Hashing should be your default Hash Table implementation](https://www.sebastiansylvan.com/post/robin-hood-hashing-should-be-your-default-hash-table-implementation/)
