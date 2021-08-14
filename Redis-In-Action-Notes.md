# Redis In Action Notes

# 1. Chapter 1 Getting Started

A lot about the five basic data structures and reasons why redis is sepecial in comparison to other database management system, e.g., MySQL, Memcached and so on. Might be better to read the official website for more up-to-date information.

## 1.1 Voting On Articles 

Problem: verticles are voted. For each day, 1,000 articles are submitted, but only about 50 of them are rated to be in the top-100 articles for at least one day. All of those 50 articles will receive at least 200 up votes. The scores will go down over time, so the score should also be relavant to the posting time and current time. We only track rank of those articles that are posted within this week.

So, for the score:

```
score is function f` of posted_time, up_votes.

e.g.,

let 
    ds` be the score for a day
    uv` be the number of up votes for a day

then ds` = (86,400 / 200) * uv`

    where 86,400 is number of seconds in a day
    and 200 is the number votes to be the top-100 articles
    so, 86,400 / 200 is the score per up_vote
```

For each article, we use hash to store its info:

```
e.g.,
hset article:123451 title 'Go to statement considered harmful'
hset article:123451 link 'http://goo.gl/kZUSu'
hset article:123451 poster 'user:83271'
hset article:123451 time '13313821699.33'
hset article:123451 votes '528'

then we have:

    article:123451 -> {
        title -> 'Go to statement considered harmful'
        link -> 'http://goo.gl/kZUSu'
        poster -> 'user:83271'
        time -> '13313821699.33'
        votes -> '528'
    }
```

Then we need to record as well as to order these articles based on the scores, we use ZSET (score sorted set), one for time-ordered, and another for score-ordered.

```
# sorted by time 
zadd article:sort:time 13313821699.33 article:123451
zadd article:sort:time 13313821699.34 article:123452
zadd article:sort:time 13313821699.35 article:123453
zadd article:sort:time 13313821699.36 article:123454

# sorted by score
zadd article:sort:score 13311352219.33 article:123451
zadd article:sort:score 13311352219.34 article:123452
zadd article:sort:score 13311352219.35 article:123453
zadd article:sort:score 13311352219.36 article:123454
```

To prevent user from voting same article multiple times, we again use a SET to track who have voted for each article.

```
sadd article:voted:123451 user:543211 
sadd article:voted:123451 user:543212 
sadd article:voted:123451 user:543213 
sadd article:voted:123451 user:543214 
```

## 1.2 Scenarios

### 1.2.1 Upvotes An Article 

A user upvotes an article '123451' (assume these commands are executed within same transaction).

Persudo Code:

1. Pre-calculate score for an up-vote

    ```
    LET SCORE_OF_AN_UPVOTE = 86,400 / 200
    ```

1. Check if the article upvoted is posted within this week 

    ```
    LET posted_time = 'hget article:123451 time'

    IF posted_time < (NOW - ONE_WEEK_IN_SECONDS)
    THEN 
        GIVE_UP 
    FI
    ```

2. Increase the score of the article

    ```
    'zincrby article:sort:score $SCORE_OF_AN_UPVOTE article:123451'
    ```

3. Incrase the votes of the article

    ```
    'hincrby article:123451 votes 1'
    ```

4. Record who has voted it

    ```
    'sadd article:voted:123451 user:32123'
    ```

### 1.2.2 Posting 

A user wants to post a new article.

1. User post the article, and the system uses redis to generate an article id (say, 87654). 

    ```
    LET next_article_id = 'incr article:lastid'
    ```

2. Set the poster that he/she has been voted for post (this essentially means that the author can't vote article he/she posted). We also make the set expires after a week, because, we don't track this sort of information after a week.

    ```
    'sadd article:voted:87654 $CURRENT_USER'
    'expire article:voted:87654 $ONE_WEEK_IN_SECONDS'
    ```

3. Record information about the post into a set.

    ```
    'hmset article:87654 .......'
    ```

4. Record it into the sorted set for score and time. Notice that the initial score is ("SCORE_OF_AN_UPVOTE + NOW_EPOCH_TIME"), so the posts with same scores are also sorted by time desc.

    ```
    'zadd article:sort:time $NOW article:87654'

    LET INIT_SCORE  = SCORE_OF_AN_UPVOTE + NOW_EPOCH_TIME
    'zadd article:sort:score $INIT_SCORE article:87654'
    ```

### 1.2.3 Grouping Articles

Articles are sometimes grouped. To group articles, we simply provide a set for each group, and add the article id into the set.

1. Add article to group

    ```
    'sadd group:default article:123456'
    'sadd group:redis article:123457'
    ```

2. Remove article from group

    ```
    'srem group:default article:123456'
    'srem group:redis article:123457'
    ```

If users want the articles in group are sorted based on time or score, we can do intersection of these sets or zsets. Items in sets all have a score of 1.

1. Add articles to group 

    ```
    'sadd group:redis article:123456'
    'sadd group:redis article:123457'
    ```

2. Use zset for their scores

    ```
    'zadd article:sort:score 13311352219.36 article:123456'
    'zadd article:sort:score 13311352220.37 article:123457'
    ```

2. Intersect the group and the scores, and store the temporary result (as scores may change over time). 'group:sorted:redis" is result of intersection, 2 refers to the number of keys (we intersect two set/zset, thus 2 keys). We can also make it expirt after, say, 60 seconds, and create such intersection only when it's not found.

    ```
    'zinterstore group:sorted:redis 2 group:redis article:sort:score'
    ```

# 2. Chapter 2 Anatomy of A Redis Web Application

## 2.1 Login Session With Token Cookie 

### 2.1.1 Remember user with session

Simply uses a hash for each user 

1. Check if user is logged-in

    ```
    LET session = 'hgetall session:$USER_NAME' 
    IF !session
    THEN
    USER_SHOULD_LOG_IN 
    FI
    ```

2. User is logged-in, create a session for user

    ```
    RETURN 'hmset session:$USER_NAME recent NOW'
    ```

### 2.1.2 Track items viewed by user

We can create a zset for each session to record what items are viewed by users. User's name is used for the zset, and the NOW timestamp is used as a score, so that we can sort these items by time.

```
'zadd viewed:$USER_NAME NOW item'
```

When the items exceed the size limit, we can trim it by removing N earliest items. In example below, we deleted all the other items except the most recent 25 items, '-26' refers to the 26th item with highest score, i.e., we are keeping the 1th-25th items with higher scores.

```
'zremrangebyrank viewed:$USER_NAME 0 -26'
```

### 2.1.3  Clean up sessions 

Sessions can be cleaned up when it exceeds a certain limit.  We simply create a scheduled job for this. Say, we have a set that sorted set that tracks the recent sessions ('sessions'). First, use **zcard** (cardinality) to get the size of zset. when the count exceeds the limit, we fetch the oldest N tokens to deleted (tokens are ordered by timestamp in ascending order, and timestamp are stored in forms of epoch time, so the first one is the earliest one). 

```
LET COUNT = 'zcard sessions'

WHILE count < THRESHOLD 
DO
    SLEEP 1 SEC
ELSE 
    LET end = MIN(count - THRESHOLD, 100)
    LET tokens_to_be_deleted = COLLECT('zrange sessions 0 $end')
    'delete $tokens_to_be_deleted'
DONE
```

## 2.2 Shopping Carts

Use the hash for the user to store items in shopping carts, the field is the item's id, and the value is the count of the item.

## 2.3 Static Content Caching, Database Row Caching, and Web Page Analytics

Same thing, just use combination of hash, set, zset to cache. The key is to find a proper way to use these data structures, it's more about the design than Redis.

# 3. Chapter 3 Commands in Redis

Should read offcial website for up-to-date information :( 

## 3.1 Data Structures

- String 
    - Byte string values
    - Integer values
    - Floating-point values 
- List
    - also works as a queue
- Set
- Hashes
- Sorted Sets

## 3.2 Pub/Sub

- Subscribe
- Unsubscribe
- Publish
- Psubscribe
- Punsubscribe

## 3.3 Sort

Some basic explaination about Sort.

## 3.4 Transaction

Related commands:

- watch
- multi
- exec
- unwatch
- discard

To start a transaction, we first call **MULTI**, then redis will queue up commands from the same connection until it sees **EXEC**. Plus, commands in this kind of transaction doesn't execute until an **EXEC** is received (with pipelining), which means that we can't use the result returned by our commands to decide what we may do with the data within the transaction.

## 3.5 Expiration

- persist
- ttl
- expire
- expireat
- pttl (milliseconds)
- pexpire (milliseconds)
- pexpireat (milliseconds)

# 4. Chapter 4 Keeping Data Safe And Ensuring Performance

## 4.1 Persistence Options

- Snapshotting
    - takes snapshots of data at one moment in time
- AOF (Apend-Only File)
    - writing changes to disk as they happen

### 4.1.1 Configuration Examples

(Should read official doc for this part)

For snapshotting

```
save 60 1000
stop-writes-on-basave-error no
rdbcompression yes
dbfilename dum.rdb
```

For AOF

```
appendonly no
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-precentage 100
auto-aof-rewrite-min-size 64mb
```

Shared 

```
# where to store the backup files
dir ./ 
```

### 4.1.2 Using Snapshots

There five methods to initiate a snapshot:

1. **BGSAVE**
    - call **BGSAVE** command, Redis will *fork* a child process which writes snapshot at the background while the parent process continues to respond to commands.
2. **SAVE**
    - call **SAVE** command, which causes Redis to stop responding any commands until the snapshots complete, it's generally faster than **BGSAVE**.
3. **config set save ...**
    - if Redis is configured with **save**, e.g., **`save 60 10000`**, **BGSAVE** is triggered when the rules are matched, e.g., when 10,000 writes occurred within 60 secconds. E..g, if we are able to accept one hour loss of data, we might use *save 3600 1*.
4. **SHUTDOWN** command or **TERM** signal
    - when Redis receives a **SHUTDOWN** command, or **TERM** signal, it will perform a **SAVE** command, and then shutdown.
5. **SYNC**
    - if master issues a **SYNC** to begin master/slave replication, the master will start a **BGSAVE** if one isn't recently completed.

### 4.1.3 Using AOF

We can enable AOF using config **`appendonly yes`**. It's essentially replaying the AOF file to recover.

#### 4.1.3.1 File Syncing

File syncing is affected by config **`appendfsync`**. It has three options:

- always
    - every write command to redis results in a fsync
- everysec
    - explictly call fsync once per second
- no
    - let OS controls the syncing to disk

When files are written disk, three things occur (how **`appendfsync`** affects):

1. the file is written to buffer by `file.write()`.
2. when the data is in the buffer, the OS takes the data and write it to disk at some point in future, we may also call `file.flush()` to request OS to flush the cache to disk, but it's merely a request.
3. we can also asks OS to sync the file to disk, whichis a blocking operation, when it completes, we are pretty sure that the file is now on disk. This config affects how Redis may call fsync.

So, when **`appendfsync always`** is configured, we know that every write to result in a fsync, which dramatically slows down Redis performance, because it's a blocking operation. In most cases, we may just use the default option **everysec**. The option **no** is also not recommended, because it lets OS to do fsync, and when the OS fails, we will lose unknow amount of data. Further, there may be cases where the write buffer is full without enough syncing, and Redis is blocked until the buffer is flushed to disk.

Though AOF seem to nicely mitigate data loss. It has following problems:

- huge AOF file 
    - we can use **BGREWRITEAOF** to rewrite the AOF file, making it as short as possible. This command *forks* a child process to rewrite the file in the background.
    - we can also configure an automatic rewrite of AOF using **auto-aof-rewrite-percentage** and **auto-aof-rewrite-min-size** config. E.g., *`auto-aof-rewrite-percentage 100 auto-aof-rewrite-min-size 64mb`*, which means the AOF is rewritten when it's 100 percentage bigger than previous AOF size, or when it's bigger than 64mb in size.
- slow startup because of the replay

## 4.2 Replication

To create a slave node, we may simply config **`slaveof $host $port`**, Redis started with configuration like this will connect to the master node using the provided host and port. To switch to another master node, we simply issue the command again.

What happens between master and slave during replication:

1. slave issue **sync** to master
2. master starts **BGSAVE**, and keeps **backlog** of all write commands sent after **BGSAVE**.
3. master finishes **BGSAVE**, starts sending **snapshot** to slave
4. master finishes sending **snapshot**.
5. slave discard old data, starts to load the **snapshot** from master
6. master starts sending the **backlog** to slave, and slave load and parses the **backlog**, meanwhile, commands are streamed from master to slave. 

There are four important details: 

1. slave data are **all discarded** when replication starts
2. snapshot created
3. backlog (after snapshot)
4. commands streaming (after backlog)

**Master/slave Chaining**: it's basically acyclic graph like structure, where slave can be master of other slave nodes.

## 4.3 System Failures

### 4.3.1 Verifying Snapshots And AOF files

we can use cli-tools like **redis-check-aof** and **redis-check-dump** to verify the generated files. 

# 5. Chapter 5 Using Redis For Application Support

Some basic usage e.g.,

- logging
- counters
- statistics
- look-up table
- service discovery

# 6. Application Components in Redis

## 6.1 Distributed Locking

Follwing examples about locking are all assumed to be executed in a single atomic transaction, e..g, using LUA or WATCH + MULTI/EXEC.

### 6.1.1 Acquiring Lock

The basic idea for acquiring lock is as follows:

1. we create a unique identifier using UUID, we may also create one using our threa_id, so that we can unlock it later without remembering the identifier.
2. we specify for how long we want to keep trying to acquire the lock
3. we specify the timeout for our lock, in this case it's 10 seconds
4. while we are within acquire timeout, we use **`'set $lock_name $identifier EX $lock_timeout NX'`** to acquire lock if not exists. **EX** means seconds, we can use **PX** for miliseconds if necessary. **NX** means, the value is set only when it's not exists.

persudo code:

```
# generate unqiue identifier
LET identifier = UUID()

# for how long we try to acquire the lock
LET acquire_timeout = NOW() + 1_SEC

# expiry time of the lock
LET lock_timeout = 10_SEC

WHILE NOW() < acquire_timeout
DO
    IF 'set $lock_name $identifier EX $lock_timeout NX'
    THEN 
        RETURN HAS_LOCK 
DONE

RETURN NO_LOCK
```

### 6.1.2 Releasing Lock

For this part, we either need to use LUA or WATCH to make sure the operation is atomic and exclusive.

The basic idea for releasing lock is as follows:

1. we watch the key, to make sure the key is not modified while we are using it, if it's modified, we simply give-up 
2. once we found that the lock's value equals our identifier, we remove it

persudo code:

```
WHILE TRUE
DO
    'watch $lock_name'
    IF 'get $lock_name' == identifier
    THEN 
        'del $lock_name'
        RETURN
    FINALLY 
        'unwatch $lock_name'

    CATCH watch_error
    THEN
        RETURN
DONE
```

## 6.2 Semaphore

For the following persudo code, we either need to use LUA or WATCH to make sure the operation is atomic and exclusive.
Semaphore may encounter race condition, for strict use of semaphore, we need extra lock, though it's not used in following examples.

### 6.2.1 Acquire Semaphore

The general idea is as follows:

1. we create an identifier using UUID, this is also used to return semaphore once we have done
2. we record now in forms of Epoch time, this is the *score* that we will use in the zset
3. the timeout for each semaphore is set to 10 seconds, so the entries with time/score that is less than now-10 are all expired, which is why we use **zremrangebyscore** to remove any entries that has value less than *now-timeout*.
4. we use **zcard** to check if there is still any semaphore we can acquire, if so, use **zadd** to add our identifer into it, where the *score* will be our time. Notice that it's a zset, and entries are ordered by time (Epoch time) in ascending order, so the oldest entries are always removed from the head.

persudo code:

```
LET identifier = UUID()
LET now = NOW()
LET timeout = 10_SEC
 
# remove old entries 
LET END = now - timeout
'zremrangebyscore $sem_name $INTEGER_MIN $END'

# try to acquire semaphore
IF 'zcard $sem_name' < threshold 
THEN
    'zadd $sem_name $now $identifier'
FI
```

### 6.2.2 Release Semaphore

Releasing the sempahore is much easier. We just remove the entry from zset using the same identifier.

persudo code:

```
'zrem $sem_name $identifier'
```

### 6.2.3 Refresh Semaphore

Refreshing semaphore is also very straightforward, we try to replace our entry with a new time (score), if we succeed (the entry is still in the zset), then we stll have the semaphore.

persudo code:

```
IF 'zadd $sem_name $identifier $now'
THEN
    RETURN TRUE
ELSE
    RETURN FALSE 
FI
```

## 6.3 Task Queues & Pull Messaging

Use list to achieve. For Pull Messaging, pub/sub doesn't do much.

# 7. Chapter 7 Search-Based Applications

About how to use set, zset and their operations like union, intersection for indexing and searching.

# 8 - 11 Chapter 8-11 

Skipped. More about use cases, some optimisation things like reduing memory usage stuff. Lua is another language to learn, once it's understood, we can then use it for scripting naturally without much learning curve.
