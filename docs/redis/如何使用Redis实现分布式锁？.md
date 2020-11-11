# 如何使用Redis实现分布式锁？
## setnx+expire实现
```
public boolean tryLock(String key,String requset,int timeout) {
    Long result = jedis.setnx(key, requset);
    // result = 1时，设置成功，否则设置失败
    if (result == 1L) {
        return jedis.expire(key, timeout) == 1L;
    } else {
        return false;
    }
}
```
这是最简单的实现方式，也是出现频率很高的一个版本，通过Redis的SETNX命令将key的值设置为value，当键不存在的时候，设置成功（返回1，获取锁成功）；若键存在，则什么也不做（返回0，获取锁失败）。同时为了避免服务重启或异常导致锁无法释放，通过expire命令给锁设置了超时时间。但是该方案的主要问题在于：

    setnx和expire是分开的两步操作，不具有原子性。若执行完第一条指令应用异常或者重启了，锁将无法过期。
    
## Lua脚本实现
```
public boolean tryLock_with_lua(String key, String UniqueId, int seconds) {
    String lua_scripts = "if redis.call('setnx',KEYS[1],ARGV[1]) == 1 then" + "redis.call('expire',KEYS[1],ARGV[2]) return 1 else return 0 end";
    List<String> keys = new ArrayList<>();
    List<String> values = new ArrayList<>();
    keys.add(key);
    values.add(UniqueId);
    values.add(String.valueOf(seconds));
    Object result = jedis.eval(lua_scripts, keys, values);
    //判断是否成功
    return result.equals(1L);
}
```
这种方案主要是解决setnx和expire不是原子操作的问题。
## set key value [EX seconds][PX milliseconds][NX|XX]命令实现
```
public boolean tryLock_with_set(String key, String UniqueId, int seconds) {
    return "OK".equals(jedis.set(key, UniqueId, "NX", "EX", seconds));
}
```
这种方案主要是解决setnx和expire不是原子操作的问题。除此之外，还需要保证value具有唯一性，可以用UUID来做，设置随机字符串保证唯一性，至于为什么要保证唯一性？保证唯一性的目的在释放锁的时候会提到。

## 释放锁的实现
释放锁时需要验证value值，即上述在获取锁的时候需要设置一个value，不能直接用del key这种粗暴的方式，因为直接del key任何客户端都可以进行解锁了，所以解锁时，需要判断锁是否是自己的，基于value值来判断，代码如下：
```
public boolean releaseLock_with_lua(String key,String value) {
    String luaScript = "if redis.call('get',KEYS[1]) == ARGV[1] then " + "return redis.call('del',KEYS[1]) else return 0 end";
    return jedis.eval(luaScript, Collections.singletonList(key), Collections.singletonList(value)).equals(1L);
}
```
## Redis命令实现分布式锁的问题
### 锁误解除
如果线程A成功获取到了锁，并且设置了过期时间30秒，但线程A执行时间超过了30秒，锁过期自动释放，此时线程B获取到了锁；随后A执行完成，线程A使用DEL命令来释放锁，但此时线程B加的锁还没有执行完成，线程A实际释放的线程B加的锁。

![](/images/redis/redis-lock-误解除.png)

这个问题的解决在释放锁的实现中已经介绍过了。
### 超时解锁导致并发
如果线程A成功获取锁并设置过期时间30秒，但线程A执行时间超过了30秒，锁过期自动释放，此时线程B获取到了锁，线程A和线程B并发执行。

![](/images/redis/redis-lock-超时解锁.png)

A、B两个线程发生并发显然是不被允许的，一般有两种方式解决该问题：

    将过期时间设置足够长，确保代码逻辑在锁释放之前能够执行完成。
    为获取锁的线程增加守护线程，为将要过期但未释放的锁增加有效时间。

![](/images/redis/redis-lock-超时解锁解决.png)

### 无法等待锁释放
上述命令执行都是立即返回的，如果客户端可以等待锁释放就无法使用。

    可以通过客户端轮询的方式解决该问题，当未获取到锁时，等待一段时间重新获取锁，直到成功获取锁或等待超时。这种方式比较消耗服务器资源，当并发量比较大时，会影响服务器的效率。
    另一种方式是使用 Redis 的发布订阅功能，当获取锁失败时，订阅锁释放消息，获取锁成功后释放时，发送锁释放消息。如下：

![](/images/redis/redis-lock-无法等待锁释放.png)

### 集群主备切换
为了保证Redis的可用性，一般采用主从方式部署。主从数据同步有异步和同步两种方式，Redis 将指令记录在本地内存buffer中，然后异步将buffer中的指令同步到从节点，从节点一边执行同步的指令流来达到和主节点一致的状态，一边向主节点反馈同步情况。

在包含主从模式的集群部署方式中，当主节点挂掉时，从节点会取而代之，但客户端无明显感知。当客户端 A 成功加锁，指令还未同步，此时主节点挂掉，从节点提升为主节点，新的主节点没有锁的数据，当客户端 B 加锁时就会成功。

![](/images/redis/redis-lock-主备切换.png)

### 集群脑裂
集群脑裂指因为网络问题，导致Redis master节点跟slave节点和sentinel集群处于不同的网络分区，因为sentinel集群无法感知到master的存在，所以将slave节点提升为master节点，此时存在两个不同的master节点。Redis Cluster集群部署方式同理。当不同的客户端连接不同的master节点时，两个客户端可以同时拥有同一把锁。如下：

![](/images/redis/rredis-lock-集群脑裂.png)

