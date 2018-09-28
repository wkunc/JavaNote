# 基本操作

## SET key value

>TIME COMPLEXITY: O(1)
>
>DESCRIPTION: Set key to hold the **string** value. If key already holds a value, it is overwritten, regardless of its type. Any previous time to live associated with the key is discarded on successful SET operation.


| 操作符          | 描述                                       |
| --------------- | ------------------------------------------ |
| EX seconds      | 设置特定时间过期, 单位秒                   |
| PX milliseconds | 设置特定时间过期, 单位毫秒                 |
| NX              | Only set key if it does not already exist. |
| XX              | Only set the key if it already exist.      |
## RETURN VALUE: 
>Status code reply: OK if SET was executed correctly. 
>
>Null multi-bulk reply: a Null Bulk Reply is returned if the SET operation was not performed becase the user specified the NX or XX option but the condition was not met.

## GET key

## DEL Key, Key2, Key3...

## INCR/DECR(自增/自减)
INCR key

DECR key

return : 自增或自减之后的值,

## EXPIRE/TTL

## EXPIRE key seconds    (expire 过期)
>TIME COMPLEXITY O(1)
>
>DESCRIPTION Set a timeout on key. After the timeout has expired, the key will automatically be deleted. A key with an associated timeout is often said to be volatile(易变的) in Redis terminology(术语). 

RETURN VALUE integer reply, specifically:

>1 if the timeout was set.
>
>0 if key does not exist or the timeout could not be set.

## TTL key

>DESCRIPTION: The TTL command returns the remaining(其余的) time to live in seconds of a key that has an EXPIRE set. This introspection capability allows a Redis client to check how many seconds a given key will continue to be part of the dataset. 

RETURN VALUE: Integer reply
>If the key does not have an associated expire, -1 is returned. 
>
>If the key does not exist, -2 is returned.

# Redus 中的 List (有序列表)

Redis also supports several more complex data structures. The first one we'll look at is a list. A list is a series of ordered values.

列表中的值类型不能不同

重要的命令
| 命令   | 描述                                                                           |
| ------ | ------------------------------------------------------------------------------ |
| RPUSH  | puts the new value at the end of the list.                                     |
| LPUSH  | puts the new value at the start of the list.                                   |
| LLEN   | returns the current length of the list.                                        |
| LRANGE | 返回list 中 你指定 开始和结尾的元素(如果给定范围是 0 -1 就视为 请求 list 全部) |
| LPOP   | removes the first element from the list and returns it.                        |
| RPOP   | removes the last element from the list and returns it.                         |

# Redis 中的 Set
Redis 中的 Set 是无序的, 并且每一个 元素只允许出现一次

重要的命令
|命令|描述
|----|--
|SADD|adds the given value to the set.
|SREM|removes the given value from the set.
|SISMEMBER|tests if the given value is in the set. It returns 1 if the value is there and 0 if it is not.
|SMEMBERS|returns a list of all the members of this set.
|SUNION|combines two or more sets and returns the list of all elements.(合并两个或更多指定的 Set ,其中重复的元素在结果中也只会出现一次)

# Redis 中的 ZSet
有序的 Set 集合

# Redis 中的 Hash
