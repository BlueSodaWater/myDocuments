## HttpClient

为了简单的开始，我们只需要做一个小的程序来下载网页并保存他，为此我们将会使用HttpClient类。

HttpClient是在TCP连接的基础上来从任何提供网页服务的服务器（或响应HTTP请求）获得响应。使用起来很简单，但是你需要异步编程。这会是一个稍微复杂一些的HttpClient示例，但是它展示了async，await，Task的简单使用方式。

我们的程序首先设置一个网页来获取，等服务器响应（一个 `200 HTTP Response`响应），将其数据转化为UTF-8编码格式的字节数组。最后将其保存到我们在顶部指定的文件中，在`Main()`方法中，在我们调用`DownloadWebPage()`，它将在代码内部开始执行，但一旦碰到`await`语句，执行返回外部，等待被等待的语句完成。但是由于在这里调用了`Thread.Sleep()`，它将暂停main函数的线程5秒。这根本不会停止下载。所以，如果你的网络连接良好，网页在5秒结束之前就应该下载完成了。如果没有，将会在结尾调用`GetAwaiter().GetResult()`阻塞执行，直到`Task`完成。

```c#
using System;
using System.IO;
using System.Text;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

namespace HttpClientExample
{
    class Downloader
    {
        // Where to download from, and where to save it to
        public static string urlToDownload = "https://16bpp.net/";
        public static string filename = "index.html";

        public static async Task DownloadWebPage()
        {
            Console.WriteLine("Starting download...");

            // Setup the HttpClient
            using (HttpClient httpClient = new HttpClient())
            {
                // Get the webpage asynchronously
                HttpResponseMessage resp = await httpClient.GetAsync(urlToDownload);

                // If we get a 200 response, then save it
                if (resp.IsSuccessStatusCode)
                {
                    Console.WriteLine("Got it...");

                    // Get the data
                    byte[] data = await resp.Content.ReadAsByteArrayAsync();

                    // Save it to a file
                    FileStream fStream = File.Create(filename);
                    await fStream.WriteAsync(data, 0, data.Length);
                    fStream.Close();

                    Console.WriteLine("Done!");
                }
            }
        }

        public static void Main (string[] args)
        {
            Task dlTask = DownloadWebPage();

            Console.WriteLine("Holding for at least 5 seconds...");
            Thread.Sleep(TimeSpan.FromSeconds(5));

            dlTask.GetAwaiter().GetResult();
        }
    }
}
```

你可能会注意到在`Main()`函数结尾有一个`dlTask.GetAwaiter().GetResult()`更长的调用。在那里我们可以直接调用`dlTask.Wait()`。这样做的原因是，如果`Task`抛出一个异常，我们将会得到抛出给我们的原始异常，而不是将其包装在`await()`方法产生的`AggregateException`中。

## HttpListener

简单的说，HttpListener是一个创建服务器的类。该服务器将监听并响应请求。我原本想要写一个TCP服务器可以手动处理HTTP消息，但是思考之后，这个类更容易处理。

为了我们简单的网站，我们需要做下面的事情：

1. 将网页传到浏览器上
2. 展示页面浏览数量（动态内容）
3. 让用户在页面上点击一个按钮可以关闭浏览器中的Http Server

像上一个例子一样，我们将会使用异步编程：

```c#
using System;
using System.IO;
using System.Text;
using System.Net;
using System.Threading.Tasks;

namespace HttpListenerExample
{
    class HttpServer
    {
        public static HttpListener listener;
        public static string url = "http://localhost:8000/";
        public static int pageViews = 0;
        public static int requestCount = 0;
        public static string pageData = 
            "<!DOCTYPE>" +
            "<html>" +
            "  <head>" +
            "    <title>HttpListener Example</title>" +
            "  </head>" +
            "  <body>" +
            "    <p>Page Views: {0}</p>" +
            "    <form method=\"post\" action=\"shutdown\">" +
            "      <input type=\"submit\" value=\"Shutdown\" {1}>" +
            "    </form>" +
            "  </body>" +
            "</html>";


        public static async Task HandleIncomingConnections()
        {
            bool runServer = true;

            // While a user hasn't visited the `shutdown` url, keep on handling requests
            while (runServer)
            {
                // Will wait here until we hear from a connection
                HttpListenerContext ctx = await listener.GetContextAsync();

                // Peel out the requests and response objects
                HttpListenerRequest req = ctx.Request;
                HttpListenerResponse resp = ctx.Response;

                // Print out some info about the request
                Console.WriteLine("Request #: {0}", ++requestCount);
                Console.WriteLine(req.Url.ToString());
                Console.WriteLine(req.HttpMethod);
                Console.WriteLine(req.UserHostName);
                Console.WriteLine(req.UserAgent);
                Console.WriteLine();

                // If `shutdown` url requested w/ POST, then shutdown the server after serving the page
                if ((req.HttpMethod == "POST") && (req.Url.AbsolutePath == "/shutdown"))
                {
                    Console.WriteLine("Shutdown requested");
                    runServer = false;
                }

                // Make sure we don't increment the page views counter if `favicon.ico` is requested
                if (req.Url.AbsolutePath != "/favicon.ico")
                    pageViews += 1;

                // Write the response info
                string disableSubmit = !runServer ? "disabled" : "";
                byte[] data = Encoding.UTF8.GetBytes(String.Format(pageData, pageViews, disableSubmit));
                resp.ContentType = "text/html";
                resp.ContentEncoding = Encoding.UTF8;
                resp.ContentLength64 = data.LongLength;

                // Write out to the response stream (asynchronously), then close it
                await resp.OutputStream.WriteAsync(data, 0, data.Length);
                resp.Close();
            }
        }


        public static void Main(string[] args)
        {
            // Create a Http server and start listening for incoming connections
            listener = new HttpListener();
            listener.Prefixes.Add(url);
            listener.Start();
            Console.WriteLine("Listening for connections on {0}", url);

            // Handle requests
            Task listenTask = HandleIncomingConnections();
            listenTask.GetAwaiter().GetResult();

            // Close the listener
            listener.Close();
        }
    }
}
```

当代码运行并且终端报告服务器正常的时候，直接访问[http://localhost:8000](http://localhost:8000/)，刷新几次页面，查看控制台输出，然后点击网页上的关闭按钮，然后尝试刷新。让我们来解释发生了什么。

当我们在我们的`HttpListener`实例中调用`Start()`，它将打开一个TCP套接字然后开始监听由我们添加的任何`Prefixes`发来的任何HTTP消息。在这种情况下，我们在根路径下监听`localhost`的`8000`端口。如果你只希望服务器仅仅响应`asdf`路径下的消息，你需要将`url`的值改为`http://localhost:8000/asdf/`。**请注意你所有的`prefixes`需要以/结尾，不然的话将会抛出`ArgumentException`异常。**

当我们调用`listener.GetContextAsync()` `HandleIncomingConnections()`方法将会等待直到有人向我们的服务器发生HTTP请求（它可以是一个网页浏览器，或者想上面所做的网页下载器）。一旦我们建立了连接，从`HttpListenerContext`我们可以剥离出必要的请求和响应对象。

- `HttpListenerRequest`告诉我们用户（客户端）想从我们（服务器）获得什么
- `HttpListenerResponse`让我们与用户通信

48-54行仅仅打印了一些请求的元数据。我们最关心的是他们发送给我们的`HttpMethod`和他们想要请求哪个路径。如果他们给我们在`/shutdown`路径下发送`POST`信息。这代表他们点击了关闭按钮想要我们关闭服务器。但如果不是，仅仅给他们一个标准的含有浏览计数的网页。我们也希望确认他们是否请求`favicon.ico`文件（大部分浏览器都会这么做），这样的话就无需加入计数。

在响应中，我们需要先在头部写一些信息：

1. 指定`MIME`类型，我们这里仅仅发送一个简单的HTML，因此我们使用`text/html`
2. 告诉响应消息使用的是什么编码类型。在这里我们使用UTF-8
3. 我们的消息字节长度是多少

一旦这些完成之后，我们从响应对象中获取`OutputStream`然后像其他流一样将其写入（异步的）。最后我们关闭`HttpListenerResponse`来告诉他我们对这条消息已经处理完成。这也会让我们关闭流。

这种情况可能会持续一段时间（比如刷新），但是一旦点击了关闭按钮，`HandleIncomingConnections()`将会传递最后一条消息给客户段然后退出。我们也需要清理所有我们使用的资源，调用`listener.Close()`来释放端口。

## TCP Chat

使用这些HTTP帮助类可能会覆盖很多基类。但我不认为每个人都想用它来通信。当你和互联网上某个人交流的时候，确保消息到达那里，并且是按顺序的，这就是基于TCP套接字的工作。

注意这不是TCP如何工作的教程，仅仅说明如何使用C#中的TCP APIs。如果你希望找这些教程，[可以从这边开始](http://ssfnet.org/Exchange/tcp/tcpTutorialNotes.html)。我同时也推荐可以看一些视频，你不需要知道TCP工作的每个细节，但是了解其底层原理是个好主意。

为了开始简单点，我们将会做一个简单的聊天程序。我不想要像其他教程一样使用标准的“echo server”。如果你想要看这些教程，[可以看这里](https://docs.microsoft.com/zh-cn/archive/blogs/jmanning/simple-tcpclient-echo-clientserver-example)。但我认为它不利于理解

在C#中使用TCP通信有两种方式：

1. 使用原始Socket对象
2. 使用TcpClient和TcpListener类

前者看起来和[Berkeley/POSIX](https://en.wikipedia.org/wiki/Berkeley_sockets)套接字API很像，而后者是一个使用TCP方便的包装套接字对象（可以从名字看出来）。我想要从简单开始我们将会使用`TcpClient`和`TcpListener`来开始我们的程序。不要担心，如果你对`Socket`感兴趣，后面会讲到。

我还想强调的是在这个项目中，我们将会使用同步，单线程的代码。在现实的应用中，当有网络操作的时候，你应该使用异步，多线程的代码。我现在只想要将事件简单化，在此之后，我将会给你们一个多线程的例子。`TcpClient`和`TcpListener`的`async`方法应该很容易使用，如果你知道如何异步编程和知道这两个类非同步的方法的话。

### Design

我们将把TCP聊天程序分为三块。一个服务器（用来分发消息），一个发消息端（用来发送消息），一个浏览端（用来接收消息）。你可能会疑问我为什么要分为三个部分。因为我不想要处理任何GUI代码。由于每一个部分将会自己独立执行，请确保这三个项目都加入到你的解决方案中。我们将会依次完成服务器，发消息端，浏览端

### 应用协议

对于那些了解[OSI模型](https://en.wikipedia.org/wiki/OSI_model)的人，我们现在将定义我们的应用层协议。如果你不知道它是什么，这是我们传输数据和解释数据的方式。注意到我们有两个不同的客户端（发消息端和浏览端），因此服务器需要区别对待他们。我们将会只使用文本（没有字节数据），并且是UTF-8的编码格式。

浏览端：

- 在浏览端和服务器建立一个连接之后，将会发送一个消息：viewer。服务器就会把它认作浏览端
- 服务器端仅仅给每个浏览端发送文本信息。浏览端只会展示收到的信息。

发消息端：

- 在发消息端和服务器建立一个连接之后，他将会送一个信息：`name:MY_NAME`。MY_NAME是发消息端想要使用的名称（如`name:Kevin Bacon`）。如果这个名字已经被其他发消息端取了，服务器端将会发送消息`name taken`然后断开连接。
- 一但发消息端成功连接。服务器端只管监听他们发送了什么消息，然后分发给浏览端。

这确实不是功能最丰富的协议设计（也不是最好的），但是是一个很简单的例子，如果你需要一些帮助来理解流程，可以看下面的图。

![TCP Chat Server diagram](https://storage.googleapis.com/sixteenbpp/tutorials/images/tcp-chat-server-diagram.png)

### 开始前的提示

这个比其他的例子代码多出很多，但是三个文件中也有很多重复的。不要灰心，如果需要的话可以休息一下。服务器端和浏览端有“Ctrl-C”功能，供参考。很多行代码都有注释，告诉你是否调用阻塞。此外，大部分代码没有try-catch块，这些在生产环境中应该存在。

当一个可能断开连接的时候，我们不用给客户端或者服务器端发送消息。当某一方想要断开连接的时候，我们只需要关闭`TcpClient` and `NetworkStream`。他们之间仍会发送 FIN & ACK TCP 包。我们在本地可以使用这些代码来检查这些包是否发送。

### Tcp Chat - Server

我看过很多网络编程教程都是从客户端代码开始的，我认为先从服务器端代码开始好点。这里代码有很多。

主要用到的两个类是`TcpClient`和`TcpListener`。简而言之，我们使用`TcpListener`来监听即将到来的连接，一旦有人连接，我们将会创建一个新的`TcpClient`来与远程进程通信.

下面是服务器的代码

```c#
using System;
using System.Text;
using System.Collections.Generic;
using System.Net;
using System.Net.Sockets;
using System.Threading;

namespace TcpChatServer
{
    class TcpChatServer
    {
        // What listens in
        private TcpListener _listener;

        // types of clients connected
        private List<TcpClient> _viewers = new List<TcpClient>();
        private List<TcpClient> _messengers = new List<TcpClient>();

        // Names that are taken by other messengers
        private Dictionary<TcpClient, string> _names = new Dictionary<TcpClient, string>();

        // Messages that need to be sent
        private Queue<string> _messageQueue = new Queue<string>();

        // Extra fun data
        public readonly string ChatName;
        public readonly int Port;
        public bool Running { get; private set; }

        // Buffer
        public readonly int BufferSize = 2 * 1024;  // 2KB

        // Make a new TCP chat server, with our provided name
        public TcpChatServer(string chatName, int port)
        {
            // Set the basic data
            ChatName = chatName;
            Port = port;
            Running = false;

            // Make the listener listen for connections on any network device
            _listener = new TcpListener(IPAddress.Any, Port);
        }

        // If the server is running, this will shut down the server
        public void Shutdown()
        {
            Running = false;
            Console.WriteLine("Shutting down server");
        }

        // Start running the server.  Will stop when `Shutdown()` has been called
        public void Run()
        {
            // Some info
            Console.WriteLine("Starting the \"{0}\" TCP Chat Server on port {1}.", ChatName, Port);
            Console.WriteLine("Press Ctrl-C to shut down the server at any time.");

            // Make the server run
            _listener.Start();           // No backlog
            Running = true;

            // Main server loop
            while (Running)
            {
                // Check for new clients
                if (_listener.Pending())
                    _handleNewConnection();

                // Do the rest
                _checkForDisconnects();
                _checkForNewMessages();
                _sendMessages();

                // Use less CPU
                Thread.Sleep(10);
            }

            // Stop the server, and clean up any connected clients
            foreach (TcpClient v in _viewers)
                _cleanupClient(v);
            foreach (TcpClient m in _messengers)
                _cleanupClient(m);
            _listener.Stop();

            // Some info
            Console.WriteLine("Server is shut down.");
        }
            
        private void _handleNewConnection()
        {
            // There is (at least) one, see what they want
            bool good = false;
            TcpClient newClient = _listener.AcceptTcpClient();      // Blocks
            NetworkStream netStream = newClient.GetStream();

            // Modify the default buffer sizes
            newClient.SendBufferSize = BufferSize;
            newClient.ReceiveBufferSize = BufferSize;

            // Print some info
            EndPoint endPoint = newClient.Client.RemoteEndPoint;
            Console.WriteLine("Handling a new client from {0}...", endPoint);

            // Let them identify themselves
            byte[] msgBuffer = new byte[BufferSize];
            int bytesRead = netStream.Read(msgBuffer, 0, msgBuffer.Length);     // Blocks
            //Console.WriteLine("Got {0} bytes.", bytesRead);
            if (bytesRead > 0)
            {
                string msg = Encoding.UTF8.GetString(msgBuffer, 0, bytesRead);

                if (msg == "viewer")
                {
                    // They just want to watch
                    good = true;
                    _viewers.Add(newClient);

                    Console.WriteLine("{0} is a Viewer.", endPoint);

                    // Send them a "hello message"
                    msg = String.Format("Welcome to the \"{0}\" Chat Server!", ChatName);
                    msgBuffer = Encoding.UTF8.GetBytes(msg);
                    netStream.Write(msgBuffer, 0, msgBuffer.Length);    // Blocks
                }
                else if (msg.StartsWith("name:"))
                {
                    // Okay, so they might be a messenger
                    string name = msg.Substring(msg.IndexOf(':') + 1);

                    if ((name != string.Empty) && (!_names.ContainsValue(name)))
                    {
                        // They're new here, add them in
                        good = true;
                        _names.Add(newClient, name);
                        _messengers.Add(newClient);

                        Console.WriteLine("{0} is a Messenger with the name {1}.", endPoint, name);

                        // Tell the viewers we have a new messenger
                        _messageQueue.Enqueue(String.Format("{0} has joined the chat.", name));
                    }
                }
                else
                {
                    // Wasn't either a viewer or messenger, clean up anyways.
                    Console.WriteLine("Wasn't able to identify {0} as a Viewer or Messenger.", endPoint);
                    _cleanupClient(newClient);
                }
            }

            // Do we really want them?
            if (!good)
                newClient.Close();
        }

        // Sees if any of the clients have left the chat server
        private void _checkForDisconnects()
        {
            // Check the viewers first
            foreach (TcpClient v in _viewers.ToArray())
            {
                if (_isDisconnected(v))
                {
                    Console.WriteLine("Viewer {0} has left.", v.Client.RemoteEndPoint);

                    // cleanup on our end
                    _viewers.Remove(v);     // Remove from list
                    _cleanupClient(v);
                }
            }

            // Check the messengers second
            foreach (TcpClient m in _messengers.ToArray())
            {
                if (_isDisconnected(m))
                {
                    // Get info about the messenger
                    string name = _names[m];

                    // Tell the viewers someone has left
                    Console.WriteLine("Messeger {0} has left.", name);
                    _messageQueue.Enqueue(String.Format("{0} has left the chat", name));

                    // clean up on our end 
                    _messengers.Remove(m);  // Remove from list
                    _names.Remove(m);       // Remove taken name
                    _cleanupClient(m);
                }
            }
        }

        // See if any of our messengers have sent us a new message, put it in the queue
        private void _checkForNewMessages()
        {
            foreach (TcpClient m in _messengers)
            {
                int messageLength = m.Available;
                if (messageLength > 0)
                {
                    // there is one!  get it
                    byte[] msgBuffer = new byte[messageLength];
                    m.GetStream().Read(msgBuffer, 0, msgBuffer.Length);     // Blocks

                    // Attach a name to it and shove it into the queue
                    string msg = String.Format("{0}: {1}", _names[m], Encoding.UTF8.GetString(msgBuffer));
                    _messageQueue.Enqueue(msg);
                }
            }
        }

        // Clears out the message queue (and sends it to all of the viewers
        private void _sendMessages()
        {
            foreach (string msg in _messageQueue)
            {
                // Encode the message
                byte[] msgBuffer = Encoding.UTF8.GetBytes(msg);

                // Send the message to each viewer
                foreach (TcpClient v in _viewers)
                    v.GetStream().Write(msgBuffer, 0, msgBuffer.Length);    // Blocks
            }

            // clear out the queue
            _messageQueue.Clear();
        }

        // Checks if a socket has disconnected
        // Adapted from -- http://stackoverflow.com/questions/722240/instantly-detect-client-disconnection-from-server-socket
        private static bool _isDisconnected(TcpClient client)
        {
            try
            {
                Socket s = client.Client;
                return s.Poll(10 * 1000, SelectMode.SelectRead) && (s.Available == 0);
            }
            catch(SocketException se)
            {
                // We got a socket error, assume it's disconnected
                return true;
            }
        }

        // cleans up resources for a TcpClient
        private static void _cleanupClient(TcpClient client)
        {
            client.GetStream().Close();     // Close network stream
            client.Close();                 // Close client
        }





        public static TcpChatServer chat;

        protected static void InterruptHandler(object sender, ConsoleCancelEventArgs args)
        {
            chat.Shutdown();
            args.Cancel = true;
        }
            
        public static void Main(string[] args)
        {
            // Create the server
            string name = "Bad IRC";//args[0].Trim();
            int port = 6000;//int.Parse(args[1].Trim());
            chat = new TcpChatServer(name, port);

            // Add a handler for a Ctrl-C press
            Console.CancelKeyPress += InterruptHandler;

            // run the chat server
            chat.Run();
        }
    }
}
```

当我们实例化`TcpChatServer`的时候，将会创建一个监听器，但是它不会立刻开始监听，直到我们在`Run()`方法中调用`Start()`。`IPAdress.Any`代表允许来自任何网络的任何人连接我们。

`Run()`方法是程序的核心。就像上面说的，当我们调用 `_listener.Start()`，`TcpListener`会开始监听我们提供给它的端口的新传入的TCP连接（这里是6000）。如果有一个新的待处理连接（通过`Pending()`方法），我们跳到检查新客户端是谁的函数中。

`HandleNewConnection()`所做的事情正如其名，在连接稳定之后，我们通过调用监听器中`AcceptTcpClient`来获取`TcpClient`实例。客户端应该发送一条`viewer`或者`name:MY_NAME`的消息

- 如果我们有新的浏览端，将其标识为浏览端，然后给他们发送一个欢迎消息
- 如果我们有新的发消息端，确认`MY_NAME`是否已经被取了。
  - 如果没有，给浏览端发消息有一个新的发消息端加入了我们
  - 否则，关闭连接
- 其他的，无法辨识身份的，关闭连接。

回到`Run()`，我们检测断开连接，新消息，然后在循环中将其排队发送，还调用了`Thread.Sleep(10)`，这样我们可以节省CPU资源。

`CheckForDisconnects()`是一个两次循环遍历所有的客户端然后调用`IsDisConnected()`方法来测试 是否有发送FIN/ACK包。

<table><tr><td bgcolor=#C0FF3E>TcpClient的`Connected`属性在这里看来比使用`IsDisConnected()`正确，但是它存在问题。`Connected`只有在最后的IP操作成功的情况下才会返回`true`。这代表FIN/ACK可能已经被发送，但是由于我们结尾没有IO操作，`Connected`仍然返回`true`，这一点需要当心。</td></tr></table>

`CheckForNewMessage()`也是一个循环检查所有的消息，我们可以使用`Available`属性来看消息有多大（字节）然后将他们从客户端的`NetworkStream`读出来。

`SendMessages()`边写浏览端的流边清理`_messageQueue`。

还有一个`CleanUppClient()`函数。是帮助关闭`TcpClient`的`NetworkStream`和它自己。

<table><tr><td bgcolor=#D1EEEE>你可能会好奇为什么我们需要手动关闭流，因为流是由底层的Socket对象创建的（可以通过 `TcpClient`'s的`Client`属性访问），这代表你必须手动清理它。[Read more about it here.](https://msdn.microsoft.com/en-us/library/system.net.sockets.tcpclient.getstream.aspx)</td></tr></table>

### TCP Chat - Messenger

发消息端比服务器内容要少。连接，查看名字是否合法，然后让用户输入信息直到他们退出。

```c#
using System;
using System.Text;
using System.Net;
using System.Net.Sockets;
using System.Threading;

namespace TcpChatMessenger
{
    class TcpChatMessenger
    {
        // Connection objects
        public readonly string ServerAddress;
        public readonly int Port;
        private TcpClient _client;
        public bool Running { get; private set; }

        // Buffer & messaging
        public readonly int BufferSize = 2 * 1024;  // 2KB
        private NetworkStream _msgStream = null;

        // Personal data
        public readonly string Name;

        public TcpChatMessenger(string serverAddress, int port, string name)
        {
            // Create a non-connected TcpClient
            _client = new TcpClient();          // Other constructors will start a connection
            _client.SendBufferSize = BufferSize;
            _client.ReceiveBufferSize = BufferSize;
            Running = false;

            // Set the other things
            ServerAddress = serverAddress;
            Port = port;
            Name = name;
        }

        public void Connect()
        {
            // Try to connect
            _client.Connect(ServerAddress, Port);       // Will resolve DNS for us; blocks
            EndPoint endPoint = _client.Client.RemoteEndPoint;

            // Make sure we're connected
            if (_client.Connected)
            {
                // Got in!
                Console.WriteLine("Connected to the server at {0}.", endPoint);

                // Tell them that we're a messenger
                _msgStream = _client.GetStream();
                byte[] msgBuffer = Encoding.UTF8.GetBytes(String.Format("name:{0}", Name));
                _msgStream.Write(msgBuffer, 0, msgBuffer.Length);   // Blocks

                // If we're still connected after sending our name, that means the server accepts us
                if (!_isDisconnected(_client))
                    Running = true;
                else
                {
                    // Name was probably taken...
                    _cleanupNetworkResources();
                    Console.WriteLine("The server rejected us; \"{0}\" is probably in use.", Name);
                }
            }
            else
            {
                _cleanupNetworkResources();
                Console.WriteLine("Wasn't able to connect to the server at {0}.", endPoint);
            }
        }

        public void SendMessages()
        {
            bool wasRunning = Running;

            while (Running)
            {
                // Poll for user input
                Console.Write("{0}> ", Name);
                string msg = Console.ReadLine();

                // Quit or send a message
                if ((msg.ToLower() == "quit") || (msg.ToLower() == "exit"))
                {
                    // User wants to quit
                    Console.WriteLine("Disconnecting...");
                    Running = false;
                }
                else if (msg != string.Empty)
                {
                    // Send the message
                    byte[] msgBuffer = Encoding.UTF8.GetBytes(msg);
                    _msgStream.Write(msgBuffer, 0, msgBuffer.Length);   // Blocks
                }

                // Use less CPU
                Thread.Sleep(10);

                // Check the server didn't disconnect us
                if (_isDisconnected(_client))
                {
                    Running = false;
                    Console.WriteLine("Server has disconnected from us.\n:[");
                }
            }

            _cleanupNetworkResources();
            if (wasRunning)
                Console.WriteLine("Disconnected.");
        }

        // Cleans any leftover network resources
        private void _cleanupNetworkResources()
        {
            _msgStream?.Close();
            _msgStream = null;
            _client.Close();
        }

        // Checks if a socket has disconnected
        // Adapted from -- http://stackoverflow.com/questions/722240/instantly-detect-client-disconnection-from-server-socket
        private static bool _isDisconnected(TcpClient client)
        {
            try
            {
                Socket s = client.Client;
                return s.Poll(10 * 1000, SelectMode.SelectRead) && (s.Available == 0);
            }
            catch(SocketException se)
            {
                // We got a socket error, assume it's disconnected
                return true;
            }
        }





        public static void Main(string[] args)
        {
            // Get a name
            Console.Write("Enter a name to use: ");
            string name = Console.ReadLine();

            // Setup the Messenger
            string host = "localhost";//args[0].Trim();
            int port = 6000;//int.Parse(args[1].Trim());
            TcpChatMessenger messenger = new TcpChatMessenger(host, port, name);

            // connect and send messages
            messenger.Connect();
            messenger.SendMessages();
        }
    }
}
```

`TcpClient`有很多不同的构造函数。他们所有的，除了一个没有参数的，当他们被实例化的时候都会尝试连接到提供给的地址和端口。我们使用没有参数的那个

我们的`Connect()`方法会尝试初始化连接。如果我们连接成功，它将会发送`name:MY_NAME`消息，`MY_NAME`是在运行的时候用户提供的，如果我们在这之后连接仍然存在，我们就准备好发送消息了。如果用户发送了空字符串作为名称，服务器将会引导他们。

`SendMessages()`是我们程序的主循环。如果我们连接上（通过检查`Running`属性），等待用户输入。

- 如果输入是`quit`或者`exit`，将会和服务器断开连接
- 如果消息不为空，将其发送给服务器

在这之后，每隔10毫秒(就像服务器一样)一次。一旦休息结束，通过调用`IsDisconnected()`确保我们仍然和我们的`TcpClient`保持连接

`CleanUpNetworkResources()`用来确保我们使用的`NetworkStream`和`TcpClient`已经关闭。

### TCP Chat - Viewer

这个也和发消息端一样小巧，事实上，大部分代码都很小巧。一旦和服务器连接数，将会得到“hello”消息，然后持续打印消息直到按了Ctrl-C。

```c#
using System;
using System.Text;
using System.Net;
using System.Net.Sockets;
using System.Threading;

namespace TcpChatViewer
{
    class TcpChatViewer
    {
        // Connection objects
        public readonly string ServerAddress;
        public readonly int Port;
        private TcpClient _client;
        public bool Running { get; private set; }
        private bool _disconnectRequested = false;

        // Buffer & messaging
        public readonly int BufferSize = 2 * 1024;  // 2KB
        private NetworkStream _msgStream = null;

        public TcpChatViewer(string serverAddress, int port)
        {
            // Create a non-connected TcpClient
            _client = new TcpClient();          // Other constructors will start a connection
            _client.SendBufferSize = BufferSize;
            _client.ReceiveBufferSize = BufferSize;
            Running = false;

            // Set the other things
            ServerAddress = serverAddress;
            Port = port;
        }

        // connects to the chat server
        public void Connect()
        {
            // Now try to connect
            _client.Connect(ServerAddress, Port);   // Will resolve DNS for us; blocks
            EndPoint endPoint = _client.Client.RemoteEndPoint;

            // check that we're connected
            if (_client.Connected)
            {
                // got in!
                Console.WriteLine("Connected to the server at {0}.", endPoint);

                // Send them the message that we're a viewer
                _msgStream = _client.GetStream();
                byte[] msgBuffer = Encoding.UTF8.GetBytes("viewer");
                _msgStream.Write(msgBuffer, 0, msgBuffer.Length);     // Blocks

                // check that we're still connected, if the server has not kicked us, then we're in!
                if (!_isDisconnected(_client))
                {
                    Running = true;
                    Console.WriteLine("Press Ctrl-C to exit the Viewer at any time.");
                }
                else
                {
                    // Server doens't see us as a viewer, cleanup
                    _cleanupNetworkResources();
                    Console.WriteLine("The server didn't recognise us as a Viewer.\n:[");
                }
            }
            else
            {
                _cleanupNetworkResources();
                Console.WriteLine("Wasn't able to connect to the server at {0}.", endPoint);
            }
        }

        // Requests a disconnect
        public void Disconnect()
        {
            Running = false;
            _disconnectRequested = true;
            Console.WriteLine("Disconnecting from the chat...");
        }

        // Main loop, listens and prints messages from the server
        public void ListenForMessages()
        {
            bool wasRunning = Running;

            // Listen for messages
            while (Running)
            {
                // Do we have a new message?
                int messageLength = _client.Available;
                if (messageLength > 0)
                {
                    //Console.WriteLine("New incoming message of {0} bytes", messageLength);

                    // Read the whole message
                    byte[] msgBuffer = new byte[messageLength];
                    _msgStream.Read(msgBuffer, 0, messageLength);   // Blocks

                    // An alternative way of reading
                    //int bytesRead = 0;
                    //while (bytesRead < messageLength)
                    //{
                    //    bytesRead += _msgStream.Read(_msgBuffer,
                    //                                 bytesRead,
                    //                                 _msgBuffer.Length - bytesRead);
                    //    Thread.Sleep(1);    // Use less CPU
                    //}

                    // Decode it and print it
                    string msg = Encoding.UTF8.GetString(msgBuffer);
                    Console.WriteLine(msg);
                }

                // Use less CPU
                Thread.Sleep(10);

                // Check the server didn't disconnect us
                if (_isDisconnected(_client))
                {
                    Running = false;
                    Console.WriteLine("Server has disconnected from us.\n:[");
                }

                // Check that a cancel has been requested by the user
                Running &= !_disconnectRequested;
            }

            // Cleanup
            _cleanupNetworkResources();
            if (wasRunning)
                Console.WriteLine("Disconnected.");
        }

        // Cleans any leftover network resources
        private void _cleanupNetworkResources()
        {
            _msgStream?.Close();
            _msgStream = null;
            _client.Close();
        }

        // Checks if a socket has disconnected
        // Adapted from -- http://stackoverflow.com/questions/722240/instantly-detect-client-disconnection-from-server-socket
        private static bool _isDisconnected(TcpClient client)
        {
            try
            {
                Socket s = client.Client;
                return s.Poll(10 * 1000, SelectMode.SelectRead) && (s.Available == 0);
            }
            catch(SocketException se)
            {
                // We got a socket error, assume it's disconnected
                return true;
            }
        }





        public static TcpChatViewer viewer;

        protected static void InterruptHandler(object sender, ConsoleCancelEventArgs args)
        {
            viewer.Disconnect();
            args.Cancel = true;
        }

        public static void Main(string[] args)
        {
            // Setup the Viewer
            string host = "localhost";//args[0].Trim();
            int port = 6000;//int.Parse(args[1].Trim());
            viewer = new TcpChatViewer(host, port);

            // Add a handler for a Ctrl-C press
            Console.CancelKeyPress += InterruptHandler;

            // Try to connect & view messages
            viewer.Connect();
            viewer.ListenForMessages();
        }
    }
}

如同发消息端一样，我们时候TcpClient的构造函数但不连接。我们在`Connect()`函数中连接。如果我们已经连接上了，我们立刻发送`viewer`的消息告诉服务器我们是谁，一个浏览端。如果我们仍然连接，我们就可以认定服务器把我们认作浏览端并且将会给我们发送消息。在这一点上，服务器已经给我们发送一个"Welcome to the 'SERVER_NAME' Chat Server" 但直到我们进入`ListenForMessages()`才会知道。

如果我们连接上了，我们测试是否有新的消息到来，如果是的，`TcpClient.Available`将会报告它有多少字节大小。如果这样，我们会尝试从`NetworkStream`读取它。调用`Read()`方法会阻塞直到读取大量字节。一旦接收到了消息，他将会打印出来。就像其他程序一样，我们将会休眠10毫秒来节省CPU资源。然后我们将会检查FIN/ACK包和检查是或否用户按了Ctrl-C。

一旦推出了`while(Running)`循环，我们会清理所有剩下的网络资源。

### TCP Chat - Recap

说了这么多，做了这么多，打了这么多字，还有一些事情需要做。这不是互联网聊天程序的良好实现。记住，我的目标是教你如何使用后C#`System.Net`中的Api。让我们看看有哪些是十分错误的。

1. 程序的协议的定义十分糟糕
   1. 当一方想要从另一方断开连接时，应该有一条消息。我这里说的不是FIN/ACK包。我说的是真实的`byte`应该被客户端服务器发送，以告诉他们对方将断开连接。当然FIN/ACK包在`byte`被发送后仍然会被发生，但是这是一种更加优雅，更具有描述性的方法，很多程序（像网络游戏）都使用这种方式来判断某人时候正常的退出应用程序
   2. 应该有心跳消息。意思是每隔一段时间（比如30秒）客户端/服务器互相发送消息“我仍然在连接”。这些通常是非常小的消息，只有一些例如心跳时间之类的东西，但是我看到有人在其中加了一些元数据，ping也可以兼做这些。
   3. 服务器端应该向客户端发送错误消息，例如，如果发消息端使用了一个之前已经用过的名字，服务器端在关闭连接前应该通知他“错误，名字已经被取了”。
2. 看`TcpChatServer.cs`中的`HandleNewConnection()`函数，其中有些非常糟糕的东西。
   1. 如果一个客户端连接了，但是没有发送任何消息，服务器只会保持他。我们在代码中没有设置任何超时，所以默认情况下，流会一直尝试读取很长很长的事件。
   2. 有人通过telnet进入了聊天服务器，但是不发送消息来标识自己，会使得服务器被劫持。
3. 当从网络中读取消息的时候我们应该设置超时，现在`NetworkStream`在无限等待数据，可以查看[NetworkStream](https://msdn.microsoft.com/en-us/library/system.net.sockets.networkstream(v=vs.110).aspx)的超时相关属性。
4. 将用户端拆分为两个（浏览端和发消息端）不是一个好的做法。对于那些只想要收听的人来说是很好的，但是对于那些想要发送和接受消息的人来说是一个麻烦。
5. 任何现实中的网络应用程序应该使用`async`，多线程，超时机制，这里一个都没有。正常来说，当一个新的客户端连接进来的时候，服务器会开辟一个新的线程或者进程，这样的话，既可以处理新的连接也可以处理现有连接的客户端，我们在读和写`NetworkStream`应该也使用异步。

