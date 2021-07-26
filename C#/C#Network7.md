## Socket Class Intro

在 [TcpClient](https://msdn.microsoft.com/en-us/library/system.net.sockets.tcpclient(v=vs.110).aspx)，[TcpListener](https://msdn.microsoft.com/en-us/library/system.net.sockets.tcplistener(v=vs.110).aspx)，[UdpClient](https://msdn.microsoft.com/en-us/library/system.net.sockets.udpclient(v=vs.110).aspx)（以及其他很多的类中）底层都是基于[Socket](https://msdn.microsoft.com/en-us/library/system.net.sockets.socket(v=vs.110).aspx)对象。如果你希望找一些简单的来使用或者快速开发应用，我推荐你先使用这些对象。如果你希望更多的性能或者更接近[Berkeley/POSIX Socket API](https://en.wikipedia.org/wiki/Berkeley_sockets)的东西，`Socket`类是你想要的。

在这个系列中，我会先给你展示一些C的代码（使用Berkeley Socket API）然后给你展示这部分代码的C#版本。C代码是从[Wikibooks article](https://en.wikibooks.org/wiki/C_Programming/Networking_in_UNIX)改变的。我不会给你展示任何一部程序，我想要事情简单化，我们会在下个章节包含异步的`Socket`。

### The Application

应用很简单，服务器将会开启一个套接字然后等待连接，一旦客户端连接上了，会在终端窗口打印一些信息然后给客户端响应一些信息，一旦信息发送出去就会和客户端断开连接，所有这些都会通过TCP运行。

我在这里不打算解释发生了什么，因为代码很直观，到处都是有注释来解释每一行是干什么的。

### Socket Class Intro - C# Port

### Server

```c#
using System;
using System.Text;
using System.Net;
using System.Net.Sockets;

namespace Server
{
    class TcpSocketServerExample
    {
        public static int PortNumber = 6000;
        public static bool Running = false;
        public static Socket ServerSocket;


        // An interrupt handler for Ctrl-C presses
        public static void InterruptHandler(object sender, ConsoleCancelEventArgs args)
        {
            Console.WriteLine("Received SIGINT, shutting down server.");

            // Cleanup
            Running = false;
            ServerSocket.Shutdown(SocketShutdown.Both);
            ServerSocket.Close();
        }


        // Main method
        public static void Main(string[] args)
        {
            Socket clientSocket;
            byte[] msg = Encoding.ASCII.GetBytes("Hello, Client!\n");

            // Set the endpoint options
            IPEndPoint serv = new IPEndPoint(IPAddress.Any, PortNumber);

            ServerSocket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
            ServerSocket.Bind(serv);

            // Start listening for connections (queue of 5 max)
            ServerSocket.Listen(5);

            // Setup the Ctrl-C
            Console.CancelKeyPress += InterruptHandler;
            Running = true;
            Console.WriteLine("Running the TCP server.");

            // Main loop
            while (Running)
            {
                // Wait for a new client (blocks)
                clientSocket = ServerSocket.Accept();

                // Print some infor about the remote client
                Console.WriteLine("Incoming connection from {0}, replying.", clientSocket.RemoteEndPoint);

                // Send a reply (blocks)
                clientSocket.Send(msg, SocketFlags.None);

                // Close the connection
                clientSocket.Close();
            }
        }
    }
}
```

### Client

```c#
using System;
using System.Text;
using System.Net;
using System.Net.Sockets;

namespace Client
{
    class TcpSocketClientExample
    {
        public static int MaxReceiveLength = 255;
        public static int PortNumber = 6000;


        // Main method
        public static void Main(string[] args)
        {
            int len;
            byte[] buffer = new byte[MaxReceiveLength + 1];

            // Create a TCP/IP Socket
            Socket clientSocket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
            IPEndPoint serv = new IPEndPoint(IPAddress.Loopback, PortNumber);

            // Connect to the server
            Console.WriteLine("Connecting to the server...");
            clientSocket.Connect(serv);

            // Get a message (blocks)
            len = clientSocket.Receive(buffer);
            Console.Write("Got a message from the server[{0} bytes]:\n{1}",
                len, Encoding.ASCII.GetString(buffer, 0, len));

            // Cleanup
            clientSocket.Close();
        }
    }
}
```

### Why is using the Socket class directly more efficient than something like TcpClient or UdpClient?

就像我之前说的，类似于`TcpClient`和`UdpClient`这样的类有底层`Socket`对象（可以通过`Client`属性访问）。”如果`Socket`更复杂为何要使用它？“你可能会问

如果你选择使用`TcpClient`和`UdpClient`和直接使用`Socket`相比会有少量的开销。如果你真的需要一个高性能的app的话，使用`Socket`。性能/效率的好处只要来自于`Socket`中的异步方法，他们都是基于回调的，虽然`TcpClient`和`UdpClient`中也有异步方法。但是`Socket`中给了你更灵活的使用。

### Where To Go From Here

如果你想要学习更多，选择一个你想要制作的项目然后构建它。

下面两部分我的教程没有覆盖到

1. 异步Socket通信。这不是简单的使用`async`关键字来调用函数，这是一个完全不同的模式，下面是MSDN的一些文档

   [Using an Asynchronous Server Socket](https://msdn.microsoft.com/en-us/library/5w7b7x5f(v=vs.110).aspx)
   [Asynchronous Server Socket Example](https://msdn.microsoft.com/en-us/library/fx6588te(v=vs.110).aspx)
   [Using an Asynchronous Client Socket](https://msdn.microsoft.com/en-us/library/bbx2eya8(v=vs.110).aspx)
   [Asynchronous Client Socket Example](https://msdn.microsoft.com/en-us/library/bew39x2a(v=vs.110).aspx)

2. SSL/TLS和安全。在现代通信和计算中是非常重要的，有很多的事情可以做。 [Security in the .NET Framework](https://msdn.microsoft.com/en-us/library/fkytk30f.aspx)是一个好的开端，[SslStream](https://msdn.microsoft.com/en-us/library/system.net.security.sslstream(v=vs.110).aspx)也很值得研究。