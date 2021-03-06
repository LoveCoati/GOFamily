# redis的几种基本的类型
## string
redis的string是动态数组，也就是说它字符串占的长度，要小于它底层数组的容量，这一点跟go里面的slice的扩容还是有点像的，它也有类似的原理，它扩容的时候会判断字符串的长度，当长度小于1m的时候每次扩容都是容量翻倍，当大于1m的时候，每次扩容增加一个1m的空间，但是有大小限制，redis中的字符串最大值是512m

redis中采用这种冗余的预处理机制来扩容主要是为了防止频繁的内存申请，内存的分配是很浪费时间的。

**基础操作**

```shell
// 设置一个string // 可以同时两次set一个string，就是把它直接颠覆了，
set googege redis
// 得到一个string
get googege
// 获取字符串的长度
strlen googege
// 对string做切片,getrange string fitst-index end-index
// 并且必须提供后面的两个起始和结束的index，否则报错。
getrange googege 1 2
// 覆盖字符串 setrange string first-index new string.
setrange googege 1 ddddddd
// 追加字符串 append string xxx
append googege ttt

```
从以上操作发现了一个问题，并没有提供字符串的 **子串** 插入（提供的只有覆盖）和删除的操作，意思就是比如在3 5之间插入一个子串，或者是删除5-7之间的子串。

**计数器**

当字符串是整数的时候，可以将它当成计数器

```shell
// 设置一个计数器 set key int
set aa 1
// 得到一个计数器 get key
get aa
// 增加一个固定的值 incrby key number
incrby aa 100
// 减去一个固定的值 decrby key number
decrby aa 100
// 每次增加1 incr key
incr aa
// 每次减去1 decr key
decr aa
```
注意计数器有返回，它的区间是 `[long.MIN,long.Max]`全闭区间
> [-9223372036854775807,9223372036854775807]

错误提示是` ERR increment or decrement would overflow`

**字符串的删除和自动删除(过期)**
```shell
//删除字符串 del key
del aa
// 设置过期时间(单位是s) expire key time
expire aa 60
// 查看寿命 ttl key
ttl aa
```
## list
list的底层是一个双向链表，所以可以使用这个链表来实现队列或者stack的功能
```shell
// rpush rpop lpush lpop 分别是 右边推入 右边推出 左边推入 左边推出

// 实现栈的功能
rpush aaa go  java tt
rpop aaa
rpop aaa
rpop aaa
//or
lpush aaa go java tt
rpop aaa
rpop aaa
rpop aaa
// 实现队列的功能
rpush aaa go java tt
lpop aaa
lpop aaa
lpop aaa
//or
lpush aaa go java tt
rpop aaa
rpop aaa
rpop aaa
// 获取链表的长度
//llen key
llen aaa
```
我们可以获取list的子链

```shell
rpush aaa go rust dart2
lindex aaa 1 // 获取aaa的第2个元素
lrang aaa 0 2 // 获取子链
lrang aaa 0 -1 // -的意思就是倒着数
// 遍历全部的数据
lrang 0 -1 // 这个时候使用-数不用使用llen了，也是极好的。
```
**修改和插入元素**
```shell
rpush aa go java rust
// 修改
lset aa 1 t // 变成了  go t rust

// 插入数据
linsert aa before go tt// tt go java rust
linsert aa after go tt // go tt java rust
```
这里要注意，插入数据不是根据的下标的顺序，因为redis经常用在分布式的环境中，那么分布式中的下标就没有意义了，所以特别的指定在某个元素后面或者前面插入xx元素。
**删除元素**
```shell
rpush aa go rust dd
lrem aa 1 go // 删除的时候不仅仅要写出来个数还要写出来元素的值
```
**固定长度的列表**
```shell
rpush aa 1 2 3 4 5 6 7 8 9 10 // 往队列里添加数据
ltrim 0 8 // 标注起始和结束即可。
lrange aa 0 -1
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
6) "6"
7) "7"
8) "8"
// 小技巧
ltrim -1 0 // 因为end比start小，那么这个list就会被删除光。
```
### 探究list的链表的真实底层
![p](./1.1.png)

当数据很少的时候只是用ziplist，当数据起来了才会采用quicklist，Ziplist 是由一系列特殊编码的内存块构成的列表
### ziplist
[可以参考这里](http://zhangtielei.com/posts/blog-redis-ziplist.html) 这个要以后填坑
## hash
跟一般的hash table没有区别，使用哈希函数 + 数组 + 链表

哈希函数算出值，然后加入到那个值对应的数组，然后数组中是一个链表，链表表示都是算出来的这个的值的kv结构（这个时候如果链表太长就意味着是hash碰撞了，这也是攻击手段的一种。）

**增加查询删除**
```shell
//hset mapname key value 增加一个
hset ui a b
// hmset mapnae key value key value 增加一堆
hmset ui a b c d e f g h
// 获取一个key hget mapname key
hget ui a
//获取一堆key value hmget mapname key key key
hmget ui a c e h
// 获取全部k-v hgetall mapname
hgetall ui
//获取全部key
hkeys mapname
// 获取全部value
hvals mapname
// 删除元素
// hdel mapname key key key
// 判断元素是否存在
// hexists mapname key
hexists ui a // 0表示不存在 1 表示存在
```
**计数器hash**

每一个k-v都是一个独立的计数器

```shell

hincrby mapname key 1 // 没有 hincr那种每次+1的命令。

hset u a 1
hincrby u a 4
// 5
// 假如不是整数调用了hincrby就会报错。
```

### map的扩容
当hash碰撞太多的时候（时间复杂度已经从o(1)变成了n））这个时候就该扩容了，
扩容的原则就是直接将数组扩大一倍，并且将各种数据从老的数组中转移到心的数组中
同时为了避免数据迁移带来的巨大损耗，redis是新旧同时保留，然后在后台使用一个定时的任务，以及hash读写指令，将数据逐步转移到新的数据结构中，新旧同时保留如何查找数据呢，其实只要在两个数组中都查查就行了。
### map的缩容
跟扩容原理一样，也是直接缩写一倍，然后逐步使用定时任务将数据逐步转移。
## set
集合

![p](./1.2.png)
其实你可以把set想像成一个多个key一个value的数据结构，但是这个value又是隐藏的。

举个例子 比如这个set我们要增加数据

**增加元素**

```shell
// 增加元素sadd setname value value value value
sadd y 1 2 3 4 5
// 读取元素 smembers 读取全部元素。 scard 获取set的长度 srandmember获取随机个数的元素，默认是1
smembers y
scard y
srandmember y 2
```

**删除元素**

```shell
// 删除元素 srem zetname 1 2 3 4 删除必须指出来删除谁
srem q 1 2 3 4
// 随机删除一个元素 spop zetname
spop q

```
**判断元素是否存在**

```shell
// 判断是否存在 sismember setname value
sismember q 43
```
## zset
有序集合

内部数据结构 map 跳表

hash是关联 元素key和权重（key和权重一一对应，其实相当于权重就是它的value值），跳表在于排序value 根据score范围查找元素

**增加删除数据**
```shell
zadd u 4.0 go // zaddd zsetname 权重 value 权重 value

zadd u 4.0 go 1.0 java // 后面跟着的是权重。
// 获取元素个数
zcard q
// 删除元素
//zrem zsetname value
zrem u  Java

````

**计数器**

```shell
zadd q 4.0 python // zadd zsetname 权重 value
// 获取排名和分数
zscore q python
// zrank获取指定元素的正向排名
zrank q python
// zrerank 反向排名
```

**根据排名范围获取元素**
这个原理主要是因为跳表

```shell
zrange q 0 -1 // 获取所有数据
zrange q 0 -1 withscores // 获取数据和他们的权重
zrevrange q 0 -1 // 按照负的顺序去排名

// 根据sore返回获取value
zrangebysocre q 0 5// zrangebyscore zsetname 权重区间
zrangebyscore q -inf + inf withscores // -inf负无穷 +inf 正无穷

```
**根据范围移除元素**

```shell
zremrangenyrank zsetname scorestart scoreend
```

## 跳表的简单讲述

排序很定是用的跳表来实现的，

在set zset hash list中使用的链表都是ziplist（小数据）（list中大数据使用quicklist，zset使用跳表）

这里跳表就是把这些数据添加了很多的索引，并且这些索引是根据权重来添加计算的，所以说这个你可以想像成类似b+树，他们都是属于查找

//todo zset是如何能实现跳表的功能又能实现哈希table的功能

这个里面就是一边是hash 然后储存在数组中，数组中如果数量级小就是ziplist，如果大了，那么就是skiplist。
