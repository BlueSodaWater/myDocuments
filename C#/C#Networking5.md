## UDP Pong

现在我们准备一些有趣的例子，我们准备写一个可以在网上玩的弹球游戏。将有一个服务器，许多玩家连接并且进行一对一的游戏，弹球游戏是游戏中的“Hello World”。我认为这是尝试网络的一个好的开端。

由于我们准备制作游戏，我选择[MonoGame ](http://www.monogame.net/)作为我们的媒体库。它是跨平台的，使用起来很简单。我本想本系列大部分教程都放在命令行中，但我不得不打破这一原则。我使用[MonoGame.Framework.DesktopGL](https://www.nuget.org/packages/MonoGame.Framework.DesktopGL/)，如果你使用的不是Linux版本的，可能需要使用Windows或者MacOS版本。

注意：音频播放会有问题（在我写这篇文章的时候是3.5），据我所知这只适用于桌面包.这是一个很无趣的问题，因为有些项目使用比较老版本的MonoGame，但听起来还不错。如果你使用的是Linux系统，如果你真的想要播放声音，你可能需要老一点的版本。

我包含了一个宏定义，允许你在客户端上的时候切换音频播放，默认是禁用的。

### Architecture Overview

这个教程分为两个应用，`PongServer`和`PongClient`（在你的解决方案中建立这两个项目）。服务器有权处理冲突（球和板，目标，上下边框），客户端播放音效和位置更新，所有客户的都要连接服务器并且告诉它球拍的位置。

在游戏实例中（在服务器代码中被称为竞技场）。最多有两个玩家，尽管服务器应该有能力同时处理多个游戏。直到两个客户端都连接上了，游戏才会开始，如果一个客户端连接之后断开了连接，游戏被认定为结束，无论比赛是否已经开始

### Game Assets

下面的文件在客户端中被称为`Content`。确保这些文件在编译的时候复制到构建目录中。

- [paddle.png](https://storage.googleapis.com/sixteenbpp/tutorials/c-sharp-networking/networked-pong/Content/paddle.png)
- [ball.png](https://storage.googleapis.com/sixteenbpp/tutorials/c-sharp-networking/networked-pong/Content/ball.png)
- [establishing-connection-msg.png](https://storage.googleapis.com/sixteenbpp/tutorials/c-sharp-networking/networked-pong/Content/establishing-connection-msg.png)
- [waiting-for-game-start-msg.png](https://storage.googleapis.com/sixteenbpp/tutorials/c-sharp-networking/networked-pong/Content/waiting-for-game-start-msg.png)
- [game-over-msg.png](https://storage.googleapis.com/sixteenbpp/tutorials/c-sharp-networking/networked-pong/Content/game-over-msg.png)
- [ball-hit.wav](https://storage.googleapis.com/sixteenbpp/tutorials/c-sharp-networking/networked-pong/Content/ball-hit.wav)
- [score.wav](https://storage.googleapis.com/sixteenbpp/tutorials/c-sharp-networking/networked-pong/Content/score.wav)

### UDP Pong - Protocol Design

这里我们有很多要做的比如在UDP中传输文件。由于UDP是无连接的，但是我们的客服端需要有一个稳定的连接连维持，传输数据，想要关闭的时候可以关闭，我们将自己实现这些

### Basic Packet Structure

每个包的前四个字节代表他的`PacketType`。我们将使用32位无符号整数（4个字节）而不是像文件传输那个例子里面一样使用一些ASCII编码。加下来的八个字节会是一个有符号的64位整数（8个字节）编码的`Timestamp`。接下来是`Payload`，只是一个字节数组，可以为空，也可以有内容。这都取决于`PacketType`，因此`Packet`最小是12个字节。

### Types of Packets

下面是每个包具体内容和描述：

1. `RequestJoin`：由服务器给客户端发送来加入游戏
2. `AcceptJoin`：由服务器给客户端发送作为上面的回应，包含了客户端应该是哪个`Paddle`
3. `AcceptJoinAck`：有客户端给服务器发送了解了上面的内容
4. `Heartbeat`：客户端发送给服务器通知他还在连接中
5. `HeartbeatAck`：服务器发送给客户端告知收到了上面的消息
6. `GameStart`：服务器给客户端发游戏开始了
7. `GameStartAck`：客户端给服务器发了解了上面的消息
8. `PaddlePosition`：客户端给服务器发`Paddle`当前位置
9. `GameState`：服务器给客户端发当前游戏的状态（`Paddle`位置，`Ball`位置等）
10. `PlaySoundEffect`：服务器给客户端发播放声音
11. `Bye`：两方同时发送来提醒游戏结束

### Connection Establishment

这是我们方式的[TCP Three-Way Handshake](http://www.omnisecu.com/tcpip/tcp-three-way-handshake.php)。客户端给服务器发送一个`RequestJoin`。服务器会回复一个`AcceptJoin`消息。他是唯一一个包含数据的（一个无符号32位整数代表客户端的`Paddle`）。然后客户端会夫回复一个`AcceptJoinAck`。下面是示意图：



![UDP Pong Handshake](https://storage.googleapis.com/sixteenbpp/tutorials/images/udp-pong-handshake.png)

### Maintaining the Connection

很简单，客户端发送`Heartbeat`消息然后服务器会发送`HeartbeatAck`消息来返回。客户端和服务器也会记录获取连接的 事件。心跳的值将会是20秒。在连接稳定之后，在等待`Game'Start`/`GameStartAck`，和游戏在进行中的时候发送。

### Game Start

直到两个客户端连接上，游戏才会开始。一旦满足了这些标准。将会发送`GameStart`消息。客户端也需要响应一个`GameStaartAck`在服务器发送任何`GameState`消息之前。

### In Game

客户端和服务器每秒发送几次他们的状态信息。客户端仅仅需要发送`Paddle Packet`，它包含了`Paddle`在Y轴位置上的浮点数。服务器同时给每个客户端发送`GameState Packet`。他会包含`Paddle`和`Ball`的位置和分数。服务器会定期发送`PalySoundEffect`消息。他会告诉客户端打开音频。音频的名字是UTF-8格式的字符串。

查看Packet.cs中的代码来获取更多细节，尤其是`GameStatePacket`，`PaddlePositionPacket`和`PlaySoundEffect`类，他们会告诉你数据如何布局

### Ending the Connection

服务器和客户端任何时候都可以发送`Bye`。简单的告诉另一方需要断开连接。如果客户端在游戏进行中发此消息给服务器，服务器需要提醒其另一个客户端游戏结束了。当服务器关闭的时候，需要给所有连接的客户端发送`Bye`。

### UDP Pong - Game Mechaics

就像之前说的，弹球游戏就像游戏开发中的“Hello,World”。我很确信大部分人都有这样的经历，但我选择用一种不同的方式来处理它。所以我觉得我们需要讨论一下我们要做什么。

### Scoring

如果`Ball`击中了屏幕的左侧，右侧的玩家得分，反之亦然。球应该重置到屏幕中间，还应该有”得分“的声音

### Key Press

这个只适用于客户端，如果用户任何时候点了`Esc`，将会关闭客户端并且通知服务器。上下键用来控制`Paddle`的位置

### Hitting Top & Bottom Bounds

如果`Ball`击中屏幕的上下方，它应该反转速度的Y分量。速度也应该随机增加并且发出“弹击”的声音。

### Hitting the Paddles

每一个`Paddle`有三个碰撞区域。一个在顶端，一个在底部，还有一个在中间（称之为"Front"）。如果Ball击中了其中的任何一个，速度会小幅提示并且X轴方向需要翻转（同时也应该播放音效）。但是如果击中了顶部或这地步，应该也同时翻转`Ball`速度的Y分量，下面是示意图：

![Ball & Paddle Hitboxes](https://storage.googleapis.com/sixteenbpp/tutorials/images/ball-paddle-hitboxes.png)

### UDP Pong - Common Files

将这些文件加到服务器和客户端项目中，如果你想要拷贝复制`Packet.cs`。我不会责怪你（因为它很长，无聊，并且重复）。但需要关注`Packet`类，它的子类有`Payload`数据。

正确的方法是在解决方法中添加一个额外的项目来包含这些公告代码（比如`PongFramework`）。但是我把这些公共文件放入一个项目中（比如`PongServer`），然后将这些文件关联到其他项目中（比如`PongClient`）。这个方式比较容易

### GameGeometry

这是一个静态数据类，包含着硬编码的值。

```c#
using System;
using Microsoft.Xna.Framework;

namespace PongGame
{
    // This is a class that contains all the information for the geometry
    // of the objects in the game (play area, paddle/ball sizes, etc.)
    public static class GameGeometry
    {
        public static readonly Point PlayArea = new Point(320, 240);    // Client area
        public static readonly Vector2 ScreenCenter                     // Center point of the screen
            = new Vector2(PlayArea.X / 2f, PlayArea.Y / 2f);
        public static readonly Point BallSize = new Point(8, 8);        // Size of Ball
        public static readonly Point PaddleSize = new Point(8, 44);     // Size of the Paddles
        public static readonly int GoalSize = 12;                       // Width behind paddle
        public static readonly float PaddleSpeed = 100f;                // Speed of the paddle, (pixels/sec)
    }
}

### NetworkMessage

这和文件传输例子中的几乎相同，但也增加了字段，当一方接收到`Packet`我们标记的时间。

​```c#
using System;
using System.Net;

namespace PongGame
{
    // Data structure used to store Packets along with their sender
    public class NetworkMessage
    {
        public IPEndPoint Sender { get; set; }
        public Packet Packet { get; set; }
        public DateTime ReceiveTime { get; set; }
    }
}
```

### ThreadSafe

这是的一个变量（泛型）线程安全（通过锁）。只能通过`Value`属性访问

```c#
using System;

namespace PongGame
{
    // Makes one variable thread safe
    public class ThreadSafe<T>
    {
        // The data & lock
        private T _value;
        private object _lock = new object();

        // How to get & set the data
        public T Value
        {
            get
            {
                lock (_lock)
                    return _value;
            }

            set {
                lock (_lock)
                    _value = value;
            }
        }
            
        // Initializes the value
        public ThreadSafe(T value = default(T))
        {
            Value = value;
        }
    }
}
```

### Packet

确实，代码很多，和文件传输例子中的那个有点相似，除了他有额外的字段代表`Packet`创建的时间。许多子类带有`PacketType`的具体实现。就像之前，你可以使用`GetBytes()`方法以字节数组形式获取通过网络发送的包数据。唯一含有负载的子类是`AcceptJoinPacket`，`PaddlePositionPacket`，`GameStatePacket`和`PlaySoundEffectPacket`。确保注意这些是如何存储数据的。

```c#
using System;
using System.Text;
using System.Linq;
using System.Net;
using System.Net.Sockets;
using Microsoft.Xna.Framework;
using System.Runtime.InteropServices;


namespace PongGame
{
    // These are used to define what each packet means
    // they all have different payloads (if any)
    // Check for subclasses below the `Packet` class
    public enum PacketType : uint
    {
        RequestJoin = 1,    // Client Request to join a game
        AcceptJoin,         // Server accepts join
        AcceptJoinAck,      // Client acknowledges the AcceptJoin
        Heartbeat,          // Client tells Server its alive (before GameStart)
        HeartbeatAck,       // Server acknowledges Client's Heartbeat (before GameStart)
        GameStart,          // Server tells Clients game is starting
        GameStartAck,       // Client acknowledges the GameStart
        PaddlePosition,     // Client tell Server position of the their paddle
        GameState,          // Server tells Clients ball & paddle position, and scores
        PlaySoundEffect,    // Server tells the clinet to play a sound effect
        Bye,                // Either Server or Client tells the other to end the connection
    }

    public class Packet
    {
        // Packet Data
        public PacketType Type;
        public long Timestamp;                  // 64 bit timestamp from DateTime.Ticks 
        public byte[] Payload = new byte[0];

        #region Constructors
        // Creates a Packet with the set type and an empty Payload
        public Packet(PacketType type)
        {
            this.Type = type;
            Timestamp = DateTime.Now.Ticks;
        }

        // Creates a Packet from a byte array
        public Packet(byte[] bytes)
        {
            // Start peeling out the data from the byte array
            int i = 0;

            // Type
            this.Type = (PacketType)BitConverter.ToUInt32(bytes, 0);
            i += sizeof(PacketType);

            // Timestamp
            Timestamp = BitConverter.ToInt64(bytes, i);
            i += sizeof(long);

            // Rest is payload
            Payload = bytes.Skip(i).ToArray();
        }
        #endregion // Constructors

        // Gets the packet as a byte array
        public byte[] GetBytes()
        {
            int ptSize = sizeof(PacketType);
            int tsSize = sizeof(long);

            // Join the Packet data
            int i = 0;
            byte[] bytes = new byte[ptSize + tsSize + Payload.Length];

            // Type
            BitConverter.GetBytes((uint)this.Type).CopyTo(bytes, i);
            i += ptSize;

            // Timestamp
            BitConverter.GetBytes(Timestamp).CopyTo(bytes, i);
            i += tsSize;

            // Payload
            Payload.CopyTo(bytes, i);
            i += Payload.Length;

            return bytes;
        }

        public override string ToString()
        {
            return string.Format("[Packet:{0}\n  timestamp={1}\n  payload size={2}]",
                this.Type, new DateTime(Timestamp), Payload.Length);
        }

        // Sends a Packet to a specific receiver 
        public void Send(UdpClient client, IPEndPoint receiver)
        {
            // TODO maybe be async instead?
            byte[] bytes = GetBytes();
            client.Send(bytes, bytes.Length, receiver);
        }

        // Send a Packet to the default remote receiver (will throw error if not set)
        public void Send(UdpClient client)
        {
            byte[] bytes = GetBytes();
            client.Send(bytes, bytes.Length);
        }
    }

    #region Specific Packets
    // Client Join Request
    public class RequestJoinPacket : Packet
    {
        public RequestJoinPacket()
            : base(PacketType.RequestJoin)
        {
        }
    }

    // Server Accept Request Join, assigns a paddle
    public class AcceptJoinPacket : Packet
    {
        // Paddle side
        public PaddleSide Side {
            get { return (PaddleSide)BitConverter.ToUInt32(Payload, 0); }
            set { Payload = BitConverter.GetBytes((uint)value); }
        }

        public AcceptJoinPacket()
            : base(PacketType.AcceptJoin)
        {
            Payload = new byte[sizeof(PaddleSide)];

            // Set a dfeault paddle of None
            Side = PaddleSide.None;
        }

        public AcceptJoinPacket(byte[] bytes)
            : base(bytes)
        {
        }
    }

    // Ack packet for the one above
    public class AcceptJoinAckPacket : Packet
    {
        public AcceptJoinAckPacket()
            : base(PacketType.AcceptJoinAck)
        {
        }
    }

    // Client tells the Server it's alive
    public class HeartbeatPacket : Packet
    {
        public HeartbeatPacket()
            : base(PacketType.Heartbeat)
        {
        }
    }

    // Server tells the client is knows it's alive
    public class HeartbeatAckPacket : Packet
    {
        public HeartbeatAckPacket()
            : base(PacketType.HeartbeatAck)
        {
        }
    }

    // Tells the client to begin sending data
    public class GameStartPacket : Packet
    {
        public GameStartPacket()
            : base(PacketType.GameStart)
        {
        }
    }

    // Ack for the packet above
    public class GameStartAckPacket : Packet
    {
        public GameStartAckPacket()
            : base(PacketType.GameStartAck)
        {
        }
    }

    // Sent by the client to tell the server it's Y Position for the Paddle
    public class PaddlePositionPacket : Packet
    {
        // The Paddle's Y position
        public float Y {
            get { return BitConverter.ToSingle(Payload, 0); }
            set { BitConverter.GetBytes(value).CopyTo(Payload, 0); }
        }

        public PaddlePositionPacket()
            : base(PacketType.PaddlePosition)
        {
            Payload = new byte[sizeof(float)];

            // Default value is zero
            Y = 0;
        }

        public PaddlePositionPacket(byte[] bytes)
            : base(bytes)
        {
        }

        public override string ToString()
        {
            return string.Format("[Packet:{0}\n  timestamp={1}\n  payload size={2}" +
                "\n  Y={3}]",
                this.Type, new DateTime(Timestamp), Payload.Length, Y);
        }
    }

    // Sent by the server to thd Clients to update the game information
    public class GameStatePacket : Packet
    {
        // Payload array offets
        private static readonly int _leftYIndex = 0;
        private static readonly int _rightYIndex = 4;
        private static readonly int _ballPositionIndex = 8;
        private static readonly int _leftScoreIndex = 16;
        private static readonly int _rightScoreIndex = 20;

        // The Left Paddle's Y position
        public float LeftY
        {
            get { return BitConverter.ToSingle(Payload, _leftYIndex); }
            set { BitConverter.GetBytes(value).CopyTo(Payload, _leftYIndex); }
        }

        // Right Paddle's Y Position
        public float RightY
        {
            get { return BitConverter.ToSingle(Payload, _rightYIndex); }
            set { BitConverter.GetBytes(value).CopyTo(Payload, _rightYIndex); }
        }

        // Ball position
        public Vector2 BallPosition
        {
            get {
                return new Vector2(
                    BitConverter.ToSingle(Payload, _ballPositionIndex),
                    BitConverter.ToSingle(Payload, _ballPositionIndex + sizeof(float))
                );
            }
            set {
                BitConverter.GetBytes(value.X).CopyTo(Payload, _ballPositionIndex);
                BitConverter.GetBytes(value.Y).CopyTo(Payload, _ballPositionIndex + sizeof(float));
            }
        }

        // Left Paddle's Score
        public int LeftScore
        {
            get { return BitConverter.ToInt32(Payload, _leftScoreIndex); }
            set { BitConverter.GetBytes(value).CopyTo(Payload, _leftScoreIndex); }
        }

        // Right Paddle's Score
        public int RightScore
        {
            get { return BitConverter.ToInt32(Payload, _rightScoreIndex); }
            set { BitConverter.GetBytes(value).CopyTo(Payload, _rightScoreIndex); }
        }

        public GameStatePacket()
            : base(PacketType.GameState)
        {
            // Allocate data for the payload (we really shouldn't hardcode this in...)
            Payload = new byte[24];

            // Set default data
            LeftY = 0;
            RightY = 0;
            BallPosition = new Vector2();
            LeftScore = 0;
            RightScore = 0;
        }

        public GameStatePacket(byte[] bytes)
            : base(bytes)
        {
        }

        public override string ToString()
        {
            return string.Format(
                "[Packet:{0}\n  timestamp={1}\n  payload size={2}" +
                "\n  LeftY={3}" +
                "\n  RightY={4}" +
                "\n  BallPosition={5}" +
                "\n  LeftScore={6}" +
                "\n  RightScore={7}]",
                this.Type, new DateTime(Timestamp), Payload.Length, LeftY, RightY, BallPosition, LeftScore, RightScore);
        }
    }

    // Sent by the Server to tell the client they should play a sound effect
    public class PlaySoundEffectPacket : Packet
    {
        public string SFXName {
            get { return Encoding.UTF8.GetString(Payload); }
            set { Payload = Encoding.UTF8.GetBytes(value); }
        }

        public PlaySoundEffectPacket()
            : base(PacketType.PlaySoundEffect)
        {
            SFXName = "";
        }

        public PlaySoundEffectPacket(byte[] bytes)
            : base(bytes)
        {
        }

        public override string ToString()
        {
            return string.Format(
                "[Packet:{0}\n  timestamp={1}\n  payload size={2}" +
                "\n  SFXName={3}",
                this.Type, new DateTime(Timestamp), Payload.Length, SFXName);
        }
    }

    // Sent by either the Client or the Server to end the game/connection
    public class ByePacket : Packet
    {
        public ByePacket()
            : base(PacketType.Bye)
        {
        }
    }
    #endregion  // Specific Packets
}
```

### Ball

球是游戏中的对象，将会被弹来弹去。`LoadContent()`和`Draw()`应该只由客户端程序调用。`Initialize()`和`ServerSideUpdate()`将会被服务端调用。

```c#
using System;
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;
using Microsoft.Xna.Framework.Content;

namespace PongGame
{
    // The ball that's bounced around
    public class Ball
    {
        // Statics
        public static Vector2 InitialSpeed = new Vector2(60f, 60f);

        // Private data members
        private Texture2D _sprite;
        private Random _random = new Random();     // Random Number Generator

        // Public data members
        public Vector2 Position = new Vector2();
        public Vector2 Speed;
        public int LeftmostX { get; private set; }      // Bounds
        public int RightmostX { get; private set; }
        public int TopmostY { get; private set; }
        public int BottommostY { get; private set; }

        // What gets hit
        public Rectangle CollisionArea
        {
            get { return new Rectangle(Position.ToPoint(), GameGeometry.BallSize); }
        }

        public void LoadContent(ContentManager content)
        {
            _sprite = content.Load<Texture2D>("ball.png");
        }

        // this is used to reset the postion of the ball to the center of the board
        public void Initialize()
        {
            // Center the ball
            Rectangle playAreaRect = new Rectangle(new Point(0, 0), GameGeometry.PlayArea);
            Position = playAreaRect.Center.ToVector2();
            Position = Vector2.Subtract(Position, GameGeometry.BallSize.ToVector2() / 2f);

            // Set the velocity
            Speed = InitialSpeed;

            // Randomize direction
            if (_random.Next() % 2 == 1)
                Speed.X *= -1;
            if (_random.Next() % 2 == 1)
                Speed.Y *= -1;

            // Set bounds
            LeftmostX = 0;
            RightmostX = playAreaRect.Width - GameGeometry.BallSize.X;
            TopmostY = 0;
            BottommostY = playAreaRect.Height - GameGeometry.BallSize.Y;
        }

        // Moves the ball, should only be called on the server
        public void ServerSideUpdate(GameTime gameTime)
        {
            float timeDelta = (float)gameTime.ElapsedGameTime.TotalSeconds;

            // Add the distance
            Position = Vector2.Add(Position, timeDelta * Speed);
        }

        // Draws the ball to the screen, only called on the client
        public void Draw(GameTime gameTime, SpriteBatch spriteBatch)
        {
            spriteBatch.Draw(_sprite, Position);
        }
    }
}
```

### Paddle

文件的顶部有两个枚举量，第一个决定了`Paddle`是哪一边的（使用None会引起错误）。另一个是标记`Paddle`的碰撞类型。`LoadContent()`，`Draw()`和`ClientSideUpdate()`应该只有客户端调用。最后一个函数是检测是否用户是否尝试移动板（也做了一些边界检测）。`Initialize()`和`Collides()`应该只由服务端调用。

`Collides()`用来检测球是否与`Paddle`进行了碰撞，如果是的话，函数将会返回true然后传出参数`typeOfCollision`将会被`PaddleCollision`枚举中的一个值填充。

```c#
using System;
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;
using Microsoft.Xna.Framework.Content;
using Microsoft.Xna.Framework.Input;

namespace PongGame
{
    public enum PaddleSide : uint
    {
        None,
        Left,
        Right
    };

    // Type of collision with paddle
    public enum PaddleCollision
    {
        None,
        WithTop,
        WithFront,
        WithBottom
    };

    // This is the Paddle class for the Server
    public class Paddle
    {
        // Private data members
        private Texture2D _sprite;
        private DateTime _lastCollisiontime = DateTime.MinValue;
        private TimeSpan _minCollisionTimeGap = TimeSpan.FromSeconds(0.2);

        // Public data members
        public readonly PaddleSide Side;
        public int Score = 0;
        public Vector2 Position = new Vector2();
        public int TopmostY { get; private set; }               // Bounds
        public int BottommostY { get; private set; }

        #region Collision objects
        public Rectangle TopCollisionArea
        {
            get { return new Rectangle(Position.ToPoint(), new Point(GameGeometry.PaddleSize.X, 4)); }
        }

        public Rectangle BottomCollisionArea
        {
            get
            {
                return new Rectangle(
                    (int)Position.X, FrontCollisionArea.Bottom,
                    GameGeometry.PaddleSize.X, 4
                );
            }
        }

        public Rectangle FrontCollisionArea
        {
            get
            {
                Point pos = Position.ToPoint();
                pos.Y += 4;
                Point size = new Point(GameGeometry.PaddleSize.X, GameGeometry.PaddleSize.Y - 8);

                return new Rectangle(pos, size);
            }
        }
        #endregion // Collision objects

        // Sets which side the paddle is
        public Paddle(PaddleSide side)
        {
            Side = side;
        }

        public void LoadContent(ContentManager content)
        {
            _sprite = content.Load<Texture2D>("paddle.png");
        }

        // Puts the paddle in the middle of where it can move
        public void Initialize()
        {
            // Figure out where to place the paddle
            int x;
            if (Side == PaddleSide.Left)
                x = GameGeometry.GoalSize;
            else if (Side == PaddleSide.Right)
                x = GameGeometry.PlayArea.X - GameGeometry.PaddleSize.X - GameGeometry.GoalSize;
            else
                throw new Exception("Side is not `Left` or `Right`");

            Position = new Vector2(x, (GameGeometry.PlayArea.Y / 2) - (GameGeometry.PaddleSize.Y / 2));
            Score = 0;

            // Set bounds
            TopmostY = 0;
            BottommostY = GameGeometry.PlayArea.Y - GameGeometry.PaddleSize.Y;
        }

        // Moves the paddle based on user input (called on Client)
        public void ClientSideUpdate(GameTime gameTime)
        {
            float timeDelta = (float)gameTime.ElapsedGameTime.TotalSeconds;
            float dist = timeDelta * GameGeometry.PaddleSpeed;

            // Check Up & Down keys
            KeyboardState kbs = Keyboard.GetState();
            if (kbs.IsKeyDown(Keys.Up))
                Position.Y -= dist;
            else if (kbs.IsKeyDown(Keys.Down))
                Position.Y += dist;

            // bounds checking
            if (Position.Y < TopmostY)
                Position.Y = TopmostY;
            else if (Position.Y > BottommostY)
                Position.Y = BottommostY;
        }

        public void Draw(GameTime gameTime, SpriteBatch spriteBatch)
        {
            spriteBatch.Draw(_sprite, Position);
        }

        // Sees what part of the Paddle collises with the ball (if it does)
        public bool Collides(Ball ball, out PaddleCollision typeOfCollision)
        {
            typeOfCollision = PaddleCollision.None;

            // Make sure enough time has passed for a new collisions
            // (this prevents a bug where a user can build up a lot of speed in the ball)
            if (DateTime.Now < (_lastCollisiontime.Add(_minCollisionTimeGap)))
                return false;

            // Top & bottom get first priority
            if (ball.CollisionArea.Intersects(TopCollisionArea))
            {
                typeOfCollision =  PaddleCollision.WithTop;
                _lastCollisiontime = DateTime.Now;
                return true;
            }

            if (ball.CollisionArea.Intersects(BottomCollisionArea))
            {
                typeOfCollision =  PaddleCollision.WithBottom;
                _lastCollisiontime = DateTime.Now;
                return true;
            }

            // And check the front
            if (ball.CollisionArea.Intersects(FrontCollisionArea))
            {
                typeOfCollision =  PaddleCollision.WithFront;
                _lastCollisiontime = DateTime.Now;
                return true;
            }

            // Nope, nothing
            return false;
        }
    }
}
```

### UDP Pong - Server

在开始之前，我想提醒一下，`PongServer.cs`中没有游戏逻辑的成分，代码在`Arena.cs`中，因此你会看到一个叫做Arena的类，它的代码在下一页。

### PlayerInfo

这是另一个数据类型只在服务器中使用，它包含这连接客户端的信息和他们目前状态

```c#
using System;
using System.Net;

namespace PongGame
{
    // Data structure for the server to manage clients
    public class PlayerInfo
    {
        public Paddle Paddle;
        public IPEndPoint Endpoint;
        public DateTime LastPacketReceivedTime = DateTime.MinValue;     // From Server Time
        public DateTime LastPacketSentTime = DateTime.MinValue;         // From Server Time
        public long LastPacketReceivedTimestamp = 0;                    // From Client Time
        public bool HavePaddle = false;
        public bool Ready = false;

        public bool IsSet {
            get { return Endpoint != null; }
        }
    }
}
```

三个处理时间的字段看着有点令人困惑，让我来解释一下

- `LastPacketReceivedTime` - 这代表我们从一个客户端获取一个`Packet`的时间，它是通过`DateTime.Now`获取的，在函数`PongServer._networkRun()`。它用来检测超时。
- `LastPacketSentTime` - 这代表我们最后一次发送`Packet`给客户端的时间。它记录在`Arena._sendTo()`方法中。着用来确保我们不会太频繁的给客户端发送消息。
- `LastPacketReceivedTimestamp` - 不行其他的两个，这个是用来测量客户端的时间的，来自于`Packert.Timestamp`字段。当设置他的时候，它的值只能比之前更高，用来确保我们使用的是客户端的最新消息。因为在UDP中，一些较晚发送的数据报可能比较早发送的报更早到达，这会导致旧的报被遗弃。

### PongServer

`PongServer`的职责只是用来从网络中读取数据，将其写入客户端，管理`Arena`

```c#
using System;
using System.Linq;
using System.Collections.Generic;
using System.Collections.Concurrent;
using System.Net;
using System.Net.Sockets;
using System.Threading;

namespace PongGame
{
    public class PongServer
    {
        // Network Stuff
        private UdpClient _udpClient;
        public readonly int Port;

        // Messaging
        Thread _networkThread;
        private ConcurrentQueue<NetworkMessage> _incomingMessages
            = new ConcurrentQueue<NetworkMessage>();
        private ConcurrentQueue<Tuple<Packet, IPEndPoint>> _outgoingMessages
            = new ConcurrentQueue<Tuple<Packet, IPEndPoint>>();
        private ConcurrentQueue<IPEndPoint> _sendByePacketTo
            = new ConcurrentQueue<IPEndPoint>();

        // Arena management
        private ConcurrentDictionary<Arena, byte> _activeArenas             // Being used as a HashSet
            = new ConcurrentDictionary<Arena, byte>();
        private ConcurrentDictionary<IPEndPoint, Arena> _playerToArenaMap
            = new ConcurrentDictionary<IPEndPoint, Arena>();
        private Arena _nextArena;

        // Used to check if we are running the server or not
        private ThreadSafe<bool> _running = new ThreadSafe<bool>(false);

        public PongServer(int port)
        {
            Port = port;

            // Create the UDP socket (IPv4)
            _udpClient = new UdpClient(Port, AddressFamily.InterNetwork);
        }

        // Notifies that we can start the server
        public void Start()
        {
            _running.Value = true;
        }

        // Starts a shutdown of the server
        public void Shutdown()
        {
            if (_running.Value)
            {
                Console.WriteLine("[Server] Shutdown requested by user.");

                // Close any active games
                Queue<Arena> arenas = new Queue<Arena>(_activeArenas.Keys);
                foreach (Arena arena in arenas)
                    arena.Stop();

                // Stops the network thread
                _running.Value = false;
            }
        }

        // Cleans up any necessary resources
        public void Close()
        {
            _networkThread?.Join(TimeSpan.FromSeconds(10));
            _udpClient.Close();
        }

        // Small lambda function to add a new arena
        private void _addNewArena ()
        {
            _nextArena = new Arena(this);
            _nextArena.Start();
            _activeArenas.TryAdd(_nextArena, 0);
        }

        // Used by an Arena to notify the PingServer that it's done
        public void NotifyDone(Arena arena)
        {
            // First remove from the Player->Arena map
            Arena a;
            if (arena.LeftPlayer.IsSet)
                _playerToArenaMap.TryRemove(arena.LeftPlayer.Endpoint, out a);
            if (arena.RightPlayer.IsSet)
                _playerToArenaMap.TryRemove(arena.RightPlayer.Endpoint, out a);

            // Remove from the active games hashset
            byte b;
            _activeArenas.TryRemove(arena, out b);
        }

        // Main loop function for the server
        public void Run()
        {
            // Make sure we've called Start()
            if (_running.Value)
            {
                // Info
                Console.WriteLine("[Server] Running Ping Game");

                // Start the packet receiving Thread
                _networkThread = new Thread(new ThreadStart(_networkRun));
                _networkThread.Start();

                // Startup the first Arena
                _addNewArena();
            }

            // Main loop of game server
            bool running = _running.Value;
            while (running)
            {
                // If we have some messages in the queue, pull them out
                NetworkMessage nm;
                bool have = _incomingMessages.TryDequeue(out nm);
                if (have)
                {
                    // Depending on what type of packet it is process it
                    if (nm.Packet.Type == PacketType.RequestJoin)
                    {
                        // We have a new client, put them into an arena
                        bool added = _nextArena.TryAddPlayer(nm.Sender);
                        if (added)
                            _playerToArenaMap.TryAdd(nm.Sender, _nextArena);

                        // If they didn't go in that means we're full, make a new arena
                        if (!added)
                        {
                            _addNewArena();

                            // Now there should be room
                            _nextArena.TryAddPlayer(nm.Sender);
                            _playerToArenaMap.TryAdd(nm.Sender, _nextArena);
                        }

                        // Dispatch the message
                        _nextArena.EnqueMessage(nm);
                    }
                    else
                    {
                        // Dispatch it to an existing arena
                        Arena arena;
                        if (_playerToArenaMap.TryGetValue(nm.Sender, out arena))
                            arena.EnqueMessage(nm);
                    }
                }
                else
                    Thread.Sleep(1);    // Take a short nap if there are no messages

                // Check for quit
                running &= _running.Value;
            }
        }

        #region Network Functions
        // This function is meant to be run in its own thread
        // Is writes and reads Packets to/from the UdpClient
        private void _networkRun()
        {
            if (!_running.Value)
                return;
             
            Console.WriteLine("[Server] Waiting for UDP datagrams on port {0}", Port);

            while (_running.Value)
            {
                bool canRead = _udpClient.Available > 0;
                int numToWrite = _outgoingMessages.Count;
                int numToDisconnect = _sendByePacketTo.Count;

                // Get data if there is some
                if (canRead)
                {
                    // Read in one datagram
                    IPEndPoint ep = new IPEndPoint(IPAddress.Any, 0);
                    byte[] data = _udpClient.Receive(ref ep);              // Blocks

                    // Enque a new message
                    NetworkMessage nm = new NetworkMessage();
                    nm.Sender = ep;
                    nm.Packet = new Packet(data);
                    nm.ReceiveTime = DateTime.Now;

                    _incomingMessages.Enqueue(nm);

                    //Console.WriteLine("RCVD: {0}", nm.Packet);
                }

                // Write out queued
                for (int i = 0; i < numToWrite; i++)
                {
                    // Send some data
                    Tuple<Packet, IPEndPoint> msg;
                    bool have = _outgoingMessages.TryDequeue(out msg);
                    if (have)
                        msg.Item1.Send(_udpClient, msg.Item2);

                    //Console.WriteLine("SENT: {0}", msg.Item1);
                }

                // Notify clients of Bye
                for (int i = 0; i < numToDisconnect; i++)
                {
                    IPEndPoint to;
                    bool have = _sendByePacketTo.TryDequeue(out to);
                    if (have)
                    {
                        ByePacket bp = new ByePacket();
                        bp.Send(_udpClient, to);
                    }
                }

                // If Nothing happened, take a nap
                if (!canRead && (numToWrite == 0) && (numToDisconnect == 0))
                    Thread.Sleep(1);
            }

            Console.WriteLine("[Server] Done listening for UDP datagrams");

            // Wait for all arena's thread to join
            Queue<Arena> arenas = new Queue<Arena>(_activeArenas.Keys);
            if (arenas.Count > 0)
            {
                Console.WriteLine("[Server] Waiting for active Areans to finish...");
                foreach (Arena arena in arenas)
                    arena.JoinThread();
            }

            // See which clients are left to notify of Bye
            if (_sendByePacketTo.Count > 0)
            {
                Console.WriteLine("[Server] Notifying remaining clients of shutdown...");

                // run in a loop until we've told everyone else
                IPEndPoint to;
                bool have = _sendByePacketTo.TryDequeue(out to);
                while (have)
                {
                    ByePacket bp = new ByePacket();
                    bp.Send(_udpClient, to);
                    have = _sendByePacketTo.TryDequeue(out to);
                }
            }
        }

        // Queues up a Packet to be send to another person
        public void SendPacket(Packet packet, IPEndPoint to)
        {
            _outgoingMessages.Enqueue(new Tuple<Packet, IPEndPoint>(packet, to));
        }

        // Will queue to send a ByePacket to the specified endpoint
        public void SendBye(IPEndPoint to)
        {
            _sendByePacketTo.Enqueue(to);
        }
        #endregion  // Network Functions





        #region Program Execution
        public static PongServer server;

        public static void InterruptHandler(object sender, ConsoleCancelEventArgs args)
        {
            // Do a graceful shutdown
            args.Cancel = true;
            server?.Shutdown();
        }

        public static void Main(string[] args)
        {
            // Setup the server
            int port = 6000;//int.Parse(args[0].Trim());
            server = new PongServer(port);

            // Add the Ctrl-C handler
            Console.CancelKeyPress += InterruptHandler;

            // Run it
            server.Start();
            server.Run();
            server.Close();
        }
        #endregion // Program Execution
    }
}
```

让我们解释这里发生了什么。

在类的顶部，我们有一些 [ConcurrentQueues](https://msdn.microsoft.com/en-us/library/dd267265(v=vs.110).aspx) 和 [ConcurrentDictionaries](https://msdn.microsoft.com/en-us/library/dd287191(v=vs.110).aspx)。前三个用来处理消息。注意一下，`_activeArenas`用作 [HashSet](https://msdn.microsoft.com/en-us/library/bb359438(v=vs.110).aspx)。因为 collection没有concurrent的hashset。

构造函数的作用是用来设定`Port`的号码，然后初始化底层的 [UdpClient](http://msdn.microsoft.com/en-us/library/system.net.sockets.udpclient(v=vs.110).aspx)。用来监听是由IPv6连接。

`Start()`用来标记我们的服务器可以开始处理客户端。`Shutdown()`则相反。如果服务器早在运行，它会告诉运行的`Arena`来结束游戏。`Close()`将会清除所有资源。

`AddNewArena()`是一个小的帮助函数开启一个新的`Arena`实例（也在是其他的线程中启动）然后把它放到`_nextArena`变量中。

`NotifyDone()`仅仅由`Arena`调用。它告诉`PongServer`从`_playerToArenaMap`移除任何连接的客户端，并且`_activeArenas`中移除自己

`Run()`是`PongServer`中的主方法。它只会在之前已经调用过`Start()`之后才会执行。首先，他会创建一个新的`Thread`来解决所有的即将到来和即将发出的消息（通过网络），然后实例化第一个`Arena`。在`while(running)`循环中，他会检测我们是否获得了新的`NetworkMessage`。

- 如果一个`Packet`是`RequestJoin`，他将会将其加入到下一个运行的`Arena`中。如果`_nextArena`已经满了（`TryAdd()`返回`false`），他会调用`AddArena()`然后重新试着将其加入。
- 但如果`Packet`如果不是`RequestJoin`，他将会将他发送到`Sender`所在的`Arena`。
  - 如果你纳闷如果`Sender`不在运行的`Arena`中的情况会发生什么，消息会遗弃。

在循环的结束`Thread.Sleep()`将会调用来节省CPU资源然后检查服务器是否仍然运行。

`NetworkRun()`是另一个重要的循环函数。他在自己的独立线程中运行(`_networkThread`)。当我们不告诉`PongServer`停止，他会检查是否有数据报等待我们，并写出我们已经在排队的`Packet`（从`_outgoingMessages`和`_sendByPacketTo`）。在没有任何数据接收和发送的情况下，我们的函数休息一会。当结束监听消息的时候（即`PongServer`关闭），他会让所有活跃的`Arena`停止他们的线程。自此之后，如果有任何连接的客户端，给他们发送`ByePacket`消息

`SendPacket()`和`SendBye()`在功能上是基本相同的，将消息排队发送给客户端。唯一的区别是后一个方法用来和客户端断开连接。

在文件的底部，我们有一些程序执行的硬编码代码。在`Main()`方法中，我们创建`PongServer`实例，然后增加按压Ctrl-C事件。

### UDP Pong - Arena

这是服务器上弹球游戏的逻辑

```c#
using System;
using System.Threading;
using System.Net;
using System.Net.Sockets;
using System.Diagnostics;
using System.Collections.Concurrent;
using Microsoft.Xna.Framework;

namespace PongGame
{
    public enum ArenaState
    {
        NotRunning,
        WaitingForPlayers,
        NotifyingGameStart,
        InGame,
        GameOver
    }

    // This is where the game takes place
    public class Arena
    {
        // Game objects & state info
        public ThreadSafe<ArenaState> State { get; private set; } = new ThreadSafe<ArenaState>();
        private Ball _ball = new Ball();
        public PlayerInfo LeftPlayer { get; private set; } = new PlayerInfo();      // contains Paddle
        public PlayerInfo RightPlayer { get; private set; } = new PlayerInfo();     // contains Paddle
        private object _setPlayerLock = new object();
        private Stopwatch _gameTimer = new Stopwatch();

        // Connection info
        private PongServer _server;
        private TimeSpan _heartbeatTimeout = TimeSpan.FromSeconds(20);

        // Packet queue
        private ConcurrentQueue<NetworkMessage> _messages = new ConcurrentQueue<NetworkMessage>();

        // Shutdown data
        private ThreadSafe<bool> _stopRequested = new ThreadSafe<bool>(false);

        // Other
        private Thread _arenaThread;
        private Random _random = new Random();
        public readonly int Id;
        private static int _nextId = 1;

        public Arena(PongServer server)
        {
            _server = server;
            Id = _nextId++;
            State.Value = ArenaState.NotRunning;

            // Some other data
            LeftPlayer.Paddle = new Paddle(PaddleSide.Left);
            RightPlayer.Paddle = new Paddle(PaddleSide.Right);
        }

        // Returns true if the player was added,
        // false otherwise (max two players will be accepeted)
        public bool TryAddPlayer(IPEndPoint playerIP)
        {
            if (State.Value == ArenaState.WaitingForPlayers)
            {
                lock (_setPlayerLock)
                {
                    // First do the left
                    if (!LeftPlayer.IsSet)
                    {
                        LeftPlayer.Endpoint = playerIP;
                        return true;
                    }

                    // Then the Right
                    if (!RightPlayer.IsSet)
                    {
                        RightPlayer.Endpoint = playerIP;
                        return true;
                    }
                }
            }

            // Couldn't add any more
            return false;
        }

        // Initializes the game objects to a default state and start a new Thread
        public void Start()
        {
            // Shift the state
            State.Value = ArenaState.WaitingForPlayers;

            // Start the internal thread on Run
            _arenaThread = new Thread(new ThreadStart(_arenaRun));
            _arenaThread.Start();
        }

        // Tells the game to stop
        public void Stop()
        {
            _stopRequested.Value = true;
        }

        // This runs in its own Thread
        // It is the actual game
        private void _arenaRun()
        {
            Console.WriteLine("[{0:000}] Waiting for players", Id);
            GameTime gameTime = new GameTime();

            // Varibables used in the switch
            TimeSpan notifyGameStartTimeout = TimeSpan.FromSeconds(2.5);
            TimeSpan sendGameStateTimeout = TimeSpan.FromMilliseconds(1000f / 30f);  // How often to update the players

            // The loop
            bool running = true;
            bool playerDropped = false;
            while (running)
            {
                // Pop off a message (if there is one)
                NetworkMessage message;
                bool haveMsg = _messages.TryDequeue(out message);

                switch (State.Value)
                {
                    case ArenaState.WaitingForPlayers:
                        if (haveMsg)
                        {
                            // Wait until we have two players
                            _handleConnectionSetup(LeftPlayer, message);
                            _handleConnectionSetup(RightPlayer, message);

                            // Check if we are ready or not
                            if (LeftPlayer.HavePaddle && RightPlayer.HavePaddle)
                            {
                                // Try sending the GameStart packet immediately
                                _notifyGameStart(LeftPlayer, new TimeSpan());
                                _notifyGameStart(RightPlayer, new TimeSpan());

                                // Shift the state
                                State.Value = ArenaState.NotifyingGameStart;
                            }
                        }
                        break;

                    case ArenaState.NotifyingGameStart:
                        // Try sending the GameStart packet
                        _notifyGameStart(LeftPlayer, notifyGameStartTimeout);
                        _notifyGameStart(RightPlayer, notifyGameStartTimeout);

                        // Check for ACK
                        if (haveMsg && (message.Packet.Type == PacketType.GameStartAck))
                        {
                            // Mark true for those who have sent something
                            if (message.Sender.Equals(LeftPlayer.Endpoint))
                                LeftPlayer.Ready = true;
                            else if (message.Sender.Equals(RightPlayer.Endpoint))
                                RightPlayer.Ready = true;
                        }

                        // Are we ready to send/received game data?
                        if (LeftPlayer.Ready && RightPlayer.Ready)
                        {
                            // Initlize some game object positions
                            _ball.Initialize();
                            LeftPlayer.Paddle.Initialize();
                            RightPlayer.Paddle.Initialize();

                            // Send a basic game state
                            _sendGameState(LeftPlayer, new TimeSpan());
                            _sendGameState(RightPlayer, new TimeSpan());

                            // Start the game timer
                            State.Value = ArenaState.InGame;
                            Console.WriteLine("[{0:000}] Starting Game", Id);
                            _gameTimer.Start();
                        }

                        break;

                    case ArenaState.InGame:
                        // Update the game timer
                        TimeSpan now = _gameTimer.Elapsed;
                        gameTime = new GameTime(now, now - gameTime.TotalGameTime);

                        // Get paddle postions from clients
                        if (haveMsg)
                        {
                            switch (message.Packet.Type)
                            {
                                case PacketType.PaddlePosition:
                                    _handlePaddleUpdate(message);
                                    break;

                                case PacketType.Heartbeat:
                                    // Respond with an ACK
                                    HeartbeatAckPacket hap = new HeartbeatAckPacket();
                                    PlayerInfo player = message.Sender.Equals(LeftPlayer.Endpoint) ? LeftPlayer : RightPlayer;
                                    _sendTo(player, hap);

                                    // Record time
                                    player.LastPacketReceivedTime = message.ReceiveTime;
                                    break;
                            }
                        }

                        //Update the game components
                        _ball.ServerSideUpdate(gameTime);
                        _checkForBallCollisions();

                        // Send the data
                        _sendGameState(LeftPlayer, sendGameStateTimeout);
                        _sendGameState(RightPlayer, sendGameStateTimeout);
                        break;
                }

                // Check for a quit from one of the clients
                if (haveMsg && (message.Packet.Type == PacketType.Bye))
                {
                    // Well, someone dropped
                    PlayerInfo player = message.Sender.Equals(LeftPlayer.Endpoint) ? LeftPlayer : RightPlayer;
                    running = false;
                    Console.WriteLine("[{0:000}] Quit detected from {1} at {2}",
                        Id, player.Paddle.Side, _gameTimer.Elapsed);

                    // Tell the other one
                    if (player.Paddle.Side == PaddleSide.Left)
                    {
                        // Left Quit, tell Right
                        if (RightPlayer.IsSet)
                            _server.SendBye(RightPlayer.Endpoint);
                    }
                    else
                    {
                        // Right Quit, tell Left
                        if (LeftPlayer.IsSet)
                            _server.SendBye(LeftPlayer.Endpoint);
                    }
                }

                // Check for timeouts
                playerDropped |= _timedOut(LeftPlayer);
                playerDropped |= _timedOut(RightPlayer);

                // Small nap
                Thread.Sleep(1);

                // Check quit values
                running &= !_stopRequested.Value;
                running &= !playerDropped;
            }

            // End the game
            _gameTimer.Stop();
            State.Value = ArenaState.GameOver;
            Console.WriteLine("[{0:000}] Game Over, total game time was {1}", Id, _gameTimer.Elapsed);

            // If the stop was requested, gracefully tell the players to quit
            if (_stopRequested.Value)
            {
                Console.WriteLine("[{0:000}] Notifying Players of server shutdown", Id);

                if (LeftPlayer.IsSet)
                    _server.SendBye(LeftPlayer.Endpoint);
                if (RightPlayer.IsSet)
                    _server.SendBye(RightPlayer.Endpoint);
            }

            // Tell the server that we're finished
            _server.NotifyDone(this);
        }

        // Gives the underlying thread 1/10 a second to finish
        public void JoinThread()
        {
            _arenaThread.Join(100);
        }

        // This is called by the server to dispatch messages to this Arena
        public void EnqueMessage(NetworkMessage nm)
        {
            _messages.Enqueue(nm);
        }

        #region Network Functions
        // Sends a packet to a player and marks other info
        private void _sendTo(PlayerInfo player, Packet packet)
        {
            _server.SendPacket(packet, player.Endpoint);
            player.LastPacketSentTime = DateTime.Now;
        }

        // Returns true if a player has timed out or not
        // If we haven't recieved a heartbeat at all from them, they're not timed out
        private bool _timedOut(PlayerInfo player)
        {
            // We haven't recorded it yet
            if (player.LastPacketReceivedTime == DateTime.MinValue)
                return false;    

            // Do math
            bool timeoutDetected = (DateTime.Now > (player.LastPacketReceivedTime.Add(_heartbeatTimeout)));
            if (timeoutDetected)
                Console.WriteLine("[{0:000}] Timeout detected on {1} Player at {2}", Id, player.Paddle.Side, _gameTimer.Elapsed);

            return timeoutDetected;
        }

        // This will Handle the initial connection setup and Heartbeats of a client
        private void _handleConnectionSetup(PlayerInfo player, NetworkMessage message)
        {
            // Make sure the message is from the correct client provided
            bool sentByPlayer = message.Sender.Equals(player.Endpoint);
            if (sentByPlayer)
            {
                // Record the last time we've heard from them
                player.LastPacketReceivedTime = message.ReceiveTime;

                // Do they need their Side? or a heartbeat ACK
                switch (message.Packet.Type)
                {
                    case PacketType.RequestJoin:
                        Console.WriteLine("[{0:000}] Join Request from {1}", Id, player.Endpoint);
                        _sendAcceptJoin(player);
                        break;

                    case PacketType.AcceptJoinAck:
                        // They acknowledged (they will send heartbeats until game start)
                        player.HavePaddle = true;
                        break;

                    case PacketType.Heartbeat:
                        // They are waiting for the game start, Respond with an ACK
                        HeartbeatAckPacket hap = new HeartbeatAckPacket();
                        _sendTo(player, hap);

                        // Incase their ACK didn't reach us
                        if (!player.HavePaddle)
                            _sendAcceptJoin(player);

                        break;
                }
            }
        }

        // Sends an AcceptJoinPacket to a player
        public void _sendAcceptJoin(PlayerInfo player)
        {
            // They need to know which paddle they are
            AcceptJoinPacket ajp = new AcceptJoinPacket();
            ajp.Side = player.Paddle.Side;
            _sendTo(player, ajp);
        }

        // Tries to notify the players of a GameStart
        // retryTimeout is how long to wait until to resending the packet
        private void _notifyGameStart(PlayerInfo player, TimeSpan retryTimeout)
        {
            // check if they are ready already
            if (player.Ready)
                return;

            // Make sure not to spam them
            if (DateTime.Now >= (player.LastPacketSentTime.Add(retryTimeout)))
            {
                GameStartPacket gsp = new GameStartPacket();
                _sendTo(player, gsp);
            }
        }

        // Sends information about the current game state to the players
        // resendTimeout is how long to wait until sending another GameStatePacket
        private void _sendGameState(PlayerInfo player, TimeSpan resendTimeout)
        {
            if (DateTime.Now >= (player.LastPacketSentTime.Add(resendTimeout)))
            {
                // Set the data
                GameStatePacket gsp = new GameStatePacket();
                gsp.LeftY = LeftPlayer.Paddle.Position.Y;
                gsp.RightY = RightPlayer.Paddle.Position.Y;
                gsp.BallPosition = _ball.Position;
                gsp.LeftScore = LeftPlayer.Paddle.Score;
                gsp.RightScore = RightPlayer.Paddle.Score;

                _sendTo(player, gsp);
            }
        }

        // Tells both of the clients to play a sound effect
        private void _playSoundEffect(string sfxName)
        {
            // Make the packet
            PlaySoundEffectPacket packet = new PlaySoundEffectPacket();
            packet.SFXName = sfxName;

            _sendTo(LeftPlayer, packet);
            _sendTo(RightPlayer, packet);
        }

        // This updates a paddle's postion from a client
        // `message.Packet.Type` must be `PacketType.PaddlePosition`
        // TODO add some "cheat detection,"
        private void _handlePaddleUpdate(NetworkMessage message)
        {
            // Only two possible players
            PlayerInfo player = message.Sender.Equals(LeftPlayer.Endpoint) ? LeftPlayer : RightPlayer;

            // Make sure we use the latest message **SENT BY THE CLIENT**  ignore it otherwise
            if (message.Packet.Timestamp > player.LastPacketReceivedTimestamp)
            {
                // record timestamp and time
                player.LastPacketReceivedTimestamp = message.Packet.Timestamp;
                player.LastPacketReceivedTime = message.ReceiveTime;

                // "cast" the packet and set data
                PaddlePositionPacket ppp = new PaddlePositionPacket(message.Packet.GetBytes());
                player.Paddle.Position.Y = ppp.Y;
            }
        }
        #endregion // Network Functions

        #region Collision Methods
        // Does all of the collision logic for the ball (including scores)
        private void _checkForBallCollisions()
        {
            // Top/Bottom
            float ballY = _ball.Position.Y;
            if ((ballY <= _ball.TopmostY) || (ballY >= _ball.BottommostY))
            {
                _ball.Speed.Y *= -1;
                _playSoundEffect("ball-hit");
            }

            // Ball left and right (the goals!)
            float ballX = _ball.Position.X;
            if (ballX <= _ball.LeftmostX)
            {
                // Right player scores! (reset ball)
                RightPlayer.Paddle.Score += 1;
                Console.WriteLine("[{0:000}] Right Player scored ({1} -- {2}) at {3}",
                    Id, LeftPlayer.Paddle.Score, RightPlayer.Paddle.Score, _gameTimer.Elapsed);
                _ball.Initialize();
                _playSoundEffect("score");
            }
            else if (ballX >= _ball.RightmostX)
            {
                // Left palyer scores! (reset ball)
                LeftPlayer.Paddle.Score += 1;
                Console.WriteLine("[{0:000}] Left Player scored ({1} -- {2}) at {3}",
                    Id, LeftPlayer.Paddle.Score, RightPlayer.Paddle.Score, _gameTimer.Elapsed);
                _ball.Initialize();
                _playSoundEffect("score");
            }

            // Ball with paddles
            PaddleCollision collision;
            if (LeftPlayer.Paddle.Collides(_ball, out collision))
                _processBallHitWithPaddle(collision);
            if (RightPlayer.Paddle.Collides(_ball, out collision))
                _processBallHitWithPaddle(collision);
            
        }

        // Modifies the ball state based on what the collision is
        private void _processBallHitWithPaddle(PaddleCollision collision)
        {
            // Safety check
            if (collision == PaddleCollision.None)
                return;

            // Increase the speed
            _ball.Speed.X *= _map((float)_random.NextDouble(), 0, 1, 1, 1.25f);
            _ball.Speed.Y *= _map((float)_random.NextDouble(), 0, 1, 1, 1.25f);

            // Shoot in the opposite direction
            _ball.Speed.X *= -1;

            // Hit with top or bottom?
            if ((collision == PaddleCollision.WithTop) || (collision == PaddleCollision.WithBottom))
                _ball.Speed.Y *= -1;

            // Play a sound on the client
            _playSoundEffect("ballHit");
        }
        #endregion // Collision Methods

        // Small utility function that maps one value range to another
        private float _map(float x, float a, float b, float p, float q)
        {
            return p + (x - a) * (q - p) / (b - a);
        }
    }
}
```

在类的顶端，我们定义了一些成员变量。`State`需要被封装在`ThreadSafe`中，因为它可能被多个线程访问。`Paddle`对象包含在`PlayerInfo`类中。我们存储了一个引用来运行服务器实例，对于已经分配给我们的`Packet`，我们也有自己`的消息队列。

当我们用一个新的客户端来连接服务器的时候会调用`TryAddPlayer()`，一个`Arena`中只会加入两个玩家。如果客户端添加成功会返回`true`，否则返回`false`。

`Start()`会转换`Arena`的`State`至可以接受新玩家的状态，然后开始运行运行`ArenaRun()`的`_arenaThread`方法。

`ArenaRun()`是类的核心，它是另一个循环执行的函数。在函数的开头，他检查有无新的`NetworkMessage`然后根据他目前的`State`来做决定。当我们等待两个玩家的时候，他首先会尝试设置连接，一旦两个玩家连接上了，给他们发送`GameStart Packet`。一旦`GameStart Packet`被客户端确认，它将会把状态转化为`InGame`，开始`_gameTimer`，然后开始处理`PaddlePosition`和`GameState Packet`。如果一个客户端发送了`Bye`消息，便会停止循环然后通知另一个客户端游戏结束了。在循环的结尾，他会检测玩家是否超时或者服务器是否请求结束，一旦循环退出了，他会排队给连接的客户端发送`ByePacket`然后通知游戏结束了。

`JoinThread()`会给底层的`_arenaThread`100毫秒来完成执行。

`EnqueMessage()`被`PongServer`调用来告诉`Arena`他从一个客户端得到了一些东西

`SendTo()`设置一个发送给`Client`的`Packet`。他会记录我们想要发送他的时间

`TimeOut()`用来检查我们最后一次获取`Packet`的时间和多久就会超时（我们这里是20秒）的对比

`HandleConnection()`是实例化客户端和服务器的连接（尽管保存在`Arena`代码中）。他会给客户端分配一个`Paddle`并且回应`Heartbeat`消息

`SendAcceptJoin()`是一个小的帮助函数来告诉客户端他们是哪一个`Paddle`。

`NotifyGameStart(）`会告诉客户端游戏开始了，然后会在每次超时的时候重试。

`SendGameState()`会告诉客户端当前比赛的状态，`Paddle`的位置，得分，`Ball`的位置。

`PlaySoundEffect()`会排队给客户端发送`PlaySoundEffectPacket`。

`HandlePaddleUpdate()`会查看客户端发来的`PaddlePosition`信息然后更新`Paddle`的Y轴位置

下面两个方法是为了碰撞检测的。`CheckForBallCollision()`会在比赛全程运行，检查`Ball`是否击中了东西，然后采取相应措施（弹跳，得分，声音等等）。`ProcessBallHitWithPaddle()`用来转化`Ball`碰撞后的速度。

### UDP Pong - Client

这是我们发送给用户整个程序中最有趣的部分，他们在这里可以控制`Paddle`上下。这也是应用程序的图形部分。在此之前，我们只是使用MonoGame/XNA进行碰撞检测，这里我们用它做它本来的目的（游戏）

不要忘记，如果你想要声音奏效，你需要取消注释，记住声音在MonoGame 3.5 (DesktopGL version)中并不能正常工作。

```c#
// Uncomment the line below to play sounds
// This is here because MonoGame 3.5 on Linux throws an error when trying to play a SoundEffect
//#define CAN_PLAY_SOUNDS

using System;
using System.Threading;
using System.Net;
using System.Net.Sockets;
using System.Collections.Concurrent;
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Input;
using Microsoft.Xna.Framework.Graphics;
using Microsoft.Xna.Framework.Audio;

namespace PongGame
{
    public enum ClientState
    {
        NotConnected,
        EstablishingConnection,
        WaitingForGameStart,
        InGame,
        GameOver,
    }

    class PongClient : Game
    {
        // Game stuffs
        private GraphicsDeviceManager _graphics;
        private SpriteBatch _spriteBatch;

        // Network stuff
        private UdpClient _udpClient;
        public readonly string ServerHostname;
        public readonly int ServerPort;

        // Time measurement
        private DateTime _lastPacketReceivedTime = DateTime.MinValue;     // From Client Time
        private DateTime _lastPacketSentTime = DateTime.MinValue;         // From Client Time
        private long _lastPacketReceivedTimestamp = 0;                    // From Server Time
        private TimeSpan _heartbeatTimeout = TimeSpan.FromSeconds(20);
        private TimeSpan _sendPaddlePositionTimeout = TimeSpan.FromMilliseconds(1000f / 30f);  // How often to update the server

        // Messaging
        private Thread _networkThread;
        private ConcurrentQueue<NetworkMessage> _incomingMessages
            = new ConcurrentQueue<NetworkMessage>();
        private ConcurrentQueue<Packet> _outgoingMessages
            = new ConcurrentQueue<Packet>();

        // Game objects
        private Ball _ball;
        private Paddle _left;
        private Paddle _right;
        private Paddle _ourPaddle;
        private float _previousY;

        // Info messages for the user
        private Texture2D _establishingConnectionMsg;
        private Texture2D _waitingForGameStartMsg;
        private Texture2D _gamveOverMsg;

        // Audio
        private SoundEffect _ballHitSFX;
        private SoundEffect _scoreSFX;

        // State stuff
        private ClientState _state = ClientState.NotConnected;
        private ThreadSafe<bool> _running = new ThreadSafe<bool>(false);
        private ThreadSafe<bool> _sendBye = new ThreadSafe<bool>(false);

        public PongClient(string hostname, int port)
        {
            // Content
            Content.RootDirectory = "Content";

            // Graphics setup
            _graphics = new GraphicsDeviceManager(this);
            _graphics.PreferredBackBufferWidth = GameGeometry.PlayArea.X;
            _graphics.PreferredBackBufferHeight = GameGeometry.PlayArea.Y;
            _graphics.IsFullScreen = false;
            _graphics.ApplyChanges();

            // Game Objects
            _ball = new Ball();
            _left = new Paddle(PaddleSide.Left);
            _right = new Paddle(PaddleSide.Right);

            // Connection stuff
            ServerHostname = hostname;
            ServerPort = port;
            _udpClient = new UdpClient(ServerHostname, ServerPort);
        }

        protected override void Initialize()
        {
            base.Initialize();
            _left.Initialize();
            _right.Initialize();
        }

        protected override void LoadContent()
        {
            _spriteBatch = new SpriteBatch(_graphics.GraphicsDevice);

            // Load the game objects
            _ball.LoadContent(Content);
            _left.LoadContent(Content);
            _right.LoadContent(Content);

            // Load the messages
             _establishingConnectionMsg = Content.Load<Texture2D>("establishing-connection-msg.png");
            _waitingForGameStartMsg = Content.Load<Texture2D>("waiting-for-game-start-msg.png");
            _gamveOverMsg = Content.Load<Texture2D>("game-over-msg.png");

            // Load sound effects
            _ballHitSFX = Content.Load<SoundEffect>("ball-hit.wav");
            _scoreSFX = Content.Load<SoundEffect>("score.wav");
        }

        protected override void UnloadContent()
        {
            // Cleanup
            _networkThread?.Join(TimeSpan.FromSeconds(2));
            _udpClient.Close();

            base.UnloadContent();
        }

        protected override void Update(GameTime gameTime)
        {
            // Check for close
            KeyboardState kbs = Keyboard.GetState();
            if (kbs.IsKeyDown(Keys.Escape))
            {
                // Player wants to quit, send a ByePacket (if we're connected)
                if ((_state == ClientState.EstablishingConnection) ||
                    (_state == ClientState.WaitingForGameStart) ||
                    (_state == ClientState.InGame))
                {
                    // Will trigger the network thread to send the Bye Packet
                    _sendBye.Value = true;
                }

                // Will stop the network thread
                _running.Value = false;
                _state = ClientState.GameOver;
                Exit();
            }

            // Check for time out with the server
            if (_timedOut())
                _state = ClientState.GameOver;

            // Get message
            NetworkMessage message;
            bool haveMsg = _incomingMessages.TryDequeue(out message);

            // Check for Bye From server
            if (haveMsg && (message.Packet.Type == PacketType.Bye))
            {
                // Shutdown the network thread (not needed anymore)
                _running.Value = false;
                _state = ClientState.GameOver;
            }

            switch (_state)
            {
                case ClientState.EstablishingConnection:
                    _sendRequestJoin(TimeSpan.FromSeconds(1));
                    if (haveMsg)
                        _handleConnectionSetupResponse(message.Packet);                        
                    break;

                case ClientState.WaitingForGameStart:
                    // Send a heartbeat
                    _sendHeartbeat(TimeSpan.FromSeconds(0.2));

                    if (haveMsg)
                    {
                        switch (message.Packet.Type)
                        {
                            case PacketType.AcceptJoin:
                                // It's possible that they didn't receive our ACK in the previous state
                                _sendAcceptJoinAck();
                                break;

                            case PacketType.HeartbeatAck:
                                // Record ACK times
                                _lastPacketReceivedTime = message.ReceiveTime;
                                if (message.Packet.Timestamp > _lastPacketReceivedTimestamp)
                                    _lastPacketReceivedTimestamp = message.Packet.Timestamp;
                                break;

                            case PacketType.GameStart:
                                // Start the game and ACK it
                                _sendGameStartAck();
                                _state = ClientState.InGame;
                                break;
                        }

                    }
                    break;

                case ClientState.InGame:
                    // Send a heartbeat
                    _sendHeartbeat(TimeSpan.FromSeconds(0.2));

                    // update our paddle
                    _previousY = _ourPaddle.Position.Y;
                    _ourPaddle.ClientSideUpdate(gameTime);
                    _sendPaddlePosition(_sendPaddlePositionTimeout);

                    if (haveMsg)
                    {
                        switch (message.Packet.Type)
                        {
                            case PacketType.GameStart:
                                // It's possible the server didn't receive our ACK in the previous state
                                _sendGameStartAck();
                                break;

                            case PacketType.HeartbeatAck:
                                // Record ACK times
                                _lastPacketReceivedTime = message.ReceiveTime;
                                if (message.Packet.Timestamp > _lastPacketReceivedTimestamp)
                                    _lastPacketReceivedTimestamp = message.Packet.Timestamp;
                                break;

                            case PacketType.GameState:
                                // Update the gamestate, make sure its the latest
                                if (message.Packet.Timestamp > _lastPacketReceivedTimestamp)
                                {
                                    _lastPacketReceivedTimestamp = message.Packet.Timestamp;

                                    GameStatePacket gsp = new GameStatePacket(message.Packet.GetBytes());
                                    _left.Score = gsp.LeftScore;
                                    _right.Score = gsp.RightScore;
                                    _ball.Position = gsp.BallPosition;

                                    // Update what's not our paddle
                                    if (_ourPaddle.Side == PaddleSide.Left)
                                        _right.Position.Y = gsp.RightY;
                                    else
                                        _left.Position.Y = gsp.LeftY;
                                }

                                break;

                            case PacketType.PlaySoundEffect:
                                #if CAN_PLAY_SOUNDS

                                // Play a sound
                                PlaySoundEffectPacket psep = new PlaySoundEffectPacket(message.Packet.GetBytes());
                                if (psep.SFXName == "ball-hit")
                                    _ballHitSFX.Play();
                                else if (psep.SFXName == "score")
                                    _scoreSFX.Play();
                                
                                #endif
                                break;
                        }
                    }

                    break;

                case ClientState.GameOver:
                    // Purgatory is here
                    break;
            }

            base.Update(gameTime);
        }

        public void Start()
        {
            _running.Value = true;
            _state = ClientState.EstablishingConnection;

            // Start the packet receiving/sending Thread
            _networkThread = new Thread(new ThreadStart(_networkRun));
            _networkThread.Start();
        }

        #region Graphical Functions
        protected override void Draw(GameTime gameTime)
        {
            _graphics.GraphicsDevice.Clear(Color.Black);

            _spriteBatch.Begin();

            // Draw different things based on the state
            switch (_state)
            {
                case ClientState.EstablishingConnection:
                    _drawCentered(_establishingConnectionMsg);
                    Window.Title = String.Format("Pong -- Connecting to {0}:{1}", ServerHostname, ServerPort);
                    break;
                
                case ClientState.WaitingForGameStart:
                    _drawCentered(_waitingForGameStartMsg);
                    Window.Title = String.Format("Pong -- Waiting for 2nd Player");
                    break;

                case ClientState.InGame:
                    // Draw game objects
                    _ball.Draw(gameTime, _spriteBatch);
                    _left.Draw(gameTime, _spriteBatch);
                    _right.Draw(gameTime, _spriteBatch);

                    // Change the window title
                    _updateWindowTitleWithScore();
                    break;

                case ClientState.GameOver:
                    _drawCentered(_gamveOverMsg);
                    _updateWindowTitleWithScore();
                    break;
            }

            _spriteBatch.End();

            base.Draw(gameTime);
        }

        private void _drawCentered(Texture2D texture)
        {
            Vector2 textureCenter = new Vector2(texture.Width / 2, texture.Height / 2);
            _spriteBatch.Draw(texture, GameGeometry.ScreenCenter, null, null, textureCenter);
        }

        private void _updateWindowTitleWithScore()
        {
            string fmt = (_ourPaddle.Side == PaddleSide.Left) ? 
                "[{0}] -- Pong -- {1}" : "{0} -- Pong -- [{1}]";
            Window.Title = String.Format(fmt, _left.Score, _right.Score);
        }
        #endregion // Graphical Functions

        #region Network Functions
        // This function is meant to be run in its own thread
        // and will populate the _incomingMessages queue
        private void _networkRun()
        {
            while (_running.Value)
            {
                bool canRead = _udpClient.Available > 0;
                int numToWrite = _outgoingMessages.Count;

                // Get data if there is some
                if (canRead)
                {
                    // Read in one datagram
                    IPEndPoint ep = new IPEndPoint(IPAddress.Any, 0);
                    byte[] data = _udpClient.Receive(ref ep);              // Blocks

                    // Enque a new message
                    NetworkMessage nm = new NetworkMessage();
                    nm.Sender = ep;
                    nm.Packet = new Packet(data);
                    nm.ReceiveTime = DateTime.Now;

                    _incomingMessages.Enqueue(nm);

                    //Console.WriteLine("RCVD: {0}", nm.Packet);
                }

                // Write out queued
                for (int i = 0; i < numToWrite; i++)
                {
                    // Send some data
                    Packet packet;
                    bool have = _outgoingMessages.TryDequeue(out packet);
                    if (have)
                        packet.Send(_udpClient);

                    //Console.WriteLine("SENT: {0}", packet);
                }

                // If Nothing happened, take a nap
                if (!canRead && (numToWrite == 0))
                    Thread.Sleep(1);
            }

            // Check to see if a bye was requested, one last operation
            if (_sendBye.Value)
            {
                ByePacket bp = new ByePacket();
                bp.Send(_udpClient);
                Thread.Sleep(1000);     // Needs some time to send through
            }
        }

        // Queues up to send a single Packet to the server
        private void _sendPacket(Packet packet)
        {
            _outgoingMessages.Enqueue(packet);
            _lastPacketSentTime = DateTime.Now;
        }

        // Sends a RequestJoinPacket,
        private void _sendRequestJoin(TimeSpan retryTimeout)
        {
            // Make sure not to spam them
            if (DateTime.Now >= (_lastPacketSentTime.Add(retryTimeout)))
            {
                RequestJoinPacket gsp = new RequestJoinPacket();
                _sendPacket(gsp);
            }
        }

        // Acks the AcceptJoinPacket
        private void _sendAcceptJoinAck()
        {
            AcceptJoinAckPacket ajap = new AcceptJoinAckPacket();
            _sendPacket(ajap);
        }

        // Responds to the Packets where we are establishing our connection with the server
        private void _handleConnectionSetupResponse(Packet packet)
        {
            // Check for accept and ACK
            if (packet.Type == PacketType.AcceptJoin)
            {
                // Make sure we haven't gotten it before
                if (_ourPaddle == null)
                {
                    // See which paddle we are
                    AcceptJoinPacket ajp = new AcceptJoinPacket(packet.GetBytes());
                    if (ajp.Side == PaddleSide.Left)
                        _ourPaddle = _left;
                    else if (ajp.Side == PaddleSide.Right)
                        _ourPaddle = _right;
                    else
                        throw new Exception("Error, invalid paddle side given by server.");     // Should never hit this, but just incase
                }

                // Send a response
                _sendAcceptJoinAck();

                // Move the state
                _state = ClientState.WaitingForGameStart;
            }
        }

        // Sends a HearbeatPacket to the server
        private void _sendHeartbeat(TimeSpan resendTimeout)
        {
            // Make sure not to spam them
            if (DateTime.Now >= (_lastPacketSentTime.Add(resendTimeout)))
            {
                HeartbeatPacket hp = new HeartbeatPacket();
                _sendPacket(hp);
            }
        }

        // Acks the GameStartPacket
        private void _sendGameStartAck()
        {
            GameStartAckPacket gsap = new GameStartAckPacket();
            _sendPacket(gsap);
        }

        // Sends the server our current paddle's Y Position (if it's changed)
        private void _sendPaddlePosition(TimeSpan resendTimeout)
        {
            // Don't send anything if there hasn't been an update
            if (_previousY == _ourPaddle.Position.Y)
                return;

            // Make sure not to spam them
            if (DateTime.Now >= (_lastPacketSentTime.Add(resendTimeout)))
            {
                PaddlePositionPacket ppp = new PaddlePositionPacket();
                ppp.Y = _ourPaddle.Position.Y;

                _sendPacket(ppp);
            }
        }

        // Returns true if out connection to the server has timed out or not
        // If we haven't recieved a packet at all from them, they're not timed out
        private bool _timedOut()
        {
            // We haven't recorded it yet
            if (_lastPacketReceivedTime == DateTime.MinValue)
                return false;    

            // Do math
            return (DateTime.Now > (_lastPacketReceivedTime.Add(_heartbeatTimeout)));
        }
        #endregion // Network Functions





        #region Program Execution
        public static void Main(string[] args)
        {
            // Get arguements
            string hostname = "localhost";//args[0].Trim();
            int port = 6000;//int.Parse(args[1].Trim());

            // Start the client
            PongClient client = new PongClient(hostname, port);
            client.Start();
            client.Run();
        }
        #endregion  // Program Execution
    }
}
```

