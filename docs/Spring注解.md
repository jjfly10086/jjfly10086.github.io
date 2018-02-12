### @Lazy
```
等同于<bean class="" lazy-init="true"/>
表示该bean不会在applicationContext初始化的时候实例化，在第一次向容器通过getBean索取bean时实例化的。
```
### @Async
```
加在方法头部：
spring aop代理执行方法,将方法执行丢入线程池中,不会阻塞主线程实现异步执行
```
### Java多线程中的设计模式 
 - future
 - Master-Worker
 - Guarded Suspeionsion
 - 不变模式
 - 生产者-消费者模式
```

```
### new Thread().start思考
```
针对主线程中新开一个线程的问题：
start()新起一个线程去运行run方法，但是对main这个线程来说，他会继续往下走，一个是新起线程去执行，一个是在本线程（已经在执行的线程）执行。
（在程序运行时，主线程已经启动并在运行中，而另外起一个线程start表示线程处于就绪状态，还要等JVM机制调用进入运行状态）
调试多线程最好用日志方式,DEBUG模式的输出结果不能确定
```