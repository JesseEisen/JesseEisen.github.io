---
layout: post
title: Redis Learn Notes
date: 2018-01-31 05:57:00
description: redis learn
---

## Data Type

#### Redis Keys

Here are few rules about keys:

+ it's binary safe
+ very long keys are not a good idea
+ very short keys are often not a good idea
+ try to sick with a schema, For instance "username:name"
+ maximun allowed key size is 512MB

#### Redis Strings

we can use `set` and `get` to set and retrieve a string value. And set will replace any existing value already stored into the key.

With two options, we can change this act. 

+ nx: if key alread exists failed
+ xx: **only** successful when key alread exists

if strings are the basic value of redis, Here are some useful options can perform to that.

```
> set counter 100
OK
> incr counter
(integer) 101
```

And other commands like: `incrby`, `decr`, `decrby`. Here those commands are *atomic*. That means if two thread call increase once time. It always return the old value add 2.

There are a number of commands for operating on strings. For instance: `getset` set the new value and return the old value.

Here a interesting command like `mset` and `mget` for set and retrieve the value of multiple keys in a single command.

```
> mset a 10 b 20 c 30
OK
> mget a b c
1) "10"
2) "20"
3) "30"
```

As you see, mget will return an array of values.

#### Altering and querying

`Exists` command will return 0 or 1 to signal if the key is exist or not. Here some example:

```
> set greeting hello
OK
> exists greeting
(integer) 1
> del greeting
(integer) 1
> exists mykey
(integer) 0
```

`Type` command will return the kind of vlaue store at the specify key

#### key with limited time to live

We can control the time of a value living. We can use `expire` to set the time.

```
> set mykey 5
OK
> expire mykey 5
(integer) 1
> get key 
5
//wait 5s and type
> get key
(nil)
```

As convient, we can set the expire time as a paramter of `set` command.  And use `persist` to cancel the expire time, make this value persist forever. Use command `ttl` to check remaining time to live for the key.

If we want set and check expires in milliseconds, use `pexpire` and `pttl`. 

#### Lists

Redis lists are implemented via linked list. New node will insert at the head of the list or at the tail of the list.

We use `lpush` command adds a new elememt into a list, on the left(or say at head). While `rpush` command adds elements into a list, on the right(or say at the tail). And we use `lrange` to travle this list.

```
< lrange mylist 0 -1
...  //all the elements in this list
```

The two index is the start and end index of this list, both of them can be negative.

If we want to get an element from list, we can use `lpop` or `rpop`. Both of them will pop element from left and right.

#### Common use cases for lists

+ remember the latest updates posted by user into a social network
+ communication between process

#### Capped lists

`Ltrim` command is similar to `lrange`, but **instead of displaying the specified range of elements** it sets this range as the new list value.All the elements outside the given range are removed.

#### Blocking operations on lists

redist implements commands called `brpop` and `blpop` to block if the list is empty: they'll return to the caller only when a new element is added to the list, or when a user-specified timeout is reached.

```
> brpop tasks 5
1) "tasks"
2) "element"
```

The option above means: "wait for elements in the list `task`, but return if after 5 seconds no elements is available"

Use `0` as timeout to wait for elements forever.  And the return value of `brpop` is different from `rpop`, the first is the name of the key. and next is the value.

#### Automatic creation and removal of keys