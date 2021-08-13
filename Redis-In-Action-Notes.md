# Redis In Action Notes

# 1. Chapter 1 Getting Started

A lot about the five basic data structures and reasons why redis is sepecial in comparison to other database management system, e.g., MySQL, Memcached and so on. Might be better to read the official website for more up-to-date information.

## 1.1 Voting On Articles 

(p.15)

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

0. Pre-calculate score for an up-vote

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

To start a transaction, we first call **MULTI**, then redis will queue up commands from the same connection until it sees **EXEC**. This is how transaction is achieved, but since we often share Redis connection between multiple threads, this isn't reliable to some extent. 

## 3.5 Expiration

- persist
- ttl
- expire
- expireat
- pttl (milliseconds)
- pexpire (milliseconds)
- pexpireat (milliseconds)

# 4.  