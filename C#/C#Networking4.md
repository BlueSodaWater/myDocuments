## UDP

UDP和TCP很不一样，因为你必须使用数据包而不是流和连接。可能会很快（但不可靠）。但还有很多事情是我们要做的。为了想出一个好的例子，我决定构建一个简单的文件传输应用程序（基于UDP）。

### Before Starting

这个程序将会被分为两个部分，发送端和接收端。发送端会像服务器一样把文件发给接收端。

代码将会使用基于[UdpClient](https://msdn.microsoft.com/en-us/library/system.net.sockets.udpclient(v=vs.110).aspx)的同步单线程类。简单提醒一下这个不是UDP的教程，而是如何使用C#中UDP的API。如果想要可以去尝试了解更多

### Testing File

鉴于我们正在处理文件传输，我认为使用视频 [Don't Copy That Floppy](https://archive.org/details/dontcopythatfloppy)是最合适的测试用例。

### UDP File Transfet - Protocal Design

这个协议可能会有些复杂。因为UDP没有内置ACK，我们需要手动添加他们。不是每个消息都要有ACK，，只有负责控制的有就行。`Packet`类，包含两个字段，`PacketType`（控制）和`Payload`（数据）。对不同类型，在`Payload`的数据结构可能会不同。

### Block

文件数据将会以`Block`的形式传输。他们有两个字段，一个无符号的32位则行数是他们的ID `Number`。另一个字节数组包含数据`Data`。在`Block`中的`Data`是被压缩的。

### Packet Types

这些是用来告诉我们的程序每个数据报是什么意思

- `ACK` 一个确认，主要是控制消息，由双方发送
- `BYE` 一个结束消息。可以有接收方或发送方随时发送
- `REQF` 请求文件。由接收方发送给发送方，需要发送方的确认
- `INFO` 关于文件的信息。由发送方在`REQF`之后发送给接收方，需要接收方的确认
- `REQB` 对数据块的请求，由接收方在`INFO`之后发送给发送方
- `SEND` 对请求块的响应，包含数据块。由发送方在`REQB`之后发送给接收方

下面一张图告诉我们数据是如何交换的

![UDP File Transfer Diagram](https://storage.googleapis.com/sixteenbpp/tutorials/images/udp-file-transfer-protocol-diagram.png)

### Packet Data(Payloads)

下面是一些交换应该包含的信息

- `REQF` `Payload`应该包含所需文件名的UTF8格式的字符串
  - `ACK`应该由发送方发出，如果文件存在，他应该包含与`REQF`相同的`Payload`，如果不存在，应该发送一个空的`Payload`。
- `INFO` `Payload`的前16个字节必须是原始文件（未压缩）的MD5校验和。接下来4个字节是代表文件大小（字节）的`UINT32`。随后四个字节是`Block`大小的最大值（字节），最后四个字节是需要发送的`Block`的数量。
  - `ACK`应该由接收方发送，应该包含一个空的`Payload`
- `REQB` `Payload`是一个`UInt32`的号码，即接收方请求的`Block.Number` 
- `SEND` `Payload`是一个`Block`，其号码是前一个`REQB`请求的那个。
- `BYE` 无需`Payload`

如果你感到困惑，可以看下面的图片

![UDP File Transfer - Packet Examples](https://storage.googleapis.com/sixteenbpp/tutorials/images/udp-file-transfer-packet-examples.jpeg)

### UDP File Transfer - Common Files

UDP例子分为两个项目，但他们中有相同的代码（我建议你输入这些到吗），可以尽管的复制他们。但确保你阅读了每个内容下的摘要。

### Block

```c#
using System;
using System.Text;
using System.Linq;

namespace UdpFileTransfer
{
    // These are the chunks of data that will be sent across the network
    public class Block
    {
        public UInt32 Number { get; set; }
        public byte[] Data { get; set; } = new byte[0];

        #region Constructors
        // Creates a new block of data w/ the supplied number
        public Block(UInt32 number=0)
        {
            Number = number;
        }

        // Creates a Block from a byte array
        public Block (byte[] bytes)
        {
            // First four bytes are the number
            Number = BitConverter.ToUInt32(bytes, 0);

            // Data starts at byte 4
            Data = bytes.Skip(4).ToArray();
        }
        #endregion // Constructors

        public override string ToString()
        {
            // Take some of the first few bits of data and turn that into a string
            String dataStr;
            if (Data.Length > 8)
                dataStr = Encoding.ASCII.GetString(Data, 0, 8) + "...";
            else
                dataStr = Encoding.ASCII.GetString(Data, 0, Data.Length);

            return string.Format(
                "[Block:\n" +
                "  Number={0},\n" +
                "  Size={1},\n" +
                "  Data=`{2}`]",
                Number, Data.Length, dataStr);
        }

        // Returns the data in the block as a byte array
        public byte[] GetBytes()
        {
            // Convert meta-data
            byte[] numberBytes = BitConverter.GetBytes(Number);

            // Join all the data into one bigger array
            byte[] bytes = new byte[numberBytes.Length + Data.Length];
            numberBytes.CopyTo(bytes, 0);
            Data.CopyTo(bytes, 4);

            return bytes;
        }
    }
}
```

这是一个小的数据容器，用于将一个较大的文件分割成较小的块。我们有两个构造函数，一个用于创建带有设定好的`Number`的新对象，另一个从字节数组重新构造他。`GetBytes()`方法用于以字节形式获取`Block`。

### Packet

```c#
using System;
using System.Text;
using System.Linq;

namespace UdpFileTransfer
{
    public class Packet
    {
        #region Messge Types (Static)
        public static UInt32 Ack = BitConverter.ToUInt32(Encoding.ASCII.GetBytes("ACK "), 0);
        public static UInt32 Bye = BitConverter.ToUInt32(Encoding.ASCII.GetBytes("BYE "), 0);
        public static UInt32 RequestFile = BitConverter.ToUInt32(Encoding.ASCII.GetBytes("REQF"), 0);
        public static UInt32 RequestBlock = BitConverter.ToUInt32(Encoding.ASCII.GetBytes("REQB"), 0);
        public static UInt32 Info = BitConverter.ToUInt32(Encoding.ASCII.GetBytes("INFO"), 0);
        public static UInt32 Send = BitConverter.ToUInt32(Encoding.ASCII.GetBytes("SEND"), 0);
        #endregion

        // The Fields for the packet
        public UInt32 PacketType { get; set; }
        public byte[] Payload { get; set; } = new byte[0];

        #region Handy Properties
        public bool IsAck { get { return PacketType == Ack; } }
        public bool IsBye { get { return PacketType == Bye; } }
        public bool IsRequestFile { get { return PacketType == RequestFile; } }
        public bool IsRequestBlock { get { return PacketType == RequestBlock; } }
        public bool IsInfo { get { return PacketType == Info; } }
        public bool IsSend { get { return PacketType == Send; } }
        public bool IsUnknown { get { return !(IsAck || IsBye || IsRequestFile || IsRequestBlock || IsInfo || IsSend); } }

        public string MessageTypeString { get { return Encoding.UTF8.GetString(BitConverter.GetBytes(PacketType)); } }
        #endregion

        #region Constructors
        public Packet(UInt32 packetType)
        {
            // Set the message type
            PacketType = packetType;
        }

        // Creates a Packet from a byte array
        public Packet(byte[] bytes)
        {
            PacketType = BitConverter.ToUInt32(bytes, 0);      // Will grab the first four bytes (which are the type)

            // Payload starts at byte 4
            Payload = new byte[bytes.Length - 4];
            bytes.Skip(4).ToArray().CopyTo(Payload, 0);
        }
        #endregion // Constructors

        public override string ToString()
        {
            // Take some of the first few bits of data and turn that into a string
            String payloadStr;
            int payloadSize = Payload.Length;
            if (payloadSize > 8)
                payloadStr = Encoding.ASCII.GetString(Payload, 0, 8) + "...";
            else
                payloadStr = Encoding.ASCII.GetString(Payload, 0, payloadSize);

            // type string
            String typeStr = "UKNOWN";
            if (!IsUnknown)
                typeStr = MessageTypeString;
            
            return string.Format(
                "[Packet:\n" +
                "  Type={0},\n" +
                "  PayloadSize={1},\n" +
                "  Payload=`{2}`]",
                typeStr, payloadSize, payloadStr);
        }


        // Gets the Packet as a byte array
        public byte[] GetBytes()
        {
            // Join the byte arrays
            byte[] bytes = new byte[4 + Payload.Length];
            BitConverter.GetBytes(PacketType).CopyTo(bytes, 0);
            Payload.CopyTo(bytes, 4);

            return bytes;
        }
    }

    #region Definite Packets
    // ACK
    public class AckPacket : Packet
    {
        public string Message
        {
            get { return Encoding.UTF8.GetString(Payload); }
            set { Payload = Encoding.UTF8.GetBytes(value); }
        }

        public AckPacket(Packet p=null) :
            base(Ack)
        {
            if (p != null)
                Payload = p.Payload;
        }
    }

    // REQF
    public class RequestFilePacket : Packet
    {
        public string Filename
        {
            get { return Encoding.UTF8.GetString(Payload); }
            set { Payload = Encoding.UTF8.GetBytes(value); }
        }

        public RequestFilePacket(Packet p=null) :
            base(RequestFile)
        {
            if (p != null)
                Payload = p.Payload;
        }

    }

    // REQB
    public class RequestBlockPacket : Packet
    {
        public UInt32 Number
        {
            get { return BitConverter.ToUInt32(Payload, 0); }
            set { Payload = BitConverter.GetBytes(value); }
        }

        public RequestBlockPacket(Packet p = null)
            : base(RequestBlock)
        {
            if (p != null)
                Payload = p.Payload;
        }
    }

    // INFO
    public class InfoPacket : Packet
    {
        // Should be an MD5 checksum
        public byte[] Checksum
        {
            get { return Payload.Take(16).ToArray(); }
            set { value.CopyTo(Payload, 0); }
        }

        public UInt32 FileSize
        {
            get { return BitConverter.ToUInt32(Payload.Skip(16).Take(4).ToArray(), 0); }
            set { BitConverter.GetBytes(value).CopyTo(Payload, 16); }
        }

        public UInt32 MaxBlockSize
        {
            get { return BitConverter.ToUInt32(Payload.Skip(16 + 4).Take(4).ToArray(), 0); }
            set { BitConverter.GetBytes(value).CopyTo(Payload, 16 + 4); }
        }

        public UInt32 BlockCount
        {
            get { return BitConverter.ToUInt32(Payload.Skip(16 + 4 + 4).Take(4).ToArray(), 0); }
            set { BitConverter.GetBytes(value).CopyTo(Payload, 16 + 4 + 4); }
        }

        public InfoPacket(Packet p = null)
            : base(Info)
        {
            if (p != null)
                Payload = p.Payload;
            else
                Payload = new byte[16 + 4 + 4 + 4];
        }
    }

    // SEND
    public class SendPacket : Packet
    {
        public Block Block {
            get { return new Block(Payload); }
            set { Payload = value.GetBytes(); }
        }

        public SendPacket(Packet p=null)
            : base(Send)
        {
            if (p != null)
                Payload = p.Payload;
        }
    }
    #endregion // Definite Packets
}
```

`Packet`是我们在网络上来回发送的东西，他只有两个数据字段，`PacketType`和`Payload`。所有的`Packet`都应该具有类型列表中的一项。就像`Block`，Packet也有一个从自己数组中的构造器，和一个`GetBytes()`方法来吧`Packet`转化为字节数组。

在底部是具体的包，是的某些消息类型更容易处理`Payload`消息。你会注意到没有`Bye`因为它不需要`Payload`。

### NetworkMessage

```c#
using System.Net;

namespace UdpFileTransfer
{
    // This is a Simple datastructure that is used in packet queues
    public class NetworkMessage
    {
        public IPEndPoint Sender { get; set; }
        public Packet Packet { get; set; }
    }
}
```

`NetworkMessage`是一个简单的数据结构用来将数据包和发送者配对。只当我们读取获取的数据的时候才会使用。

### UDP File Transfer - Sender

下面是等待文件请求并响应的程序。扮演者服务器的角色，注意到`Main()`，你可以注释掉硬编码参数，而使用命令行参数。

```c#
using System;
using System.Diagnostics;
using System.IO;
using System.IO.Compression;
using System.Text;
using System.Linq;
using System.Net;
using System.Net.Sockets;
using System.Threading;
using System.Collections.Generic;
using System.Security.Cryptography;

namespace UdpFileTransfer
{
    public class UdpFileSender
    {
        #region Statics
        public static readonly UInt32 MaxBlockSize = 8 * 1024;   // 8KB
        #endregion // Statics

        enum SenderState
        {
            NotRunning,
            WaitingForFileRequest,
            PreparingFileForTransfer,
            SendingFileInfo,
            WaitingForInfoACK,
            Transfering
        }

        // Connection data
        private UdpClient _client;
        public readonly int Port;
        public bool Running { get; private set; } = false;

        // Transfer data
        public readonly string FilesDirectory;
        private HashSet<string> _transferableFiles;
        private Dictionary<UInt32, Block> _blocks = new Dictionary<UInt32, Block>();
        private Queue<NetworkMessage> _packetQueue = new Queue<NetworkMessage>();

        // Other stuff
        private MD5 _hasher;

        // Constructor, creates a UdpClient on <port>
        public UdpFileSender(string filesDirectory, int port)
        {
            FilesDirectory = filesDirectory;
            Port = port;
            _client = new UdpClient(Port, AddressFamily.InterNetwork);      // Bind in IPv4
            _hasher = MD5.Create();
        }

        // Prepares the Sender for file transfers
        public void Init()
        {
            // Scan files (only the top directory)
            List<string> files = new List<string>(Directory.EnumerateFiles(FilesDirectory));
            _transferableFiles = new HashSet<string>(files.Select(s => s.Substring(FilesDirectory.Length + 1)));

            // Make sure we have at least one to send
            if (_transferableFiles.Count != 0)
            {
                // Modify the state
                Running = true;

                // Print info
                Console.WriteLine("I'll transfer these files:");
                foreach (string s in _transferableFiles)
                    Console.WriteLine("  {0}", s);
            }
            else
                Console.WriteLine("I don't have any files to transfer.");
        }

        // signals for a (graceful) shutdown)
        public void Shutdown()
        {
            Running = false;
        }

        // Main loop of the sender
        public void Run()
        {
            // Transfer state
            SenderState state = SenderState.WaitingForFileRequest;
            string requestedFile = "";
            IPEndPoint receiver = null;

            // This is a handly little function to reset the transfer state
            Action ResetTransferState = new Action(() =>
                {
                    state = SenderState.WaitingForFileRequest;
                    requestedFile = "";
                    receiver = null;
                    _blocks.Clear();
                });

            while (Running)
            {
                // Check for some new messages
                _checkForNetworkMessages();
                NetworkMessage nm = (_packetQueue.Count > 0) ? _packetQueue.Dequeue() : null;

                // Check to see if we have a BYE
                bool isBye = (nm == null) ? false : nm.Packet.IsBye;
                if (isBye)
                {
                    // Set back to the original state
                    ResetTransferState();
                    Console.WriteLine("Received a BYE message, waiting for next client.");
                }

                // Do an action depending on the current state
                switch (state)
                {
                    case SenderState.WaitingForFileRequest:
                        // Check to see that we got a file request
                        
                        // If there was a packet, and it's a request file, send and ACK and switch the state
                        bool isRequestFile = (nm == null) ? false : nm.Packet.IsRequestFile;
                        if (isRequestFile)
                        {
                            // Prepare the ACK
                            RequestFilePacket REQF = new RequestFilePacket(nm.Packet);
                            AckPacket ACK = new AckPacket();
                            requestedFile = REQF.Filename;

                            // Print info
                            Console.WriteLine("{0} has requested file file \"{1}\".", nm.Sender, requestedFile);

                            // Check that we have the file
                            if (_transferableFiles.Contains(requestedFile))
                            {
                                // Mark that we have the file, save the sender as our current receiver
                                receiver = nm.Sender;
                                ACK.Message = requestedFile;
                                state = SenderState.PreparingFileForTransfer;

                                Console.WriteLine("  We have it.");
                            }
                            else
                                ResetTransferState();

                            // Send the message
                            byte[] buffer = ACK.GetBytes();
                            _client.Send(buffer, buffer.Length, nm.Sender);
                        }
                        break;

                    case SenderState.PreparingFileForTransfer:
                        // Using the requested file, prepare it in memory
                        byte[] checksum;
                        UInt32 fileSize;
                        if (_prepareFile(requestedFile, out checksum, out fileSize))
                        {
                            // It's good, send an info Packet
                            InfoPacket INFO = new InfoPacket();
                            INFO.Checksum = checksum;
                            INFO.FileSize = fileSize;
                            INFO.MaxBlockSize = MaxBlockSize;
                            INFO.BlockCount = Convert.ToUInt32(_blocks.Count);

                            // Send it
                            byte[] buffer = INFO.GetBytes();
                            _client.Send(buffer, buffer.Length, receiver);

                            // Move the state
                            Console.WriteLine("Sending INFO, waiting for ACK...");
                            state = SenderState.WaitingForInfoACK;
                        }
                        else
                            ResetTransferState();   // File not good, reset the state
                        break;

                    case SenderState.WaitingForInfoACK:
                        // If it is an ACK and the payload is the filename, we're good
                        bool isAck = (nm == null) ? false : (nm.Packet.IsAck);
                        if (isAck)
                        {
                            AckPacket ACK = new AckPacket(nm.Packet);
                            if (ACK.Message == "INFO")
                            {
                                Console.WriteLine("Starting Transfer...");
                                state = SenderState.Transfering;
                            }
                        }
                        break;

                    case SenderState.Transfering:
                        // If there is a block request, send it
                        bool isRequestBlock = (nm == null) ? false : nm.Packet.IsRequestBlock;
                        if (isRequestBlock)
                        {
                            // Pull out data
                            RequestBlockPacket REQB = new RequestBlockPacket(nm.Packet);
                            Console.WriteLine("Got request for Block #{0}", REQB.Number);

                            // Create the response packet
                            Block block = _blocks[REQB.Number];
                            SendPacket SEND = new SendPacket();
                            SEND.Block = block;

                            // Send it
                            byte[] buffer = SEND.GetBytes();
                            _client.Send(buffer, buffer.Length, nm.Sender);
                            Console.WriteLine("Sent Block #{0} [{1} bytes]", block.Number, block.Data.Length); 
                        }
                        break;
                }

                Thread.Sleep(1);
            }

            // If there was a receiver set, that means we need to notify it to shutdown
            if (receiver != null)
            {
                Packet BYE = new Packet(Packet.Bye);
                byte[] buffer = BYE.GetBytes();
                _client.Send(buffer, buffer.Length, receiver);
            }

            state = SenderState.NotRunning;
        }

        // Shutsdown the underlying UDP client
        public void Close()
        {
            _client.Close();
        }

        // Trys to fill the queue of packets
        private void _checkForNetworkMessages()
        {
            if (!Running)
                return;

            // Check that there is something available (and at least four bytes for type)
            int bytesAvailable = _client.Available;
            if (bytesAvailable >= 4)
            {
                // This will read ONE datagram (even if multiple have been received)
                IPEndPoint ep = new IPEndPoint(IPAddress.Any, 0);
                byte[] buffer = _client.Receive(ref ep);

                // Create the message structure and queue it up for processing
                NetworkMessage nm = new NetworkMessage();
                nm.Sender = ep;
                nm.Packet = new Packet(buffer);
                _packetQueue.Enqueue(nm);
            }
        }

        // Loads the file into the blocks, returns true if the requested file is ready
        private bool _prepareFile(string requestedFile, out byte[] checksum, out UInt32 fileSize)
        {
            Console.WriteLine("Preparing the file to send...");
            bool good = false;
            fileSize = 0;

            try
            {
                // Read it in & compute a checksum of the original file
                byte[] fileBytes = File.ReadAllBytes(Path.Combine(FilesDirectory, requestedFile));
                checksum = _hasher.ComputeHash(fileBytes);
                fileSize = Convert.ToUInt32(fileBytes.Length);
                Console.WriteLine("{0} is {1} bytes large.", requestedFile, fileSize);

                // Compress it
                Stopwatch timer = new Stopwatch();
                using (MemoryStream compressedStream = new MemoryStream())
                {
                    // Perform the actual compression
                    DeflateStream deflateStream = new DeflateStream(compressedStream, CompressionMode.Compress, true);
                    timer.Start();
                    deflateStream.Write(fileBytes, 0, fileBytes.Length);
                    deflateStream.Close();
                    timer.Stop();

                    // Put it into blocks
                    compressedStream.Position = 0;
                    long compressedSize = compressedStream.Length;
                    UInt32 id = 1;
                    while (compressedStream.Position < compressedSize)
                    {
                        // Grab a chunk
                        long numBytesLeft = compressedSize - compressedStream.Position;
                        long allocationSize = (numBytesLeft > MaxBlockSize) ? MaxBlockSize : numBytesLeft;
                        byte[] data = new byte[allocationSize];
                        compressedStream.Read(data, 0, data.Length);

                        // Create a new block
                        Block b = new Block(id++);
                        b.Data = data;
                        _blocks.Add(b.Number, b);
                    }

                    // Print some info and say we're good
                    Console.WriteLine("{0} compressed is {1} bytes large in {2:0.000}s.", requestedFile, compressedSize, timer.Elapsed.TotalSeconds);
                    Console.WriteLine("Sending the file in {0} blocks, using a max block size of {1} bytes.", _blocks.Count, MaxBlockSize);
                    good = true;
                }
            }
            catch (Exception e)
            {
                // Crap...
                Console.WriteLine("Could not prepare the file for transfer, reason:");
                Console.WriteLine(e.Message);

                // Reset a few things
                _blocks.Clear();
                checksum = null;
            }

            return good;
        }




        #region Program Execution
        public static UdpFileSender fileSender;

        public static void InterruptHandler(object sender, ConsoleCancelEventArgs args)
        {
            args.Cancel = true;
            fileSender?.Shutdown();
        }

        public static void Main(string[] args)
        {
            // Setup the sender
            string filesDirectory = "Files";//args[0].Trim();
            int port = 6000;//int.Parse(args[1].Trim());
            fileSender = new UdpFileSender(filesDirectory, port);

            // Add the Ctrl-C handler
            Console.CancelKeyPress += InterruptHandler;

            // Run it
            fileSender.Init();
            fileSender.Run();
            fileSender.Close();
        }
        #endregion // Program Execution
    }
}
```

这里代码很多，让我们拆开说

在`UdpFileSender`的构造器中，我们告诉它想要作为文件传输的目录，然后创建`UdpClient`。我们创建它来在侦听指定端口的数据报，只侦听IPv4连接。

`Init()`将会扫描`FilesDirectory`(非递归的)中的每一个文件然后将他们标记为需要发送的。同时它也把`Running`设置为`true`，这样我们可以从接收方那边监听数据。`Shutdown()`就是简单的用来关闭服务。

`Run()`是`UdpFileSender`的主要方法。在顶端我们设置了一些传输状态的变量，并且设置了一个迷你帮助类来以便在某些情况下重置状态。在循环的开始，我们看有无发送给我们的新的数据包，如果有一个`BYE`消息，意味着客户端暂时断开连接，此时我们应该重置传输状态来等待新的客户端。switch语句用来遍历传输过程，每个状态都会等待合适类型的`Packet`，然后处理他们（例如创建响应，准备传输文件）然后继续下面的操作。这点我不想说太多细节，因为很容易理解。在循环的底部，我们暂停一会来节省CPU的资源。在函数的最后，如果有一个当前连接客户端，我们将会发送一个BYE消息。（防止发送方在发生中途想要关闭）。

讨论一下`UdpClient`中的`Send`方法，我们重载需要您的列表和端点。如果你的应用可能连接多个客户端，我推荐使用这个方法。记住UDP是无连接的，因为我们不需要和任何客户端建立连接，如果你看到Connect方法，请忽略它。

`Close()`将会清除UdpClient的资源。

`CheckForNetworkMessages()`将会看有无获得的数据报。由于我们的报至少4字节长度。我们希望这些字节都已经准备好了，我们创建IPEndPoind对象。利用`UdpClient.Receive`，它将准确抓取被网络接受的数据报。不想`TcpClient`，我们不需要在`NetworkStream`上浪费时间。在我们获得他之后，我们把他的发送者和数据包推到一个NetworkMessage之中稍后排队处理。

`PrepareFile()`所做的事情正如他名字一样，处理文件以便发送。它接受一个文件路径，将其原始数据计算校验和，将其压缩，然后分割成`Block`。如果运行正确，会返回`true`，他的`out`参数将会被填充。

这就是UdpFileSender中的全部内容，如果你忽略了`Run()`中的switch语句，我推荐你阅读他。