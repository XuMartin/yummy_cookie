Redis问题集锦

**1、redis 的 ZSET 是怎么实现的**

1. zset是redis的有序集合对象，编码是ziplist或者skiplist，使用命令`object encoding`可以查看编码方式。

2. ziplist的底层实现是压缩列表，每个集合元素使用两个紧邻的压缩列表节点表示，第一个节点存储对象，第二节点存储分值，集合元素按照分值从小到大排列；

3. skiplist的底层实现是zset，由跳跃表和字典组成。跳跃表中存储按照每个集合元素的分值从大到小存储，每个节点存储了一个元素，节点的object存储了元素的对象，score存储了元素的分数。字典存储了所有的集合元素，字典项的key是元素对象，字典项的value是score。

   ```C++
   typedef struct zset {
   	zskiplist *zsl;
   	dict *dict;
   } zset;
   ```

4. 使用跳跃表可以使查找的时间复杂度降低到O(logn),字典表可以是查找对象对应分数的复杂度降低到O(1)。跳跃表会和字典会共享集合中的对象和分值，因此不会造成内存浪费。

5. 编码的转换，当有序集合的数量少于128个且每个元素成员的字节数小于64字节时，使用ziplist编码，否则选择skiplist编码。（可通过zset-max-ziplist-entries和zset-max-ziplist-value选项调整参数）

6. | 命令     | ziplist                                            | zset                                                     |
   | -------- | -------------------------------------------------- | -------------------------------------------------------- |
   | ZADD     | ziplistInsert                                      | zslInsert && dictAdd                                     |
   | ZCARD    | ziplistLen / 2                                     | length                                                   |
   | ZCOUNT   | 遍历统计分值在给定范围内节点数量                   | 遍历统计分值在给定范围内节点数量                         |
   | ZRANGE   | 正向遍历返回索引范围内所有元素                     | 正向遍历返回索引范围内所有元素                           |
   | ZREVRANG | 反向遍历返回索引范围内所有元素                     | 反向遍历返回索引范围内所有元素                           |
   | ZRANK    | 正向遍历返回查找给定成员所途经的节点数量（排名）   | 正向遍历返回查找给定成员所途经的节点数量（排名）         |
   | ZREVRANK | 反向遍历返回查找给定成员所途经的节点数量（排名）   | 反向遍历返回查找给定成员所途经的节点数量（排名）         |
   | ZREM     | 遍历压缩列表删除包含给定成员的节点和对应分值的节点 | 遍历跳跃表删除包含指定成员的跳跃表节点并在字典中删除分值 |
   | ZSCORE   | 遍历压缩列表找到指定成员对应的分值节点的分数       | 从字典表中取出给定成员对应的分数                         |

   

**2、Redis 的 ZSET 做排行榜时，如果要实现分数相同时按时间顺序排序怎么实现？**

1. 分值double是8个字节，前32位存储分数，后32位存储时间；

2. 将时间毫秒数转化为小数。

   ```java
   long updateTime = System.currentTimeMillis();
   double timeRank = points + 1 - updateTime / Math.pow(10, (int) Math.log10(updateTime) + 1);
   ```

   

