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

```c#
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

