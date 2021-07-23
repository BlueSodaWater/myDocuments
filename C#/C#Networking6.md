## Ping

你在Linux，OS X，Windows上使用过[ping](https://en.wikipedia.org/wiki/Ping_(networking_utility))工具吗？这就是那个。这篇会放松一点来和你解释[Ping](https://msdn.microsoft.com/en-us/library/system.net.networkinformation.ping(v=vs.110).aspx) 类。微软在[System.Net.NetworkInformation](https://msdn.microsoft.com/en-us/library/system.net.networkinformation(v=vs.110).aspx)命名空间提供了一些工具避免我们来写自己的ICMP代码。它的操作很简单，即使用`Ping`类然后处理[PingReply](https://msdn.microsoft.com/en-us/library/system.net.networkinformation.pingreply(v=vs.110).aspx)。除非你在写一个多用户的程序（游戏），否则你会发现这没有多大用处。

下面是同步/异步使用这个类的例子，很简单。

```c#
using System;
using System.Threading;
using System.Net.NetworkInformation;

namespace PingExample
{
    // This is an example of how to use Ping both sychrnously
    // and asychronously (with callbacks instead of Tasks).
    class PingExample
    {
        private static string _hostname;
        private static int _timeout = 2000;     // 2 seconds in milliseconds
        private static object _consoleLock = new object();

        // Locks console output and prints info about a PingReply (with color)
        public static void PrintPingReply(PingReply reply, ConsoleColor textColor)
        {
            lock(_consoleLock)
            {
                Console.ForegroundColor = textColor;

                Console.WriteLine("Got ping response from {0}", _hostname);
                Console.WriteLine("  Remote address: {0}", reply.Address);
                Console.WriteLine("  Roundtrip Time: {0}", reply.RoundtripTime);
                Console.WriteLine("  Size: {0} bytes", reply.Buffer.Length);
                Console.WriteLine("  TTL: {0}", reply.Options.Ttl);

                Console.ResetColor();
            }
        }

        // A callback for doing an Asynchronous ping
        public static void PingCompletedHandler(object sender, PingCompletedEventArgs e)
        {
            // Canceled, error, or fine?
            if (e.Cancelled)
                Console.WriteLine("Ping was canceled.");
            else if (e.Error != null)
                Console.WriteLine("There was an error with the Ping, reason={0}", e.Error.Message);
            else  
                PrintPingReply(e.Reply, ConsoleColor.Cyan);

            // Notify the calling thread
            AutoResetEvent waiter = (AutoResetEvent)e.UserState;
            waiter.Set();
        }

        // Performs a Synchronous Ping
        public static void SendSynchronousPing(Ping pinger, ConsoleColor textColor)
        { 
            PingReply reply = pinger.Send(_hostname, _timeout);    // will block for at most 2 seconds
            if (reply.Status == IPStatus.Success)
                PrintPingReply(reply, ConsoleColor.Magenta);
            else
            {
                Console.WriteLine("Synchronous Ping to {0} failed:", _hostname);
                Console.WriteLine("  Status: {0}", reply.Status);
            }
        }

        public static void Main(string[] args)
        {
            // Setup the pinger
            Ping pinger = new Ping();
            pinger.PingCompleted += PingCompletedHandler;

            // Poll the user where to send the Ping to
            Console.Write("Send a Ping to whom: ");
            _hostname = Console.ReadLine();

            // Send async (w/ callback)
            AutoResetEvent waiter = new AutoResetEvent(false);  // Set to not-signaled
            pinger.SendAsync(_hostname, waiter);

            // Check immediately for the async ping
            if (waiter.WaitOne(_timeout) == false)
            {
                pinger.SendAsyncCancel();
                Console.WriteLine("Async Ping to {0} timed out.", _hostname);
            }
            
            // Send it synchronously
            SendSynchronousPing(pinger, ConsoleColor.Magenta);
        }
    }
}
```

`Printing()`是一个小方法来将一些关于[PingReply](https://msdn.microsoft.com/en-us/library/system.net.networkinformation.pingreply(v=vs.110).aspx)信息打印到控制台上。注意这里锁定了控制台并且请求颜色。这样使得同步方法和异步方法的输出不会交错。颜色帮助我们区分谁是谁。

`PingCompletedHandler()`是[Ping.SendAsync()](https://msdn.microsoft.com/en-us/library/ms144962(v=vs.110).aspx)方法的回调函数。它附在`Ping.PingCompleted`事件上。无论ICMP响应是否到达，都会被调用。当我们在我们的`pinger`中调用`SendAsync()`我们需要发送一个userToken。这在异步回调中是唯一的。在我们这里我们使用[AutoResetEvent](https://msdn.microsoft.com/en-us/library/system.threading.autoresetevent(v=vs.110).aspx)。这用来通知调用`Thread`是否异步方法结束，因此即使Ping没有成功，`Set()`也必须要被`waiter`调用。

`SendSynchronousPing()`调用[Ping.Send()](https://msdn.microsoft.com/en-us/library/ms144956(v=vs.110).aspx) ，然后根据`PingReply`是否成功（通过Status属性检查）打印一些信息。`Send()`方法有很多重载，这里我们使用的函数附加了一个超时（设置为2000毫秒）

你可能注意到在`Ping`中有两个不一样的异步方法。`SendAsync()`和`SendPingAsync()`。第一个用回调（就像例子中一样）。第二个完成时将会返回一个包装在`Task`中的`PingReply`。因此他本质上是同步版本的`Send()`的异步版本。

在`Main()`方法我们通过输入一个名字来ping。所有的`Ping`方法都会为我们解析DNS。首先我们尝试通过异步方法发送ICMP，然后是同步方法。注意异步方法中的`_timeout`不一定要放在`WaitOne()`中调用的，也可以在`SendAsync()`中。

把`_timeout`放在`SendAsync()`中调用和`AutoResetEvent`中的`WaitOne()`有何区别》

在第一个中，超时当我们异步调用开始的时候就开始了。如果异步方法在这段时间内没有完成，它将终止。但是如果我们放在`AutoResetEvent`中，他会等待直至超时，而不是在调用的时候就开始。这样去想他“确保ping在x毫秒后完成”，或者“检测ping能否在x毫秒内某一点完成”。

### PingReply

这没有很多可以讲。这是一个包含我们ping信息的数据结构。最重要部分就是`Status`和`RoundtripTime`。第一个可以是[IPStatus](https://msdn.microsoft.com/en-us/library/system.net.networkinformation.ipstatus(v=vs.110).aspx)中的任何内容。第二个对我们真正有用。他代表ping到达目的地再回到我们需要多久（毫秒）。根据网络结构的本质，过去的时间和回来的时间实际上是不同的。你不能简单的将值除2然后想“这就是发送一条信息所需要的时间”，这是完全错误的。

### PingOptions

在我们的代码中，我们没有使用[PingOptions](https://msdn.microsoft.com/en-us/library/system.net.networkinformation.pingoptions(v=vs.110).aspx)类。他只有两个字段`DontFragment`和`Ttl`。第一个是一个布尔值告诉我们ping是否对ICMP数据报切片，默认值是false。除了设置更大的缓冲区之外，这对测试ping所通过的设备的MTU也很有用。第二个代表“存活的时间”，直白的说就是“在ping消失之前在设备之中弹跳了多少次“，默认值是128，但是设置更小的值来有助于来测试你的连接在互联网上跳多少次才能到达目的地。



