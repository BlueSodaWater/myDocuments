## DNS

[Dns](https://msdn.microsoft.com/en-us/library/system.net.dns(v=vs.110).aspx)类是在System.Net包中经典的一个类，是一个域名服务的工具。如果你想让你的用户输入`16bpp.net`域名而不是`107.170.178.215`，这很方便。对于那些不知道DNS是啥的人，把字母变成数字很神奇。

Dns类中包含一系列方法来解析域名和其它的事情，你可以同步或者异步的使用他们。这个类中最重要的方法要么返回 [IPAddress](https://msdn.microsoft.com/en-us/library/system.net.ipaddress(v=vs.110).aspx)或者更详细的[IPHostEntry](https://msdn.microsoft.com/en-us/library/system.net.iphostentry(v=vs.110).aspx)。如果你希望获取与主机名相关的别名，后者更好。请注意，一个域名可能绑定多个ip地址。

```c#
using System;
using System.Net;

namespace DnsExample
{
    class DnsExample
    {
        public static string domain = "16bpp.net";

        public static void Main(string[] args)
        {
            // Print a little info about us
            Console.WriteLine("Local Hostname: {0}", Dns.GetHostName());
            Console.WriteLine();

            // Get DNS info synchronously
            IPHostEntry hostInfo = Dns.GetHostEntry(domain);

            // Print aliases
            if (hostInfo.Aliases.Length > 0)
            {
                Console.WriteLine("Aliases for {0}:", hostInfo.HostName);
                foreach (string alias in hostInfo.Aliases)
                    Console.WriteLine("  {0}", alias);
                Console.WriteLine();
            }

            // Print IP addresses
            if (hostInfo.AddressList.Length > 0)
            {
                Console.WriteLine("IP Addresses for {0}", hostInfo.HostName);
                foreach(IPAddress addr in hostInfo.AddressList)
                    Console.WriteLine("  {0}", addr);
                Console.WriteLine();
            }
        }
    }
}
```

## IPAddress

[IPAddress](https://msdn.microsoft.com/en-us/library/system.net.ipaddress(v=vs.110).aspx)`System.Net`命名空间下的一个小类，它的作用就是展示IP地址，支持IPv4和IPv6类型的地址。

你可以通过向构造器提供一个字节数组来创建一个新的IP地址，或者使用`IPAddress.Parse()`（或者`IPAdress.TryParse()`）来从字符串中得到一个。如果IP地址格式不对会抛出异常。让使用构造器时，如果你提供四个字节的字节数组，会假定你用的是IPv4地址。

```c#
using System;
using System.Net;

namespace IPAddressExample
{
    class IPAddressExample
    {
        public static readonly byte[] ipv6 = {
            0x20, 0x01,
            0x0d, 0xb8,
            0x00, 0x00,
            0x00, 0x42,
            0x00, 0x00,
            0x8a, 0x2e,
            0x03, 0x70,
            0x73, 0x34
        };

        public static void Main(string[] args)
        {
            // Make an IP address
            IPAddress ipAddr;
            ipAddr = new IPAddress(new byte[] {107, 70, 178, 215});  // IPv4, byte array
            //ipAddr = new IPAddress(ipv6);                          // IPv6, byte array
            //ipAddr = IPAddress.Parse("127.0.0.1");                 // IPv4, string
            //ipAddr = IPAddress.Parse("::1");                       // IPv6, string

            // Print some info
            Console.WriteLine("IPAddress: {0}", ipAddr);
            Console.WriteLine("Address Family: {0}", ipAddr.AddressFamily);
            Console.WriteLine("Loopback: {0}", IPAddress.IsLoopback(ipAddr));
        }
    }
}
```

请注意`IPAddress`中的静态字段很重要，当创建服务的时候，他们都是只读的

- `Any` - 告诉`Socket`可以监听任何网络设备的连接，可以是本地的也可以是网络的
- `Loopback` - 告诉`Socket`只监听来自本机的连接（127.0.0.1）
- `Broadcast` - 为套接字提供广播IP地址
- `None` - 告诉`Socket`不要监听任何连接

这些字段是都IPv4下的，IPv6中有更多的字段。

### IPEndPoint

[IPEndPoint](https://msdn.microsoft.com/en-us/library/system.net.ipendpoint(v=vs.110).aspx)使一个数据结构简单的包含[IPAddress](https://msdn.microsoft.com/en-us/library/system.net.ipaddress(v=vs.110).aspx)和一个端口号。它实际上代表在互联网上可以连接的资源。两个静态字段`MinPort`和`MaxPort`代表端口可能的范围（16位无符号数）

```c#
using System;
using System.Net;

namespace IPEndPointExample
{
    class IPEndPointExample
    {
        public static void Main(string[] args)
        {
            // Print some static info
            Console.WriteLine("Min. Port: {0}", IPEndPoint.MinPort);
            Console.WriteLine("Max. Port: {0}", IPEndPoint.MaxPort);
            Console.WriteLine();

            // Create one
            IPEndPoint endPoint = new IPEndPoint(IPAddress.Parse("107.70.178.215"), 6000);
            Console.WriteLine("Address: {0}", endPoint.Address);
            Console.WriteLine("Port: {0}", endPoint.Port);
            Console.WriteLine("Address Family: {0}", endPoint.AddressFamily);
        }
    }
}