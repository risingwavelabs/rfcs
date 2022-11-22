---
feature: minimize_height_join_ordering
authors:
  - "Dylan Chen"
start_date: "2022/11/21"
---

# Minimize Height Join Ordering

## Motivation

As we know, our system is a streaming processing system which aims to provide real time low latency for our users. However, the current join ordering algorithm in our system can only construct a left deep tree shape join ordering which is not compatible with our low latency target, because our barriers need to go through the whole left deep tree to be collected. In order to reduce the barrier latency, I think we should minimize the height of our join tree to construct a bushy tree instead of a left deep tree.

## Design

### Example

Considering an example TPCH Q9:

```sql
select
      nation,
      o_year,
      round(sum(amount), 2) as sum_profit
    from
      (
        select
          n_name as nation,
          extract(year from o_orderdate) as o_year,
          l_extendedprice * (1 - l_discount) - ps_supplycost * l_quantity as amount
        from
          lineitem,
          part,
          supplier,
          partsupp,
          orders,
          nation
        where
          s_suppkey = l_suppkey
          and ps_suppkey = l_suppkey
          and ps_partkey = l_partkey
          and p_partkey = l_partkey
          and o_orderkey = l_orderkey
          and s_nationkey = n_nationkey
          and p_name like '%yellow%'
      ) as profit
    group by
      nation,
      o_year
    order by
      nation,
      o_year desc;
```

<img width="556" alt="image" src="https://user-images.githubusercontent.com/9352536/203251137-79026520-23ee-4080-ba4c-b1fe2581d7c0.png">

There are 6 tables joining together. 

<img width="1034" alt="image" src="https://user-images.githubusercontent.com/9352536/202991793-664ea3f9-3838-4e5f-af6c-e5416140ca40.png">

Currently our system will give a left deep tree join ordering: lineitem, part, partsupp, supplier, orders, nation. The height of this left deep tree is 5.

<img width="1036" alt="image" src="https://user-images.githubusercontent.com/9352536/202991855-998a6d28-a366-4120-8765-be3d5de20474.png">

If we minimize the height of the join tree, we will construct a bushy tree join ordering: (part, partsupp), (lineitem, orders), (supplier, nation). The height of the bushy tree is 3.


Theoretically, given n tables we can construct a bushy tree with height equaling the ceil of log2(n).


### Algorithm

```
Input: A set of connected relations R = {R1, R2, ... RN}.
Output: A join tree.

T = R

while |T| > 1 {
  T = the smallest size partition of the set T whose subsets have at most 2 elements and the 2 elements of the subset need to be connected directly in the join graph.
}


Return the only element of T.
```

Actually, when we enumerate the partition of the set T, we can do some branch and bound pruning, because we always know the lower bound of the smallest partition is `|T| / 2`. Once we reach this lower bound, we can stop current enumeration of the partition.

Note: [Partition of a set](https://en.wikipedia.org/wiki/Partition_of_a_set)
In mathematics, a partition of a set is a grouping of its elements into non-empty subsets, in such a way that every element is included in exactly one subset.

Go through the algorithm with TPCH Q9.

```
1:  
T = R = {lineitem, part, partsupp, supplier, orders, nation}
possible partitions:
partition1 = { part, partsupp | lineitem, orders | supplier, nation }.   // size = 3, winner.
partition2 = { part, lineitem | orders | supppart | supplier, nation }.  // size = 4
partition3 = { lineitem, supplier | part | supppart | orders | nation }. // size = 5
partition4 = { lineitem | supplier | part | supppart | orders | nation }. // size = 6
...

2:
T = { part join partsupp, lineitem join orders, supplier join nation }
possible partitions:
partition1 = { part join partsupp, lineitem join orders | supplier join nation } // size = 2, we choose first winner.
partition2 = { part join partsupp, supplier join nation | lineitem join orders } // size = 2, winner too.
partition3 = { part join partsupp | supplier join nation | lineitem join orders } // size = 3
...

3:
T = { (part join partsupp) join (lineitem join orders), supplier join nation} 
possible partitions:
partition1 = { part join partsupp joinlineitem join orders, supplier join nation }  // size = 1, winner.
partition1 = { part join partsupp joinlineitem join orders | supplier join nation } // size = 2

4:
T = { (part join partsupp) join (lineitem join orders) join (supplier join nation)} // final result.
- 
```


## Future possibilities

Obviously, this algorithm uses the height of the join tree as its cost and doesn't rely on any statistics. If we have a volcano/cascade style in the future, we can also use the height as the cost of the join tree and use some join enumeration rule to achieve the same result to verify the new introduced optimizer.
