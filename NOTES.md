# Notes - The Little Redis Book by Karl Seguin

Github repo of this book:

```
https://github.com/karlseguin/the-little-redis-book
```

# 1. Chapter 1 The Basics

## 1.1 Databases

Redis has a concept of a databse. One use case for this, will be to seperate group of data between applications. The default databse is '0', we can also manually select the '1' database as follows:

```
127.0.0.1:6379> select 1
OK
127.0.0.1:6379[1]> select 0
OK
127.0.0.1:6379>

```

As we can see, `127.0.0.1:6379[1]>` means that we have switched to '1' database.

## 1.2 Commands, Keys & Values

A value is associated with a key, a key can be anything serialised as a string. Redis treats value as a **byte array**, so a value is nevery evaluated by Redis.

We set a value to a key:

```
127.0.0.1:6379> set curtis '{ "name" : "curtis", "state" : "reading book" }'
OK
```

We then can retrieve this value by the same key:

```
127.0.0.1:6379> get curtis
"{ \"name\" : \"curtis\", \"state\" : \"reading book\" }"
```

## 1.3 Memory & Persistence

### 1.3.1 Snapshoting

"... Redis is an in-memory persistent store. ... by default, Redis snapshots the database to disk based on how many keys have changed. You configure it so that if X number of keys change, then save the database every Y seconds."

### 1.3.2 Append Mode

Redis can backup (or persist) files using append-only mode. Whenever a key is changed, a append-only file is updated.

# Chapter 2 The Data Structures

Redis support five data structures.

## 2.1 Strings

The key is a plain string. We use `set` and `get` command for this data structure. We may also do some special operations, for example, `strlen <key>` for geting the length of value (not key).

### 2.1.1 strlen

```
127.0.0.1:6379> strlen curtis
(integer) 47
```

### 2.1.2 append

Or we can use `append <key> <value>` that appends value to existing value, if the key doesn't exists, then the value for this key will be the appended value.

```
127.0.0.1:6379> get curtis
(nil)
127.0.0.1:6379> append curtis 'newbie'
(integer) 6
127.0.0.1:6379> get curtis
"newbie"
127.0.0.1:6379> append curtis ' reading books'
(integer) 20
127.0.0.1:6379> get curtis
"newbie reading books"
```

### 2.1.3 incr, decr, incrby, decrby

Additionally, even though Redis doesn't evaluate values, if the value is in case an integer, we may use commands to increment or decrement it using `incr`, `decr`, `incrby` and `decrby`.

```
127.0.0.1:6379> incr cnt
(integer) 1
127.0.0.1:6379> decr cnt
(integer) 0
127.0.0.1:6379> incrby cnt 5
(integer) 5
127.0.0.1:6379> decrby cnt 5
(integer) 0
```

## 2.2 Hashes

Redis hashes are essentially maps between string fields and values. The basic "get"/"set" commands for hashes are `hget` and `hset`. First of all, there is only one key (as a string) for this 'Hashes' (Map) data structure. Then there are paris of key-value (field-value) inside the Hashes data structure.

### 2.1.1 hset

Set a single fields to a Hash.

```
127.0.0.1:6379> hset curtis name yongj
(integer) 1
```

### 2.1.2 hmset

Set multiple fields to a Hash. In this example, we are setting 'yongj' for field 'name', and '0' for field 'yongj'.

```
127.0.0.1:6379> hmset curtis name yongj money 0
OK
```

Then this can be visualised as below:

```
curtis ->

{
    { "name" : "yongj" }
    { "money" : "0" }
}
```

### 2.1.3 hgetall

We can get all the fields and values using `hgetall`:

```
127.0.0.1:6379> hgetall curtis
1) "name"
2) "yongj"
3) "money"
4) "0"
```

### 2.1.4 hkeys

Get all fields' name in Hash using `hkeys`.

```
127.0.0.1:6379> hkeys curtis
1) "name"
2) "money"
```

### 2.1.5 hget

We can also get a specfic field in Hash using `hget`. In this example, we are getting a single field 'name' from a hash identified by a key 'curtis'.

```
127.0.0.1:6379> hget curtis name
"yongj"
```

### 2.1.6 hmget

To get multiple fields, we use `hmget`. In this example, we are getting fields 'name' and 'money' from a Hash identified by a key 'curtis'.

```
127.0.0.1:6379> hmget curtis name money
1) "yongj"
2) "0"
```

### 2.1.7 hdel

To delete a field of a Hash, we use `hdel`. In this example, we deleted the field named 'money'.

```
127.0.0.1:6379> hdel curtis money
(integer) 1
127.0.0.1:6379> hgetall curtis
1) "name"
2) "yongj"
```

## 2.3 Lists

Lists are indexed data structure, we can retrieve a value by index, or we can get or set the first or last value. We can also trim the list (sublist).

### 2.3.1 lpush

To push one or more value to the list from the left. The list is identified by key 'names', 'curtis' is a value.

```
127.0.0.1:6379> lpush names curtis
(integer) 1
127.0.0.1:6379> lrange names 0 -1
1) "curtis"
```

### 2.3.2 rpush

To Push one or more value to the list from the right. The list is identified by key 'names', 'curtis', 'newbie' are values.

```
127.0.0.1:6379> rpush names curtis newbie
(integer) 2
127.0.0.1:6379> lrange names 0 -1
1) "curtis"
2) "newbie"
```

### 2.3.3 lpop

To pop a value from the left.

```
127.0.0.1:6379> lpop names
"curtis"
```

### 2.3.4 rpop

To pop a value from the right.

```
127.0.0.1:6379> rpop names
"newbie"
```

### 2.3.5 lset

To set a value at the specific position. Here, we set the value 'newbie' at 0, which then overwrites the previous value.

```
127.0.0.1:6379> lrange names 0 -1
1) "curtis"
2) "newbie"
127.0.0.1:6379> lset names 0 newbie
OK
127.0.0.1:6379> lrange names 0 -1
1) "newbie"
2) "newbie"
```

### 2.3.6 lindex

To get an item by index. The list is identified by key 'names', and we are retrieving the value at 0.

```
127.0.0.1:6379> lindex names 0
"curtis"
```

Finally, we can flush the database as below:

```
127.0.0.1:6379> flushdb
OK
```

### 2.3.7 lrange

To get a sublist, we specifiy a range that we want to retrieve, -1 means the last item. In the example below, we retrieve all items in the list.

```
127.0.0.1:6379> lrange names 0 -1
1) "curtis"
2) "newbie"
```

## 2.4 Sets

Sets are used to store unique values. There are a number of set-based operations we can do, e.g., set unions. Sets are unordered.

### 2.4.1 sadd

To add a value to the set.

```
127.0.0.1:6379> sadd friends curtis sharon
(integer) 2
```

### 2.4.2 smembers

To view all the members in the set.

```
127.0.0.1:6379> smembers friends
1) "curtis"
2) "sharon"
```

### 2.4.3 sismember

To check if the given string is a member in the set.

```
127.0.0.1:6379> sismember friends curtis
(integer) 1
```

### 2.4.4 sinter

To intersect multiple sets, the returned values are the intersection between them.

```
127.0.0.1:6379> sadd n1 1 2 3
(integer) 3
127.0.0.1:6379> sadd n2 2 3 4
(integer) 3
127.0.0.1:6379> sinter n1 n2
1) "2"
2) "3"
```

### 2.4.5 sunion

To get unoin between sets.

```
127.0.0.1:6379> sadd n1 1 2 3
(integer) 3
127.0.0.1:6379> sadd n2 2 3 4
(integer) 3
127.0.0.1:6379> sunion n1 n2
1) "1"
2) "2"
3) "3"
4) "4"
```

## 2.5 Sorted Sets

The sets that are sorted based on an assigned score, and these scores must be explicitly given. Values are ordered, so we can get the max, min, values within range.

### 2.5.1 zadd

To add entries, we use `zadd`. The key of this sorted set is 'scores'. 85 is a score for 'curtis', and 83 is a score for 'sharon'.

```
127.0.0.1:6379> zadd scores 85 curtis 83 sharon
(integer) 2
```

### 2.5.2 zcount

To count number of elements between specified range.

```
127.0.0.1:6379> zcount scores 80 84
(integer) 1
```

### 2.5.3 zrank

To get the rank (0-based) of a element in ascending order.

```
127.0.0.1:6379> zadd scores 85 curtis 83 sharon

...

127.0.0.1:6379> zrank scores curtis
(integer) 1
```

### 2.5.4 zrevrank

To get the rank (0-based) of a element in descending order.

```
127.0.0.1:6379> zadd scores 85 curtis 83 sharon

...

127.0.0.1:6379> zrevrank scores curtis
(integer) 0
```

# Chapter 3 Leveraging Data Structures

## 3.1 Pseudo Multi Key Queries

There are situations where we want to query the same value by different keys. A naive approach will be to have two keys used (set keyA {....} and then set KeyB {....}), but this wastes memory and is hard to manage. A better way to manage this will be to have a key for the value, then use a hash for a group of fields pointing to the real key (for the actual value).

For example:

First, having a key for the actual value

```
set user:9001 '{... actual value ..}'
```

Then we have:

```
user:9001 -> '{... actual value ...}'
```

Second, having a hash for these keys

```
hset user:lookup:email curtis@abc.com 9001 sharon@abc.com 9021
```

Then we have:

```
user:lookup:email -> {
    'curtis@abc.com' -> 9001
    'sharon@abc.com' -> 9021
}
```

3. in case that we want to use email to find the value, we can then use hash to first retrieve the id, and then use the id to retrieve the actual value we want.

## 3.2 References and Indexes

In Redis, we will have to mange references and indexes manually.

## 3.3 Round Trips and Pipelining

Making frequent round trips is a common pattern in Redis. Pipelining (send multiple requests without waiting for responses) is also supported in Redis, thus making the request and response much more efficient.

## 3.4 Transactions

Every Redis command is atomic, when a query uses multiple commands, these commands are executed in a single transaction. Redis is single-threaded, which is why every command is atomic. To issue multiple commands as a single group (as well as a single transaction), we use `multi` command, followed by commands that are executed as part of a transaction.

E.g.,

```
multi
hincrby groups:1percent balance -9000000000
hincrby groups:99percent balance 9000000000
exec
```

But we can't use it in code like below, which is how RDBMS works with Transaction Management.

```
redis.multi()
current = redis.get('powerlevel')
redis.set('powerlevel', current + 1)
redis.exec()
```

An alternative is add a `watch` to a key, when the value of this key is changed after we added our watch, our transaction will fail, then we can write a loop, continues until it works.

E.g.,

```
while true:

redis.watch('powerlevel')
current = redis.get('powerlevel')
redis.multi()
redis.set('powerlevel', current + 1)
redis.exec()
return;
```

## 3.5 Keys Anti-Pattern

Never use `keys` in production, because it does a linear scan for all the keys (if there are N keys, it takes O(N) time).

# Chapter 4 Beyond The Data Structures

## 4.1 Expiration
