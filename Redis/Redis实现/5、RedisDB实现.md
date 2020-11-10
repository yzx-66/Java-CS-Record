### RedisDB 

#### 核心实现

##### 概述

Redis 服务器将所有数据库都保存在服务器状态 redis.h/redisServer 结构的 db 数组中，db 数组的每个项都是一个 redis.h/redisDb 结构，每个redisDb结构代表一个数据库。



定义：

```c
struct redisServer {
    // ...
    
    // 一个数组，保存着服务器中的所有数据库
    redisDb *db;
    
    // 初始化服务器时，程序会根据服务器状态的dbnum属性来决定应该创建多少个数据库
    int dbnum;
    
    // ...
};
```

* dbnum 属性：由服务器配置的database选项决定，默认情况下，该选项的值为16，所以Redis服务器默认会创建16个数据库

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094745942.png#pic_center)






切换数据库

* Redis 命令

  * 默认情况下，Redis客户端的目标数据库为0号数据库，但客户端可以通过执行 SELECT 命令来切换目标数据库。

  

* 每个Redis客户端都有自己的目标数据库，每当客户端执行数据库写命令或者数据库读命令的时候，目标数据库就会成为这些命令的操作对象。

  * 在服务器内部，客户端状态redisClient结构的db属性记录了客户端当前的目标数据库，这个属性是一个指向redisDb结构的指针：

  ```c
  typedef struct redisClient {
        // ...
        
        // 记录客户端当前正在使用的数据库
        redisDb *db;
        
        // ...
    } redisClient;
  ```

    redisClient.db指针指向redisServer.db数组的其中一个元素，而被指向的元素就是客户端的目标数据库。

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094800654.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


















谨慎处理多数据库程序

* 到目前为止，Redis仍然没有可以返回客户端目标数据库的命令。虽然redis-cli客户端会在输入符旁边提示当前所使用的目标数据库：

  ```c
  redis> SELECT 1
  OK
  redis[1]> SELECT 2
  OK
  redis[2]>
  ```

* 但如果你在其他语言的客户端中执行Redis命令，并且该客户端没有像redis-cli那样一直显示目标数据库的号码，那么在数次切换数据库之后，你很可能会忘记自己当前正在使用的是哪个数据库。

* 当出现这种情况时，为了避免对数据库进进行误操作，在执行Redis命令特别是像FLUSHDB这样的危险命令之前，最好先执行一个SELECT命令，显式地切换到指定的数据库，然后才执行别的命令。





##### dict实现

Redis是一个键值对（key-value pair）数据库服务器，服务器中的每个数据库都由一个redis.h/redisDb结构表示。

redisDb结构的dict字典保存了数据库中的所有键值对，我们将这个字典称为键空间（key space）。

```c
typedef struct redisDb {
    // ...
    
    // 数据库键空间，保存着数据库中的所有键值对
    dict *dict;
    
    // ...
} redisDb;
```

键空间和用户所见的数据库是直接对应的：

* 键空间的键也就是数据库的键，每个键都是一个字符串对象。

* 键空间的值也就是数据库的值，每个值可以是字符串对象、列表对象、哈希表对象、集合对象和有序集合对象中的任意一种Redis对象。



示例

```c
redis> SET message "hello world"  
OK
    
redis> RPUSH alphabet "a" "b" "c"
(integer)3

redis> HMSET book name "Redis in Action" author "Josiah L. Carlson" publisher "Manning"
(integer) 3
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094815261.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)






##### 键操作

添加键

* 添加一个新键值对到数据库，实际上就是将一个新键值对添加到键空间字典里面，其中键为字符串对象，而值则为任意一种类型的Redis对象。

该操作不做类型检查。



删除键

* 删除数据库中的一个键，实际上就是在键空间里面删除键所对应的键值对对象。

该操作不做类型检查。



更新键

* 对一个数据库键进行更新，实际上就是对键空间里面键所对应的值对象进行更新，根据值对象的类型不同，更新的具体方法也会有所不同（会先根据 redisObject 的 Type 校验，然后再根据 encoding 选择执行的函数）。

* 示例（redis 没有更新命令，只有添加命令，所以存在就更新，不存在就创建）：

  * `SET message "blah blah"`（存在就更新）

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094829968.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  * 继续执行 `HSET book page 320`（不存在就创建）

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094845332.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




键取值

* 对一个数据库键进行取值，实际上就是在键空间中取出键所对应的值对象，根据值对象的类型不同，具体的取值方法也会有所不同（会先根据 redisObject 的 Type 校验，然后再根据 encoding 选择执行的函数）。






其他键操作

除了上面列出的添加、删除、更新、取值操作之外，还有很多针对数据库本身的Redis命令，也是通过对键空间进行处理来完成的。

* 比如说，用于清空整个数据库的FLUSHDB命令就是通过删除键空间中的所有键值对来实现的。
* 又比如说，用于随机返回数据库中某个键的RANDOMKEY命令，就是通过在键空间中随机返回一个键来实现的。

* 另外，用于返回数据库键数量的DBSIZE命令，就是通过返回键空间中包含的键值对的数量来实现的。

* 类似的命令还有EXISTS、RENAME、KEYS等，这些命令都是通过对键空间进行操作来实现的。





##### 维护操作

当使用Redis命令对数据库进行读写时，服务器不仅会对键空间执行指定的读写操作，还会执行一些额外的维护操作

读取：

* 在读取一个键之后（读操作和写操作都要对键进行读取），服务器会根据键是否存在来更新服务器的键空间命中（hit）次数或键空间不命中（miss）次数，这两个值可以在INFO stats命令的keyspace_hits属性和keyspace_misses属性中查看。

* 在读取一个键之后，服务器会更新键的LRU（最后一次使用）时间，这个值可以用于计算键的闲置时间，使用OBJECT idletime命令可以查看键key的闲置时间。

* 如果服务器在读取一个键时发现该键已经过期，那么服务器会先删除这个过期键，然后才执行余下的其他操作。



修改：

* 如果有客户端使用WATCH命令监视了某个键，那么服务器在对被监视的键进行修改之后，会将这个键标记为脏（dirty），从而让事务程序注意到这个键已经被修改过。

* 服务器每次修改一个键之后，都会对脏（dirty）键计数器的值增1，这个计数器会触发服务器的持久化以及复制操作。

* 如果服务器开启了数据库通知功能，那么在对键进行修改之后，服务器将按配置发送相应的数据库通知，本章稍后讨论数据库通知功能的实现时会详细说明这一点。





#### 过期字典

redisDb结构的expires字典保存了数据库中所有键的过期时间，我们称这个字典为过期字典。



##### expires实现

```c
typedef struct redisDb {
    // ...
    
    // 过期字典，保存着键的过期时间
    dict *expires;
    
    // ...
} redisDb;
```

expires

* 过期字典的键是一个指针，这个指针指向键空间中的某个键对象（也即是某个数据库键）。

* 过期字典的值是一个long long类型的整数，这个整数保存了键所指向的数据库键的过期时间——一个毫秒精度的UNIX时间戳。



示例：

* 为了展示方便，下图键空间和过期字典中重复出现了两次alphabet键对象和book键对象。在实际中，键空间的键和过期字典的键都指向同一个键对象，所以不会出现任何重复对象，也不会浪费任何空间

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094858395.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)






##### 设置键过期时间

过期时间是一个UNIX时间戳，当键的过期时间来临时，服务器就会自动从数据库中删除这个键。

Redis有四个不同的命令可以用于设置键的生存时间（键可以存在多久）或过期时间（键什么时候会被删除）。

* XPIRE \<key> \<ttl> 命令，用于将键 key 的生存时间设置为 ttl 秒。

* PEXPIRE \<key> \<ttl> 命令，用于将键 key 的生存时间设置为 ttl 毫秒。

* EXPIREAT \<key> \<timestamp> 命令，用于将键 key 的过期时间设置为 timestamp 所指定的 秒数时间戳。

* PEXPIREAT \<key> \<timestamp> 命令，用于将键 key 的过期时间设置为 timestamp 所指定的 毫秒数时间戳。



虽然有多种不同单位和不同形式的设置命令，但实际上EXPIRE、PEXPIRE、EXPIREAT三个命令都是使用PEXPIREAT命令来实现的：无论客户端执行的是以上四个命令中的哪一个，经过转换之后，最终的执行效果都和执行PEXPIREAT命令一样。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094911347.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




* 首先，EXPIRE 命令可以转换成 PEXPIRE 命令

  ```python
  def EXPIRE(key,ttl_in_sec):
      # 将 TTL 从秒转换成毫秒
      ttl_in_ms = sec_to_ms(ttl_in_sec)
          
      PEXPIRE(key, ttl_in_ms)
  ```

* 接着，PEXPIRE命令又可以转换成PEXPIREAT命令

  ```python
  def PEXPIRE(key,ttl_in_ms):
      # 获取以毫秒计算的当前 UNIX 时间戳
      now_ms = get_current_unix_timestamp_in_ms()
          
      #当前时间加上 TTL，得出毫秒格式的键过期时间
      PEXPIREAT(key,now_ms+ttl_in_ms)
  ```

* 并且，EXPIREAT命令也可以转换成PEXPIREAT命令：

  ```python
  def EXPIREAT(key,expire_time_in_sec):
      # 将过期时间从秒转换为毫秒
      expire_time_in_ms = sec_to_ms(expire_time_in_sec)
          
      PEXPIREAT(key, expire_time_in_ms)
  ```

  



示例：执行 `PEXPIREAT message 1391234400000`

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094928387.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)






##### 移除过期时间

PERSIST 命令可以移除一个键的过期时间：

* PERSIST 命令就是 PEXPIREAT 命令的反操作：PERSIST命令在过期字典中查找给定的键，并解除键和值（过期时间）在过期字典中的关联。



移除过程伪代码

```python
def PERSIST(key):
    # 如果键不存在，或者键没有设置过期时间，那么直接返回
    if key not in redisDb.expires:
        return0
    # 移除过期字典中给定键的键值对关联
    redisDb.expires.remove(key)
    # 键的过期时间移除成功
    return 1
```





示例：执行 `PERSIST book`

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201110094942133.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)






##### 查询剩余生存时间

TTL命令以秒为单位返回键的剩余生存时间，PTTL命令则以毫秒为单位返回键的剩余生存时间

* 都是通过计算键的过期时间和当前时间之间的差来实现的



伪代码实现

```python
def PTTL(key):
    # 键不存在于数据库
    if key not in redisDb.dict:
        return-2
            
    # 尝试取得键的过期时间
    # 如果键没有设置过期时间，那么 expire_time_in_ms 将为 None
    expire_time_in_ms = redisDb.expires.get(key)
    # 键没有设置过期时间
    if expire_time_in_ms is None:
        return -1
            
    # 获得当前时间
    now_ms = get_current_unix_timestamp_in_ms()
    # 过期时间减去当前时间，得出的差就是键的剩余生存时间
    return(expire_time_in_ms - now_ms)
            
def TTL(key):
    # 获取以毫秒为单位的剩余生存时间
    ttl_in_ms = PTTL(key)
        
    if ttl_in_ms < 0:
        # 处理返回值为-2和-1的情况
        return ttl_in_ms
    else:
        # 将毫秒转换为秒
        return ms_to_sec(ttl_in_ms)
```
