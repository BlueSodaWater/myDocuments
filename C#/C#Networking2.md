## TCP Games

当然，我们已经完成了一个线程的网络应用。下一步应该在做什么？当然是多线程当然让我们多走几步用一些`async`代码。这个比我想象的复杂得多，但是请原谅我，因为它可以做得更多。挺住，这会很有意思。

请注意，这个章节没有任何`async Socketing`的编程，这将会会完全不同，会在最后一个章节里面说。

我发现游戏是一个教编程概念，语言和类库很好的方式。但是因为我想要保持教程“更少的GUI”。我们

会制作一些基于文本（控制台）的游戏。

<table><tr><td bgcolor=#C0FF3E>顺便提一下，如果你想要制作一个真正的游戏，在大多数情况下使用UDP而不是TCP更好一些。在这里TCP更好一点，因为我们只做基于文本的游戏，在服务器上的游戏都应该基于回合制的。如果你的网络游戏是实时运行，UDP是一个更好的选择。</td></tr></table>

### Before Starting

如果你没有阅读之前的章节TCP Chat，你应该自己去熟悉[TcpClient](https://docs.microsoft.com/en-us/dotnet/api/system.net.sockets.tcpclient?redirectedfrom=MSDN&view=net-5.0)和[TcpListener](https://docs.microsoft.com/en-us/dotnet/api/system.net.sockets.tcplistener?redirectedfrom=MSDN&view=net-5.0)。服务端的部分设计是为了能让我们切换不同的游戏。因为时间的原因，我只制作了一款《猜我数字》的单人游戏。我也想要制作《井字棋》（多人游戏），但这个留给你们做练习，这个应该不难。

在这个章节我们的解决方案里面应该只有两个分开的程序。一个是`TcpGamesServer`和`TcpGamesClient`，我们的命名空间都用`TcpGames`。

我们在此也用`IsDisconnected()`方法来检查断开的连接，注意，服务器中的代码名称略有不同，下面是代码：

```c#
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
```

### TCP Games - Application Protocol

让我们首先讨论我们的客户端和服务端如何传递消息/包。为了比上次更有条理，我们将会使用JSON，只包含文本，每一个JSON对象应该都有两个字段，`command`和`message`。

- `command`：告诉客户端/服务器包应该如何处理
- `message`：给`command`增加了一些额外的内容。有些时候可能是空字符串。

每个包我们应该是由三个简单的`command`：

1. `bye`：客户端和服务器都可以发送，它通知另一方正在断开连接
   - 如果是服务器发给客户端，客户端应该展示包含的消息然后清除其网络资源
   - 如果是客户端发给服务器，服务器应该选择如何处理`message`（如果有的话）。他也应该清除和这个客户端相关的网络资源。
2. `message`：注意这个是一个`command`。他只负责发送纯文本消息
   - 如果是服务器发给客户端，客户端应该将其`message`字段中的内容打印出来
   - 如果是客户端发给服务器，服务器可以选择忽略他。无论如何，客户端不应该发送`message`包。
3. `input`：需要一些输入
   - 如果是服务器发给客户端，客户端应该请求用户输入
     - 消息字段的内容应该被视为对用户的提示
   - 如果是客户端发给服务器，这应该只是为了响应输入请求
     - 如果他不需要得到请求，服务器就应该忽略这个包

如果你有些困惑，下面这张图解释了我们的程序如何操作：

![img](https://storage.googleapis.com/sixteenbpp/tutorials/images/tcp-games-packet-diagram.png)

### Packet class

这是我们Packet类，一定要将他添加到解决方案中的两个项目中。我们这么做的原因是，可使用C#的方式访问数据。我们将使用 [Newtonsoft's Json.NET](http://www.newtonsoft.com/json) 类来解析JSON。

```c#
using Newtonsoft.Json;

namespace TcpGames
{
    public class Packet
    {
        [JsonProperty("command")]
        public string Command { get; set; }

        [JsonProperty("message")]
        public string Message { get; set; }

        // Makes a packet
        public Packet(string command="", string message="")
        {
            Command = command;
            Message = message;
        }

        public override string ToString()
        {
            return string.Format(
                "[Packet:\n" +
                "  Command=`{0}`\n" +
                "  Message=`{1}`]",
                Command, Message);
        }

        // Serialize to Json
        public string ToJson()
        {
            return JsonConvert.SerializeObject(this);
        }

        // Deserialize
        public static Packet FromJson(string jsonData)
        {
            return JsonConvert.DeserializeObject<Packet>(jsonData);
        }
    }
}

```

### 他将如何通过网络发送

在我们把数据塞进互联网中时，要先做一些预处理：

1. 调用`Packet`中的`ToJson()`方法来把它转化为字符串。然后将其编码（UTF-8格式）为一个字节数组
2. 获取字节数组中有多少字节，然后把这个数字编码为**16位无符号整数**再将其放进一个字节数组。得到的数组长度应该正好是**两个字节**。
3. 在“JSON 缓冲区”之前，使用“length缓冲区”将这两个数组拼接到一起。

最后的字节数组就是我们要发送到网上的数组。这样无论是谁获得了`Packet`都可以知道他的长度。

<table><tr><td bgcolor=#D1EEEE>你可能会好奇为什么我们需要手动关闭流，因为流是由底层的Socket对象创你可能会好奇为什么要这么多。因为`TcpClient.Available`只会报告有多少字节已经获得但还没有读取，也仅仅如此。但是有可能在我们的程序有机会从`NetworkStream`读取之前，已经获得了两个``Packet`。这就提出了一个问题，即如何将数据拆分到两个单独的数据包中。
<br/>
由于我们使用的是JSON，我们可以匹配第一级花括号。相反，在实际消息之前填充“真实消息”的长度，是一个常用的技术，我们最好在此实践它。</td></tr></table>

我选择使用16位无符号整数，因为它可以允许我们的JSON字符串最大为64KB。对于基于文本的游戏来说，这一句是非常充足的数据大小了。我们也可以只使用一个8位的无符号整数，但这样字符串最大只能为255字节，我们所发送的消息必须非常小。

### TCP Games - Async/Multithreaded Server

这个源代码是整个两个项目中最大的一部分。它到处都有异步调用，这里并没有太复杂的多线程

#### The Game Interface

这是”游戏“服务器，所以我们想支持基于多文本的游戏。（尽管在本教程系列的一部分中我只实现了一个）。下面是服务器上游戏应该继承的接口

```c#
using System.Net.Sockets;

namespace TcpGames
{
    interface IGame
    {
        #region Properties
        // Name of the game
        string Name { get; }

        // How many players are needed to start
        int RequiredPlayers { get; }
        #endregion // Properties

        #region Functions
        // Adds a player to the game (should be before it starts)
        bool AddPlayer(TcpClient player);

        // Tells the server to disconnect a player
        void DisconnectClient(TcpClient client);

        // The main game loop
        void Run();
        #endregion // Functions
    }
}
```

`Name`是游戏的元数据。`RequiredPlayers`来告诉服务器游戏开始前需要多少玩家。`AddPlayer()`函数被服务器用来给游戏提供一个玩家。如果玩家添加成功应该返回`true`。服务器应该在游戏开始前添加玩家。`DisconnectClient()`用来告知游戏，如果服务器检测到有客户端断开连接。有了或者消息，游戏可能选择早点结束。最后的函数`Run()`是游戏的主循环，这个方法会在它自己的线程中运行。`Run()`也负责解决处理客户端发来的包，检测是否可能断开连接。

#### The Server

这里的代码量有点大，最后有解释。你可能发现提到了`GuessMyNumberGames`类，我们将在下一节加上它，所以假设他在这里已经存在了。

```c#
using System;
using System.Collections.Generic;
using System.Text;
using System.Net;
using System.Net.Sockets;
using System.Threading;
using System.Threading.Tasks;

namespace TcpGames
{
    public class TcpGamesServer
    {
        // Listens for new incoming connections
        private TcpListener _listener;

        // Clients objects
        private List<TcpClient> _clients = new List<TcpClient>();
        private List<TcpClient> _waitingLobby = new List<TcpClient>();

        // Game stuff
        private Dictionary<TcpClient, IGame> _gameClientIsIn = new Dictionary<TcpClient, IGame>();
        private List<IGame> _games = new List<IGame>();
        private List<Thread> _gameThreads = new List<Thread>();
        private IGame _nextGame;

        // Other data
        public readonly string Name;
        public readonly int Port;
        public bool Running { get; private set; }

        // Construct to create a new Games Server
        public TcpGamesServer(string name, int port)
        {
            // Set some of the basic data
            Name = name;
            Port = port;
            Running = false;

            // Create the listener
            _listener = new TcpListener(IPAddress.Any, Port);
        }

        // Shutsdown the server if its running
        public void Shutdown()
        {
            if (Running)
            {
                Running = false;
                Console.WriteLine("Shutting down the Game(s) Server...");
            }
        }

        // The main loop for the games server
        public void Run()
        {
            Console.WriteLine("Starting the \"{0}\" Game(s) Server on port {1}.", Name, Port); 
            Console.WriteLine("Press Ctrl-C to shutdown the server at any time.");

            // Start the next game
            // (current only the Guess My Number Game)
            _nextGame = new GuessMyNumberGame(this);

            // Start running the server
            _listener.Start();
            Running = true;
            List<Task> newConnectionTasks = new List<Task>();
            Console.WriteLine("Waiting for incommming connections...");

            while (Running)
            {
                // Handle any new clients
                if (_listener.Pending())
                    newConnectionTasks.Add(_handleNewConnection());

                // Once we have enough clients for the next game, add them in and start the game
                if (_waitingLobby.Count >= _nextGame.RequiredPlayers)
                {
                    // Get that many players from the waiting lobby and start the game
                    int numPlayers = 0;
                    while (numPlayers < _nextGame.RequiredPlayers)
                    {
                        // Pop the first one off
                        TcpClient player = _waitingLobby[0];
                        _waitingLobby.RemoveAt(0);

                        // Try adding it to the game.  If failure, put it back in the lobby
                        if (_nextGame.AddPlayer(player))
                            numPlayers++;
                        else
                            _waitingLobby.Add(player);
                    }

                    // Start the game in a new thread!
                    Console.WriteLine("Starting a \"{0}\" game.", _nextGame.Name);
                    Thread gameThread = new Thread(new ThreadStart(_nextGame.Run));
                    gameThread.Start();
                    _games.Add(_nextGame);
                    _gameThreads.Add(gameThread);

                    // Create a new game
                    _nextGame = new GuessMyNumberGame(this);
                }

                // Check if any clients have disconnected in waiting, gracefully or not
                // NOTE: This could (and should) be parallelized
                foreach (TcpClient client in _waitingLobby.ToArray())
                {
                    EndPoint endPoint = client.Client.RemoteEndPoint;
                    bool disconnected = false;

                    // Check for graceful first
                    Packet p = ReceivePacket(client).GetAwaiter().GetResult();
                    disconnected = (p?.Command == "bye");

                    // Then ungraceful
                    disconnected |= IsDisconnected(client);

                    if (disconnected)
                    {
                        HandleDisconnectedClient(client);
                        Console.WriteLine("Client {0} has disconnected from the Game(s) Server.", endPoint);
                    }
                }


                // Take a small nap
                Thread.Sleep(10);
            }

            // In the chance a client connected but we exited the loop, give them 1 second to finish
            Task.WaitAll(newConnectionTasks.ToArray(), 1000);

            // Shutdown all of the threads, regardless if they are done or not
            foreach (Thread thread in _gameThreads)
                thread.Abort();

            // Disconnect any clients still here
            Parallel.ForEach(_clients, (client) =>
                {
                    DisconnectClient(client, "The Game(s) Server is being shutdown.");
                });            

            // Cleanup our resources
            _listener.Stop();

            // Info
            Console.WriteLine("The server has been shut down.");
        }

        // Awaits for a new connection and then adds them to the waiting lobby
        private async Task _handleNewConnection()
        {
            // Get the new client using a Future
            TcpClient newClient = await _listener.AcceptTcpClientAsync();
            Console.WriteLine("New connection from {0}.", newClient.Client.RemoteEndPoint);

            // Store them and put them in the waiting lobby
            _clients.Add(newClient);
            _waitingLobby.Add(newClient);

            // Send a welcome message
            string msg = String.Format("Welcome to the \"{0}\" Games Server.\n", Name);
            await SendPacket(newClient, new Packet("message", msg));
        }

        // Will attempt to gracefully disconnect a TcpClient
        // This should be use for clients that may be in a game, or the waiting lobby
        public void DisconnectClient(TcpClient client, string message="")
        {
            Console.WriteLine("Disconnecting the client from {0}.", client.Client.RemoteEndPoint);

            // If there wasn't a message set, use the default "Goodbye."
            if (message == "")
                message = "Goodbye.";

            // Send the "bye," message
            Task byePacket = SendPacket(client, new Packet("bye", message));

            // Notify a game that might have them
            try
            {
                _gameClientIsIn[client]?.DisconnectClient(client);   
            } catch (KeyNotFoundException) { }

            // Give the client some time to send and proccess the graceful disconnect
            Thread.Sleep(100);

            // Cleanup resources on our end
            byePacket.GetAwaiter().GetResult();
            HandleDisconnectedClient(client);
        }

        // Cleans up the resources if a client has disconnected,
        // gracefully or not.  Will remove them from clint list and lobby
        public void HandleDisconnectedClient(TcpClient client)
        {
            // Remove from collections and free resources
            _clients.Remove(client);
            _waitingLobby.Remove(client);
            _cleanupClient(client);
        }

        #region Packet Transmission Methods
        // Sends a packet to a client asynchronously
        public async Task SendPacket(TcpClient client, Packet packet)
        {
            try
            {
                // convert JSON to buffer and its length to a 16 bit unsigned integer buffer
                byte[] jsonBuffer = Encoding.UTF8.GetBytes(packet.ToJson());
                byte[] lengthBuffer = BitConverter.GetBytes(Convert.ToUInt16(jsonBuffer.Length));

                // Join the buffers
                byte[] msgBuffer = new byte[lengthBuffer.Length + jsonBuffer.Length];
                lengthBuffer.CopyTo(msgBuffer, 0);
                jsonBuffer.CopyTo(msgBuffer, lengthBuffer.Length);

                // Send the packet
                await client.GetStream().WriteAsync(msgBuffer, 0, msgBuffer.Length);

                //Console.WriteLine("[SENT]\n{0}", packet);
            }
            catch (Exception e)
            {
                // There was an issue is sending
                Console.WriteLine("There was an issue receiving a packet.");
                Console.WriteLine("Reason: {0}", e.Message);
            }
        }

        // Will get a single packet from a TcpClient
        // Will return null if there isn't any data available or some other
        // issue getting data from the client
        public async Task<Packet> ReceivePacket(TcpClient client)
        {
            Packet packet = null;
            try
            {
                // First check there is data available
                if (client.Available == 0)
                    return null;

                NetworkStream msgStream = client.GetStream();

                // There must be some incoming data, the first two bytes are the size of the Packet
                byte[] lengthBuffer = new byte[2];
                await msgStream.ReadAsync(lengthBuffer, 0, 2);
                ushort packetByteSize = BitConverter.ToUInt16(lengthBuffer, 0);

                // Now read that many bytes from what's left in the stream, it must be the Packet
                byte[] jsonBuffer = new byte[packetByteSize];
                await msgStream.ReadAsync(jsonBuffer, 0, jsonBuffer.Length);

                // Convert it into a packet datatype
                string jsonString = Encoding.UTF8.GetString(jsonBuffer);
                packet = Packet.FromJson(jsonString);

                //Console.WriteLine("[RECEIVED]\n{0}", packet);
            }
            catch (Exception e)
            {
                // There was an issue in receiving
                Console.WriteLine("There was an issue sending a packet to {0}.", client.Client.RemoteEndPoint);
                Console.WriteLine("Reason: {0}", e.Message);
            }

            return packet;
        }
        #endregion // Packet Transmission Methods

        #region TcpClient Helper Methods
        // Checks if a client has disconnected ungracefully
        // Adapted from: http://stackoverflow.com/questions/722240/instantly-detect-client-disconnection-from-server-socket
        public static bool IsDisconnected(TcpClient client)
        {
            try
            {
                Socket s = client.Client;
                return s.Poll(10 * 1000, SelectMode.SelectRead) && (s.Available == 0);
            }
            catch(SocketException)
            {
                // We got a socket error, assume it's disconnected
                return true;
            }
        }

        // cleans up resources for a TcpClient and closes it
        private static void _cleanupClient(TcpClient client)
        {
            client.GetStream().Close();     // Close network stream
            client.Close();                 // Close client
        }
        #endregion // TcpClient Helper Methods





        #region Program Execution
        public static TcpGamesServer gamesServer;

        // For when the user Presses Ctrl-C, this will gracefully shutdown the server
        public static void InterruptHandler(object sender, ConsoleCancelEventArgs args)
        {
            args.Cancel = true;
            gamesServer?.Shutdown();
        }

        public static void Main(string[] args)
        {
            // Some arguments
            string name = "Bad BBS";//args[0];
            int port = 6000;//int.Parse(args[1]);

            // Handler for Ctrl-C presses
            Console.CancelKeyPress += InterruptHandler;

            // Create and run the server
            gamesServer = new TcpGamesServer(name, port);
            gamesServer.Run();
        }
        #endregion // Program Execution
    }
}
```

在顶端我们定义了很多内存数据，在其中需要注意的时`_waitingLobby`和`_nextGame`，`_waitingLobby`我是一个列表但我们将其用作队列，从他的名字，可以得知我们可以得知这是我们保存已经连接上但还没有立刻进入游戏的客户端。`_nextGame`是我们放下一场游戏的地方，一旦有足够多的客户端，我们你就会运行它。

构造函数除了初始化一些数据和创建一个`TcpListener`之外没有做啥。`Shutdown()`是一个方法，我们将在中断处理程序中调用它来关闭服务器。

`Run()`是服务器的核心，最开始我们打开监听器，创建一个`GuessMyNumberGame`，如果有新的连接在等待，我们将其放到`HandleNewConnection()`。这个方法在`Task`中运行，因此它是异步的。如果我们在等待厅中有足够多的玩家，我们将其推出，然后尝试加入到`_nextGame`中。如果游戏不想接受一个客户端（我不知道，也许他们不喜欢他们的IP）。将其放回等待厅的末尾。一旦玩家被加入进来，将会开启一个新线程开始游戏，我们设置一个新的`GuessMyNumberGame`加入队列。在这之后我们遍历等待厅中的客户端看他们是否需要断开连接。

一旦`while(Running)`的执行退出了，我们确保新连接的任务都已经完成了。然后使用`Thread.Abort()`方法关闭所有游戏。我们断开所有和客户端的连接，告诉他们我们正在关闭服务器，最后我们将会停止对新连接的监听。

`HandleNewConnection()`将会异步接受客户端，然后把它们放在等待区，并且给他们发送“hi”。

`DisconnectClient()`是用来优雅的客户端断开连接方法。它他通知游戏服务器正在和客户端断开连接（游戏可以选择如何处理断开连接客户端）。我们调用`Thread.Sleep()`因为有可能`bye packet`已经被发送了，但是在客户端处理它之前，收到了服务器的FIN/ACK包。这会导致客户端认为这不是优雅的断开连接。休眠100毫秒给客户端充足的时间在服务器端清理其相关资源的之前来处理消息然后主动关闭。（这可以被归类为竞争条件，我将会在回顾中讨论他）

`HandleDisconnectedClient()`是一个小的帮助类，帮助从集合（如`_waitingLobby`）中移除客户端，然后将其在服务器上的资源释放。

`SendPacket()`是另一个异步方法来在网络流中转换`Packet`。它为我们处理所有字节数组格式。`ReceivePacket()`则和他相反，它是获得一个包。如果客户端的请求中没有可用数据，将会返回null

在转换帮助类之后，我们有`TcpClient`帮助类。`IsDisconnected()`在其他章节中描述过了，`CleanUpClient()`清除`TcpClient`的资源

在类的结尾，我们有一些程序执行的代码。最后还有按Ctrl-C来关闭服务器的中断处理。