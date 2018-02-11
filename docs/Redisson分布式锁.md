### 实现原理
```
用一个状态值表示锁，对锁的占用和释放通过状态值来标识。
Redis为单进程单线程模式，采用队列模式将并发访问变成串行访问，且多客户端对Redis的连接并不存在竞争关系。
redis的SETNX命令可以方便的实现分布式锁。
SETNX命令（SET if Not eXists） 语法： 
SETNX key value 功能： 当且仅当 key 不存在，将 key 的值设为 value ，并返回1；若给定的 key 已经存在，则 SETNX 不做任何动作，并返回0。
```
