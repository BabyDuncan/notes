This article talks only about Java servers. See [this page](https://github.com/m1ch1/mapkeeper/wiki/TThreadedServer-vs.-TNonblockingServer) if you are interested in C++ servers. 

[Thrift](http://thrift.apache.org/) is a cross-language serialization/RPC framework with three major components: protocol, transport, and server. Protocol defines how messages are serialized. Transport defines how messages are communicated between client and server. Server receives serialized messages from the transport, deserializes them according to the protocol and invokes user-defined message handlers, and serializes the responses from the handlers and writes them back to the transport. The modular architecture of Thrift allows it to offer various choices of servers. Here are the list of server available for Java:

* [TSimpleServer](http://svn.apache.org/viewvc/thrift/trunk/lib/java/src/org/apache/thrift/server/TSimpleServer.java?view=markup)
* [TNonblockingServer](http://svn.apache.org/viewvc/thrift/trunk/lib/java/src/org/apache/thrift/server/TNonblockingServer.java?view=markup)
* [THsHaServer](http://svn.apache.org/viewvc/thrift/trunk/lib/java/src/org/apache/thrift/server/THsHaServer.java?view=markup)
* [TThreadedSelectorServer](http://svn.apache.org/viewvc/thrift/trunk/lib/java/src/org/apache/thrift/server/TThreadedSelectorServer.java?view=markup)
* [TThreadPoolServer](http://svn.apache.org/viewvc/thrift/trunk/lib/java/src/org/apache/thrift/server/TThreadPoolServer.java?view=markup)

Having choices is great, but which server is right for you? In this article, I'll describe the differences among all those servers and show benchmark results to illustrate performance characteristics (the details of the benchmark is explained in Appendix B). Let's start with the simplest one: TSimpleServer.

## TSimpleServer

**TSimpleServer** accepts a connection, processes requests from the connection until the client closes the connection, and goes back to accept a new connection. Since ***it is all done in a single thread with blocking I/O***, it can only serve one client connection, and all the other clients will have to wait until they get accepted. TSimpleServer is mainly used for testing purpose. Don't use it in production!

## TNonblockingServer vs. THsHaServer

**TNonblockingServer** solves the problem with TSimpleServer of one client blocking all the other clients by using non-blocking I/O. It uses [`java.nio.channels.Selector`](http://docs.oracle.com/javase/1.4.2/docs/api/java/nio/channels/Selector.html), which allows you to get blocked on multiple connections instead of a single connection by calling [`select()`](http://docs.oracle.com/javase/1.4.2/docs/api/java/nio/channels/Selector.html#select%28%29). The `select()` call returns when one ore more connections are ready to be accepted/read/written. TNonblockingServer handles those connections either by accepting it, reading data from it, or writing data to it, and calls `select()` again to wait for the next available connections. This way, multiple clients can be served without one client starving others. 

There is a catch, however. ***Messages are processed by the same thread that calls `select()`***. Let's say there are 10 clients, and each message takes 100 ms to process. What would be the latency and throughput? While a message is being processed, 9 clients are waiting to be selected, so it takes 1 second for the clients to get the response back from the server, and throughput will be 10 requests / second. Wouldn't it be great if multiple messages can be processed simultaneously? 

This is where **THsHaServer** (Half-Sync/Half-Async server) comes into picture. It uses a single thread for network I/O, and a separate pool of worker threads to handle message processing. This way messages will get processed immediately if there is an idle worker threads, and multiple messages can be processed concurrently. Using the example above, now the latency is 100 ms and throughput will be 100 requests / sec. 

To demonstrate this, I ran a benchmark with 10 clients and a modified message handler that simply sleeps for 100 ms before returning. I used THsHaServer with 10 worker threads. The handler looks something like this:

    public ResponseCode sleep() throws TException
    {   
        try {
            Thread.sleep(100);
        } catch (Exception ex) {
        }
        return ResponseCode.Success;
    }


![](https://github.com/m1ch1/mapkeeper/raw/master/ycsb/data/thrift2/sleep_throughput.png)
![](https://github.com/m1ch1/mapkeeper/raw/master/ycsb/data/thrift2/sleep_latency.png)

The results are as expected. THsHaServer is able to process all the requests concurrently, while TNonblockingServer processes requests one at a time.

## THsHaServer vs. TThreadedSelectorServer

Thrift 0.8 introduced yet another server, **TThreadedSelectorServer**. The main difference between TThreadedSelectorServer and THsHaServer is that ***TThreadedSelectorServer allows you to have multiple threads for network I/O***. It maintains 2 thread pools, one for handling network I/O, and one for handling request processing. TThreadedSelectorServer performs better than THsHaServer when the network io is the bottleneck. To show the difference, I ran a benchmark with a handler that returns immediately without doing anything, and measured the average latency and throughput with varying number of clients. I used 32 worker threads for THsHaServer, and 16 worker threads/16 selector threads for TThreadedSelectorServer.

![](https://github.com/m1ch1/mapkeeper/raw/master/ycsb/data/thrift2/num_clients_throughput.png)
![](https://github.com/m1ch1/mapkeeper/raw/master/ycsb/data/thrift2/num_clients_latency.png)

The result shows that TThreadedSelectorServer has much higher throughput than THsHaServer while maintaining lower latency. 

## TThreadedSelectorServer vs. TThreadPoolServer

Finally, there is **TThreadPoolServer**. TThreadPoolServer is different from the other 3 servers in that:

* There is a dedicated thread for *accepting* connections.
* Once a connection is accepted, it gets scheduled to be processed by a worker thread in [`ThreadPoolExecutor`](http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/ThreadPoolExecutor.html).
* ***The worker thread is tied to the specific client connection until it's closed***. Once the connection is closed, the worker thread goes back to the thread pool.
* You can configure both minimum and maximum number of threads in the thread pool. Default values are 5 and `Integer.MAX_VALUE`, respectively. 

This means that if there are 10000 concurrent client connections, you need to run 10000 threads. As such, it is not as resource friendly as other servers. Also, if the number of clients exceeds the maximum number of threads in the thread pool, requests will be blocked until a worker thread becomes available. 

Having said that, TThreadPoolServer performs very well; on the box I'm using it's able to support 10000 concurrent clients without any problem. If you know the number of clients that will be connecting to your server in advance and you don't mind running a lot of threads, TThreadPoolServer might be a good choice for you.

![](https://github.com/m1ch1/mapkeeper/raw/master/ycsb/data/thrift2/num_clients_throughput_pool.png)
![](https://github.com/m1ch1/mapkeeper/raw/master/ycsb/data/thrift2/num_clients_latency_pool.png)

## Conclusion

I hope this article helps you decide which Thrift server is right for you. I think TThreadedSelectorServer would be a safe choice for most of the use cases. You might also want to consider TThreadPoolServer if you can afford to run lots of concurrent threads. Feel free to send me email at <mapkeeper-users@googlegroups.com> or post your comments [here](http://groups.google.com/group/mapkeeper-users) if you have any questions/comments.

## Appendix A: Hardware Configuration

    Processors:     2 x Xeon E5620 2.40GHz (HT enabled, 8 cores, 16 threads)
    Memory:         8GB
    Network:        1Gb/s <full-duplex>
    OS:             RHEL Server 5.4 Linux 2.6.18-164.2.1.el5 x86_64


## Appendix B: Benchmark Details

It's pretty straightforward to run the benchmark yourself. First clone the [MapKeeper](https://github.com/m1ch1/mapkeeper) repository and compile the [stub java server](https://github.com/m1ch1/mapkeeper/blob/master/stubjava/StubServer.java):

    git clone git://github.com/m1ch1/mapkeeper.git
    cd mapkeeper/stubjava
    make

Then, start the server you like to benchmark:

    make run mode=threadpool    # run TThreadPoolServer
    make run mode=nonblocking   # run TNonblockingServer
    make run mode=hsha          # run THsHaServer
    make run mode=selector      # run TThreadedSelectorServer

Then, clone [YCSB](https://github.com/brianfrankcooper/YCSB) repository and compile:

    git clone git://github.com/brianfrankcooper/YCSB.git
    cd YCSB
    mvn clean package
    
Once the compilation finishes, you can run YCSB against the stub server:

    ./bin/ycsb load mapkeeper -P ./workloads/workloada

For more detailed information about how to use YCSB, check out their [wiki page](https://github.com/brianfrankcooper/YCSB/wiki/Running-a-Workload). 
