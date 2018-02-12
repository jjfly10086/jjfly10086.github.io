### Future模式
future的原理是当你申请资源（计算资源或I/O资源）时，立即返回一个虚拟的资源句柄，当真正使用的时候，再将虚拟的句柄转化成真正的资源，相当于预获取。
提交任务后会立即返回一个Future,真正的耗时的任务提交给线程池去执行,Future中包含了当前任务的执行完成状态(Future.isDone()),期间主线程可以执行其他操作，
当需要获取任务的返回结果时，通过Future.get()同步获取(可通过isDone判断是否执行完毕，完成后再get返回结果);
```java
/**
* 线程池提交任务：
*/
class Test{
    public void test(){
         Future<User> future = Executors.newFixedThreadPool(3).submit(new Callable<User>() {
                        @Override
                        public User call() throws Exception {
                            User user = userRepository.findById(id).get();
                            TimeUnit.SECONDS.sleep(10);
                            return user;
                        }
                    });
         /**
         * 可以执行其他逻辑
         */
         //需要用到线程返回结果时：此方法为同步方法，如果线程未执行完，会一直阻塞
         future.get();
         future.get(10,TimeUnit.SECONDS);//等待指定时间，未获取到抛出异常
         future.isDone();//判断线程结果是否执行完成
         future.cancel();//取消任务执行
         future.isCancel();//任务是否已经取消
    }
}
```