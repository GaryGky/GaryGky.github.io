---
title: complete_system
date: 2022-12-11 21:07:42
tags: [system design]
banner_img: /img/bg/lighthouse.jpg
index_img: /img/cover/complete-system.png
---

![Auto complete system overview](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=Yzc4MmE3NGQ4ZjEwZjg3MWRlYzhiZTBlYTliMWY2YzRfYjZla1FJSHhYTnJZZkJVZEQ2S0R4SUM3TWJLdDBjWkdfVG9rZW46Ym94Y25aZkp5V0hGdDdOQjlZWXNwZDk4bmVmXzE2NzA3NjQxMDU6MTY3MDc2NzcwNV9WNA)

The requirement for auto complete system should be:

- Lightening fast reads. When users type in keywords, the service responds in a very fast way. Facebook found that an autocomplete needs to return suggestions in [less than 100 ms](https://www.facebook.com/notes/10158791367817200/).

- Relavancy of suggestions: should return the most frequent query result from the database for one keyword.

![Google Search engine's auto complete](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGE5MTkyYThkODhjMDRlNWJlZGNiZmI2OTk4NjU3NDhfS2NTR0NvSUJ3N3hOVk01ZVJ5dmY4R1pCUXNNRVJVbk1fVG9rZW46Ym94Y251aEx3blo3eEI0eWV2OWJoanFIR29lXzE2NzA3NjQxMDU6MTY3MDc2NzcwNV9WNA)

# Understand the problem and Establish design scope

- Requirement Detail: Search Top-5 frequent suggestions for a keyword

- User Scope: Assume there are 10 million DAU

- No spell check

- One of the important features of this system is that the suggested result is not user-specific. While on Facebook there is a lot of user-specific data such as **feed**. 

## Back-of-the envelope-estimation

**Traffic**: Assume that on average, one user will search 10 times a day and in each query there are 20 characters  

QPS: 10,000,000 * 10 * 20 / (24 * 60 * 60) = 24000

Peak QPS: 48000

**Storage**: each query contains 20 characters, so the size of a query is 20 bytes. It generates (10, 000, 000 * 10 * 20) bytes per day and assumes that the new query takes up 20% of the total queries.

So the storage space increases 0.4 GB per day.

# High Level Design

The client sends a request to the API gateway. The load balancer will choose one web server to handle the query request. 

The Web Server will first compute the top-5 frequent keyword queries from the query history and then increase the querying keywork count by 1.

The storage layer could be NoSQL because the storage model does not need relational information and the query can be tolerated for **eventually consistency. Availability weighs more in this scenario.** Because we always expect the search server to respond to the user query and if the suggestion is not the up-to-date order, it will be acceptable.

## System Architechture

You should clarify if the search service is for an active online search application like Twitter or an offline search like Google Search.

The former one may require you to update the trie immediately, the latter one may tolerate the result is not up to date.

![system high level architechture](../img/tech/system%20design/complete%20system/architecture.jpg)

The update can not be simply done with incrby, because you must maintain the trie tree as well. It should not update the words but should also update the prefix. It would be too time-consuming if you wanted to do this in real time.

# Dive into Deep

## [Trie Tree](https://leetcode.com/problems/implement-trie-prefix-tree/)

For the **auto complete** problem, it is easy to recall that Trie tree fit the scenario well. In a view, the data structure would be like:

```Java
class Trie{
    // assume the qeury is converted to lower-case word,
    // there are 26 characters at most
    Trie[26] children;
    // false : the node is not the end of any word
    // true : the node is the end of some words
    bool isEnd;
    // only valid when isEnd == true, record the frequency of a keyword
    int frequency;
}
```

A picture from [Alex Xu's book](https://www.amazon.sg/System-Design-Interview-insiders-guide/dp/B08CMF2CQF/ref=asc_df_B08CMF2CQF/?tag=googleshoppin-22&linkCode=df0&hvadid=404658063268&hvpos=&hvnetw=g&hvrand=1585535126343690537&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9062543&hvtargid=pla-934212337151&psc=1):

![Trie Tree example](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=NWY1YjUwNWNiOGVhODY0YmU5ZDY3OGFhMWU5NzZiZWJfeUx4TmFTUE5sNEhVNGdNWUlUUmtBZXJDeWQ1RGhHZXNfVG9rZW46Ym94Y25nVkU5cnhDTDk0UTk0aG4zTXNackxlXzE2NzA3NjQxMDU6MTY3MDc2NzcwNV9WNA)

Query Process: Query the word is to query the trie tree. If we find prefix in the trie tree, then we have to find all the children under the node and sort them within a heap to generate the top-5 results.

The total time consists of three parts: O(L) + O(N) + O(KlogK)

- The first part is raised by the length of type-in keyword

- The second part is all the completions in the Trie leafnodes

- The third part is sorting out raw results

However, the process is really time-consuming, assume that the user only enters 's' and you have to traverse all the leaf nodes starting with 's'. 

Quick Calculation:

The time complexity is **O(N)** for querying the suggestion where N is the total number of keywords (completions) in the Trie. 

This is really a bottleneck of the system which increases the responding time.

There are some optimization we can do to remove the above time bottleneck.

### Remove O(N)

One possible way to improve the query process is to cache the **top-5** result for a node. In Google Search scenario, the top-5 result can read stale data for the reason that the suggestion can be a little bit outdated. While in the Twitter scenario, the top-5 results should be as accurate as possible.

![cache the completion for trie nodes](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=NmVjMzViZmNlNTNiMjlmZTk0ZDYxZTQ2ODM4MDQ5MjBfY3pMWEFpRjVxNmIzaDNvdTJsZVZKbGJZU0draW1oYlFfVG9rZW46Ym94Y25wTzZhUURUTWxpNm1xSUJzOU5WOGRnXzE2NzA3NjQxMDU6MTY3MDc2NzcwNV9WNA)

Trade-offs

For the keyword 'bee', we have to store it multiple times in the node ('b', 'be', 'bee'). **[But since speed of reads was our priority, we were happy to make these trade offs](https://medium.com/@prefixyteam/how-we-built-prefixy-a-scalable-prefix-search-service-for-powering-autocomplete-c20f98e2eff1)****.**

### Remove O(L)

If we can limit the length of keywords, like we define the max length of L as 50 chars. For searching more than 50 chars, we can simple block the auto complete service. 

This is what Google Search does. You can try to paste different things into the search bar. And you may find that if you type in a sentence which is too long, there won't be any suggestions.

### Remove O(KlogK)

To most of the auto- complete suggestions, it only requires us to generate the top5 or top10 related results. So the K can be regarded as a constant and O(KlogK) is nothing more than O(1).

## How to Store Trie Tree in the DB

### Stored in Document DB

From [Alex Xu's book](https://www.amazon.sg/System-Design-Interview-insiders-guide/dp/B08CMF2CQF/ref=asc_df_B08CMF2CQF/?tag=googleshoppin-22&linkCode=df0&hvadid=404658063268&hvpos=&hvnetw=g&hvrand=1585535126343690537&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9062543&hvtargid=pla-934212337151&psc=1), the idea is to take a snapshot of the DB weekly and store the data into Document NoSQL like MongoDB.

### Stored in KV Store

| **Key** | **Value**                   |
| ------- | --------------------------- |
| c       | [car: 30, cat:20, curry:10] |
| ca      | [car:30, cat:20, canada:10] |

This gives us a few advantages, such that we can access any prefix in O(1) time. This eliminates the tree traverse process in the memory. But it also increases the storage space because it loses the conventional trie's ability to share common prefixes. A hashmap also requires more space to store the meta data such as load factor and map size.

**[But again, we were more than willing to trade space for speed of reads.](https://medium.com/@prefixyteam/how-we-built-prefixy-a-scalable-prefix-search-service-for-powering-autocomplete-c20f98e2eff1)**

![tradeoffs choosing the data structure](https://lo845xqmx7.feishu.cn/space/api/box/stream/download/asynccode/?code=NDhmNTYxYTVmM2RlODEzMzM5ZTY0N2JkNzFmZTU3YmZfUGFPOVlFQ2pMa0hKdzJSdld5WGpVTGRYRnF0NktXVWtfVG9rZW46Ym94Y25rMFlYdk51SDJ6YXR5UHBNYXpvVXloXzE2NzA3NjQxMDU6MTY3MDc2NzcwNV9WNA)

We can use **Redis** **/ Abase** to store the trie data. To store the completion data, we could use Redis list or **Redis Zset**.

|                      | Redis List                                                   | Redis Zset                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Performance          | Search for O(1)Update for O(K)                               | Search and Update for O(logK)                                |
| Operation Complexity | Need to take care of sorting and de-duping. When updating, it requires us to take two round trips with Redis, load all the lists and increase and insert the new entry in-place. | Not require to take care the sorting and deduping logic.  The ZSET will maintain the order and uniqueness through skiplists. |
| Commands             | LRANGE: Load all the list LREM: Remove the updated entryLINSERT: Insert new entry into the list | ZRANGE: for search  ZINCRYBY: for update                     |

### Cache by Redis and Persist By MongoDB

MongoDB is a key document NoSQL which fits well with the Redis KV model. Having similar models of abstraction among data stores is convenient, because it makes the persistence process easier to reason about.

For the cache miss handling, we can implement Cache Aside Pattern. Note that the logic for searching in the event of cache miss only happens quite a few times, and when there's cache miss it can quickly load the data into Redis so that the next query can hit the cache. Thus, the slower and more complicated logic for **searching persistent stores** only happens in a relatively small number of cases.

## How to update the Trie Tree

The query service can store the query history and periodically generates the analysis log based on the historical data. With this analysis log, we can build the Trie once and use it for a long time.

![offline update trie by OLAP](../img/tech/system%20design/complete%20system/update_trie.jpg)

But note that this mechanism is only feasible for the Google Search scenario where the auto complete suggestion does not require realtime update. For the Twitter auto complete scenario, you may want to figure out some faster way to update the Trie store.

## How to partition the data efficiently

The words starting with 's' are more than the words starting with 'x' and this might cause sharding inbalance. To mitigate the problem of unevenly distributed characters.  We may introduce a shard manager to do the routing.

![partition mechanism](../img/tech/system%20design/complete%20system/sharding.jpg)

# Wrap Up

Some questions to think about:

- How do you extend your design to support multi-language?

> Hint: Unicode

- What if we want to do auto-complete based on different countries?

> [Multi tenant solution](https://medium.com/@prefixyteam/how-we-built-prefixy-a-scalable-prefix-search-service-for-powering-autocomplete-c20f98e2eff1)

- How can we support real-time search queries? 

> Update immediately

# Reference

[Alex Xu' System Design Book](https://www.amazon.sg/System-Design-Interview-Insiders-Guide/dp/1736049119/ref=asc_df_1736049119/?tag=googleshoppin-22&linkCode=df0&hvadid=405606626615&hvpos=&hvnetw=g&hvrand=5683234730197475962&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9062543&hvtargid=pla-1645933631661&psc=1)

[FaceBook: The life of a typehead query](https://www.facebook.com/notes/10158791367817200/)

[Medium: Auto completion system design](https://medium.com/@prefixyteam/how-we-built-prefixy-a-scalable-prefix-search-service-for-powering-autocomplete-c20f98e2eff1)