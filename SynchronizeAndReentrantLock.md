#### Reentrantlock和synchronize 都可以实现锁机制,但是reentrantlock具有比synchronize更多的功能.

    1,reentrantLock 具有锁超时功能,即 在一定时间内如果没有获得锁,那么就执行一个其他动作,synchronize 只会一直等待锁的释放.
    代码如下:
    
    package com.sohu.sce.profile.adaptor;
    
    import org.apache.log4j.Logger;
    
    import java.util.concurrent.CountDownLatch;
    import java.util.concurrent.TimeUnit;
    import java.util.concurrent.locks.ReentrantLock;
    
    /**
     * synchronize and ReentrantLock
     * User: guohaozhao (guohaozhao@sohu-inc.com)
     * Date: 13-7-24 10:14
     */
    public class SyncReen {
    
        public static void main(String... args) {
    
            final Logger logger = Logger.getLogger(SyncReen.class);
    
            final ReentrantLock reentrantLock = new ReentrantLock();
    
            final byte[] lock = new byte[0];
    
            final CountDownLatch countDownLatch = new CountDownLatch(2);
    
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        logger.error(e.getMessage(), e);
                    }
                    while (true) {
                        try {
                            if (reentrantLock.tryLock(3, TimeUnit.SECONDS)) {
                                logger.info("get lock !!");
                            }
                            logger.info("do not get lock!!");
                            countDownLatch.countDown();
                        } catch (InterruptedException e) {
                            logger.error(e.getMessage(), e);
                        } finally {
                            reentrantLock.unlock();
                        }
                    }
                }
            }).start();
    
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        reentrantLock.lock();
                        logger.info("locking !!");
                        Thread.sleep(5000);
                        logger.info("unlock .");
                    } catch (InterruptedException e) {
                        logger.error(e.getMessage(), e);
                    } finally {
                        reentrantLock.unlock();
                        countDownLatch.countDown();
                    }
                }
            }).start();
    
            try {
                countDownLatch.await();
            } catch (InterruptedException e) {
                logger.error(e.getMessage(), e);
            }
    
            logger.info("done !!");
    
        }
    
    }
    
    对于此段的执行结果:
    2013-07-24 10:44:47,155 INFO  -> locking !!
    2013-07-24 10:44:51,157 INFO  -> do not get lock!!
    Exception in thread "Thread-1" java.lang.IllegalMonitorStateException
        at java.util.concurrent.locks.ReentrantLock$Sync.tryRelease(ReentrantLock.java:127)
    	at java.util.concurrent.locks.AbstractQueuedSynchronizer.release(AbstractQueuedSynchronizer.java:1239)
    	at java.util.concurrent.locks.ReentrantLock.unlock(ReentrantLock.java:431)
    	at com.sohu.sce.profile.adaptor.SyncReen$1.run(SyncReen.java:44)
    	at java.lang.Thread.run(Thread.java:680)
    2013-07-24 10:44:52,157 INFO  -> unlock .
    2013-07-24 10:44:52,158 INFO  -> done !!
    
    Process finished with exit code 0
