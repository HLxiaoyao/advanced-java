# advanced-java
本项目致力于从源码层面，剖析和挖掘互联网行业主流技术的底层实现原理，目前包括有Spring 全家桶、Mybatis、Netty、Dubbo 框架，及 Redis、Tomcat 中间件等。在分析框架源码之后，还会总结这些框架的应用、应用中问题的解决思路等

## [Redis](https://github.com/HLxiaoyao/advanced-java/tree/main/docs/redis)
* [Redis底层数据结构](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/redis/Redis%E5%BA%95%E5%B1%82%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.md)
* [Redis对象](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/redis/Redis%E5%AF%B9%E8%B1%A1.md)
* [有哪些缓存算法？是否能手写一下缓存算法的实现？](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/redis/%E7%BC%93%E5%AD%98%E6%B7%98%E6%B1%B0%E7%AE%97%E6%B3%95.md)
* [如何避免缓存”穿透”问题？](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/redis/%E5%A6%82%E4%BD%95%E9%81%BF%E5%85%8D%E7%BC%93%E5%AD%98%E2%80%9D%E7%A9%BF%E9%80%8F%E2%80%9D%E7%9A%84%E9%97%AE%E9%A2%98%EF%BC%9F.md)
* [如何避免缓存”击穿”的问题？](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/redis/%E5%A6%82%E4%BD%95%E9%81%BF%E5%85%8D%E7%BC%93%E5%AD%98%E2%80%9D%E5%87%BB%E7%A9%BF%E2%80%9D%E7%9A%84%E9%97%AE%E9%A2%98%EF%BC%9F.md)
* [如何避免缓存”雪崩”的问题？](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/redis/%E5%A6%82%E4%BD%95%E9%81%BF%E5%85%8D%E7%BC%93%E5%AD%98%E2%80%9D%E9%9B%AA%E5%B4%A9%E2%80%9D%E7%9A%84%E9%97%AE%E9%A2%98%EF%BC%9F.md)
* [Redis的线程模型？为什么Redis单线程模型也能效率这么高？为什么采用单线程模型？Redis是单线程的，如何提高多核CPU的利用率？](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/redis/Redis%E7%9A%84%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B%EF%BC%9F%E4%B8%BA%E4%BB%80%E4%B9%88Redis%E5%8D%95%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B%E4%B9%9F%E8%83%BD%E6%95%88%E7%8E%87%E8%BF%99%E4%B9%88%E9%AB%98%EF%BC%9F%E4%B8%BA%E4%BB%80%E4%B9%88%E9%87%87%E7%94%A8%E5%8D%95%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B%EF%BC%9FRedis%E6%98%AF%E5%8D%95%E7%BA%BF%E7%A8%8B%E7%9A%84%EF%BC%8C%E5%A6%82%E4%BD%95%E6%8F%90%E9%AB%98%E5%A4%9A%E6%A0%B8CPU%E7%9A%84%E5%88%A9%E7%94%A8%E7%8E%87%EF%BC%9F.md)
* [Redis有几种数据“过期”策略？Redis有哪几种数据“淘汰”策略？](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/redis/Redis%E6%9C%89%E5%87%A0%E7%A7%8D%E6%95%B0%E6%8D%AE%E2%80%9C%E8%BF%87%E6%9C%9F%E2%80%9D%E7%AD%96%E7%95%A5%EF%BC%9FRedis%E6%9C%89%E5%93%AA%E5%87%A0%E7%A7%8D%E6%95%B0%E6%8D%AE%E2%80%9C%E6%B7%98%E6%B1%B0%E2%80%9D%E7%AD%96%E7%95%A5%EF%BC%9F.md)
* [如何使用Redis实现消息队列？
](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/redis/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8Redis%E5%AE%9E%E7%8E%B0%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97%EF%BC%9F.md)
* [如何使用Redis实现分布式锁？
](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/redis/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8Redis%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%EF%BC%9F.md)
* [Redis重要的健康指标？如何提高 Redis 命中率？](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/redis/Redis%E9%87%8D%E8%A6%81%E7%9A%84%E5%81%A5%E5%BA%B7%E6%8C%87%E6%A0%87%EF%BC%9F%E5%A6%82%E4%BD%95%E6%8F%90%E9%AB%98%20Redis%20%E5%91%BD%E4%B8%AD%E7%8E%87%EF%BC%9F.md)



## [MyBatis](https://github.com/HLxiaoyao/advanced-java/tree/main/docs/Mybatis)
* [MyBatis配置文件解析源码分析](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/Mybatis/MyBatis%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E8%A7%A3%E6%9E%90%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)


## [RocketMQ](https://github.com/HLxiaoyao/advanced-java/tree/main/docs/RocketMQ)
* [路由中心NameServer源码分析](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/RocketMQ/%E8%B7%AF%E7%94%B1%E4%B8%AD%E5%BF%83NameServer%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)
* [路由中心NameServer总结](https://github.com/HLxiaoyao/advanced-java/blob/main/docs/RocketMQ/%E8%B7%AF%E7%94%B1%E4%B8%AD%E5%BF%83NameServer%E6%80%BB%E7%BB%93.md)


