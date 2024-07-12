# Select
## 概述
select是一种go可以处理多个通道之间的机制,看起来和switch语句很相似,但是select其实和IO机制中的select一样,多路复用通道,随机选取一个进行执行,如果说通道(channel)实现了多个goroutine之前的同步或者通信,那么select则实现了多个通道(channel)的同步或者通信,并且select具有阻塞的特性. <br>
select 是 Go 中的一个控制结构,类似于用于通信的 switch 语句.每个 case 必须是一个通信操作,要么是发送要么是接收. <br>
select 随机执行一个可运行的 case.如果没有 case 可运行,它将阻塞,直到有 case 可运行.一个默认的子句应该总是可运行的. <br>
## 格式
```go
select {
    case <-ch1:
        // 如果从 ch1 信道成功接收数据,则执行该分支代码
    case ch2 <- 1:
        // 如果成功向 ch2 信道成功发送数据,则执行该分支代码
    default:
        // 如果上面都没有成功,则进入 default 分支处理流程
}
```
1. select里的case后面并不带判断条件,而是一个信道的操作,不同于switch里的case,对于从其它语言转过来的开发者来说有些需要特别注意的地方.
2. golang 的 select 就是监听 IO 操作,当 IO 操作发生时,触发相应的动作每个case语句里必须是一个IO操作,确切的说,应该是一个面向channel的IO操作.
3. select 语句借鉴自 Unix 的 select() 函数,在 Unix 中,可以通过调用 select() 函数来监控一系列的文件句柄,一旦其中一个文件句柄发生了 IO 动作,该 select() 调用就会被返回（C 语言中就是这么做的）,后来该机制也被用于实现高并发的 Socket 服务器程序.
4. Go 语言直接在语言级别支持 select关键字,用于处理并发编程中通道之间异步 IO 通信问题.
## 注意
1. 如果 ch1 或者 ch2 信道都阻塞的话,就会立即进入 default 分支,并不会阻塞.但是如果没有 default 语句,则会阻塞直到某个信道操作成功为止.
2. select语句只能用于信道的读写操作
3. select中的case条件(非阻塞)是并发执行的,select会选择先操作成功的那个case条件去执行,如果多个同时返回,则随机选择一个执行,此时将无法保证执行顺序.对于阻塞的case语句会直到其中有信道可以操作,如果有多个信道可操作,会随机选择其中一个 case 执行
4. 对于case条件语句中,如果存在信道值为nil的读写操作,则该分支将被忽略,可以理解为从select语句中删除了这个case语句
5. 如果有超时条件语句,判断逻辑为如果在这个时间段内一直没有满足条件的case,则执行这个超时case.如果此段时间内出现了可操作的case,则直接执行这个case.一般用超时语句代替了default语句
6. 对于空的select{},会引起死锁
7. 对于for中的select{}, 也有可能会引起cpu占用过高的问题
## 示例
### 超时操作
```go
package main

import (
    "time"
    "fmt"
)

var chs chan int = make(chan int,1)

func write(){
    time.Sleep(3 * time.Second)
    chs<-1
}

func read(){
    select {
    case ch1:=<-chs:
        fmt.Println(ch1)
        return
    case <-time.After(2 * time.Second):
        fmt.Println("read time out")
        return
    }
}

func main() {
    go write()
    read()
}
在文本的用例中,write（） 在三秒后才会向通道发送信号,而read（） tiamout只有两秒,程序的运行结果就是打印time out：read time out
```
### 竞争选举
select这个特性到底有什么用呢,下面我们来介绍一些使用select的场景
这个是最常见的使用场景,多个通道,有一个满足条件可以读取,就可以“竞选成功”
```go
    select {
    case i := <-ch1:
        fmt.Printf("从ch1读取了数据%d", i)
    case j := <-ch2:
        fmt.Printf("从ch2读取了数据%d", j)
    case m := <- ch3
        fmt.Printf("从ch3读取了数据%d", m)
    ...
    }
```
### 超时处理
保证不阻塞
```go
select {
    case str := <- ch1
        fmt.Println("receive str", str)
    case <- time.After(time.Second * 5): 
        fmt.Println("timeout!!")
}
```
因为select是阻塞的,我们有时候就需要搭配超时处理来处理这种情况,超过某一个时间就要进行处理,保证程序不阻塞.
### 判断buffered channel是否阻塞
比如我们有一个有限的资源（这里用buffer channel实现）,我们每一秒向bufChan传送数据,由于生产者的生产速度大于消费者的消费速度,故会触发default语句,这个就很像我们web端来显示并发过高的提示了,小伙伴们可以尝试删除go func中的time.Sleep(5*time.Second),看看是否还会触发default语句
```go
package main
import (
    "fmt" 
    "time"
)

func main()  {
    bufChan := make(chan int, 5)
    
    go func ()  {
        time.Sleep(time.Second)
        for {
            <-bufChan
            time.Sleep(5*time.Second)
        }
    }() 
    

    for {
        select {    
        case bufChan <- 1:  
            fmt.Println("add success")
            time.Sleep(time.Second)  
        default:        
            fmt.Println("资源已满,请稍后再试")
            time.Sleep(time.Second) 
        } 
    }
}
```
### 阻塞main函数
有时候我们会让main函数阻塞不退出,如http服务,我们会使用空的select{}来阻塞main goroutine
```go
package main
import (
    "fmt"
    "time"
)

func main()  {
    bufChan := make(chan int)
    
    go func() {
        for{
            bufChan <-1
            time.Sleep(time.Second)
        }
    }()


    go func() {
        for{
            fmt.Println(<-bufChan)
        }
    }()
     
    select{}
}
```
### 随机执行
如果有一个或多个IO操作可以完成,则Go运行时系统会随机的选择一个执行,否则的话,如果有default分支,则执行default分支语句,如果连default都没有,则select语句会一直阻塞,直到至少有一个IO操作可以进行
```go
start := time.Now()
    c := make(chan interface{})
    ch1 := make(chan int)
        ch2 := make(chan int)

    go func() {

        time.Sleep(4*time.Second)
        close(c)
    }()

    go func() {

        time.Sleep(3*time.Second)
        ch1 <- 3
    }()

      go func() {

        time.Sleep(3*time.Second)
        ch2 <- 5
    }()

    fmt.Println("Blocking on read...")
    select {
    case <- c:

        fmt.Printf("Unblocked %v later.\n", time.Since(start))

    case <- ch1:

        fmt.Printf("ch1 case...")
      case <- ch2:

        fmt.Printf("ch1 case...")
    default:

        fmt.Printf("default go...")
    }
```
运行上述代码,由于当前时间还未到3s.所以,目前程序会走default.
### 使用 select 实现 timeout 机制
```go
    timeout := make (chan bool, 1)
    go func() {
        time.Sleep(1e9) // sleep one second
        timeout <- true
    }()
    select {
    case <- timeout:
        fmt.Println("timeout!")
    }
```
### 使用 select 语句来检测 chan 是否已满
```go
ch2 := make (chan int, 1)
    ch2 <- 1
    select {
    case ch2 <- 2:
    default:
        fmt.Println("channel is full !")
    }

for-select
package main

import (
    "fmt"
    "time"
)

func main() {
    var  errChan = make(chan int)
    //定时2s
    ticker := time.NewTicker(2 * time.Second)
    defer ticker.Stop()
    go func(a chan int) {
        //5s发一个信号
        time.Sleep(time.Second * 5)
        errChan <- 1
    }(errChan)
    LOOP:
        for {
            select {
                case <-ticker.C: {
                    fmt.Println("Task still running")
                }
                case res, ok := <-errChan:
                    if ok {
                        fmt.Println("chan number:", res)
                        break LOOP
                    }
            }
        }
    fmt.Println("end!!!")
}
//输出结果：
//Task still running
//Task still running
//chan number: 1
//end!!!
```