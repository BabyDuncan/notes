Scala vs. Go TCP Benchmark



We recently found ourselves in need of some load balancing with a few special features that weren’t available off the shelf quite the way we wanted.
We thus set out on a little investigation on what it would take to write our own software load balancer. Since most of our code base and expertise is in Scala, building this on top of the JVM would be a natural choice.
On the other hand, a lot of people, including ourselves at Fortytwo, make the often – but not always – unfounded assumption that the JVM is slower than natively compiled languages.
Since a load balancer is usually an extremely performance-critical component, perhaps a different programming language/environment would be better?
We weren’t quite willing to go all the way down the rabbit hole and start writing C/C++, so we started looking for a middle ground that would give us the purported performance advantage of native code while still having higher level features like garbage collection and built-in concurrency primitives. One such language that came up almost immediately was Google’s relatively new Go language. Natively compiled and super nice build in concurrency constructs. Perfect?

Go vs Scala

We decided to benchmark the TCP network stack processing overhead of Go vs. Scala in a very similar fashion to the recent WebSockets vs. TCP post.
We wrote a simple “ping-pong” client and server in both Go

  

//SERVER
package main
 
import (
    "net"
    "runtime"
)
 
func handleClient(conn net.Conn) {
    defer conn.Close()
 
    var buf [4]byte
    for {
        n, err := conn.Read(buf[0:])
        if err!=nil {return}
        if n>0 {
            _, err = conn.Write([]byte("Pong"))
            if err!=nil {return}
        }
    }
}
 
func main() {
    runtime.GOMAXPROCS(4)
 
    tcpAddr, _ := net.ResolveTCPAddr("tcp4", ":1201")
    listener, _ := net.ListenTCP("tcp", tcpAddr)
 
    for {
        conn, _ := listener.Accept()
        go handleClient(conn)
    }
}


	

//CLIENT
package main
 
import (
    "net"
    "fmt"
    "time"
    "runtime"
)
 
func ping(times int, lockChan chan bool) {
    tcpAddr, _ := net.ResolveTCPAddr("tcp4", "localhost:1201")
    conn, _ := net.DialTCP("tcp", nil, tcpAddr)
 
    for i:=0; i<int(times); i++ {
        _, _ = conn.Write([]byte("Ping"))
        var buff [4]byte
        _, _ = conn.Read(buff[0:])
    }
    lockChan<-true
    conn.Close()    
}
 
func main() {
    runtime.GOMAXPROCS(4)
 
    var totalPings int = 1000000
    var concurrentConnections int = 100
    var pingsPerConnection int = totalPings/concurrentConnections
    var actualTotalPings int = pingsPerConnection*concurrentConnections
 
    lockChan := make(chan bool, concurrentConnections)
 
    start := time.Now()
    for i:=0; i<concurrentConnections; i++{
        go ping(pingsPerConnection, lockChan)
    }
    for i:=0; i<int(concurrentConnections); i++{
        <-lockChan 
    }
    elapsed := 1000000*time.Since(start).Seconds()
    fmt.Println(elapsed/float64(actualTotalPings))
}

and Scala

	

//SERVER
import java.net._
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent._
 
object main{
 
    def handleClient(s: Socket) : Unit = {
      val in = s.getInputStream
      val out = s.getOutputStream
      while(s.isConnected){
        val buffer = Array[Byte](4)
        in.read(buffer)
        out.write("Pong".getBytes)
      }
    }
 
    def main(args: Array[String]){
      val server = new ServerSocket(1201)
      while(true){
        val s: Socket = server.accept()
        future { handleClient(s) }
      }
    }
}

	

//CLIENT
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global
import java.net._
 
object main{
 
    def ping(timesToPing: Int) : Unit = {
        val socket = new Socket("localhost", 1201)
        val out = socket.getOutputStream
        val in = socket.getInputStream
        for (i <- 0 until timesToPing) {
            out.write("Ping".getBytes)
            val buffer = Array[Byte](4)
            in.read(buffer)
        }
        socket.close
    }
 
    def main(args: Array[String]){
        var totalPings = 1000000
        var concurrentConnections = 100
        var pingsPerConnection : Int = totalPings/concurrentConnections
        var actualTotalPings : Int = pingsPerConnection*concurrentConnections
 
        val t0 = (System.currentTimeMillis()).toDouble
        var futures = (0 until concurrentConnections).map{_ => 
            future(ping(pingsPerConnection))
        }
 
        Await.result(Future.sequence(futures), 1 minutes)
        val t1 = (System.currentTimeMillis()).toDouble
        println(1000*(t1-t0)/actualTotalPings)
    }
}

The latter is almost exactly the same as the one used in the WebSockets vs. TCP benchmark. Both implementations are fairly naive and there is probably room for improvement. The actual test code contained some functionality to deal with connection errors, omitted here for brevity.
The client makes a certain number of persistent concurrent connections to the server and sends a certain number of pings (just the string “Ping”) to each of which the server will respond with the string “Pong”.

The experiments where performed on a 2.7Ghz quad core MacBook Pro with both client and server running locally, so as to better measure pure processing overhead. The client would make 100 concurrent connections and send a total of 1 million pings to the server, evenly distributed over the connections. We measured the average round trip time.

To our surprise, Scala did quite a bit better than Go, with an average round trip time of ~1.6 microseconds (0.0016 milliseconds) vs. Go’s ~11 microseconds (0.011 milliseconds). The numbers for Go are of course still extremely fast, but if almost all your software is doing is taking in a tcp packet and passing on to another endpoint, this can mean a big difference in maximum throughput.

Notable in the opposite direction was that the Go server had a memory footprint of only about 10 MB vs. Scala’s nearly 200 MB.

Go is still new, will likely make performance improvements as it matures and its simple concurrency primitives might make the loss in performance worth it.
Nonetheless, it’s a somewhat surprising result, at least to us. We would love to hear some thoughts in the comments!
