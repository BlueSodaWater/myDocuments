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

