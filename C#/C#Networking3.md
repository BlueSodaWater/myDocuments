## Compression

在我们开始UDP之前，我想先聊一聊压缩。通过我们称之为互联网的管道数据传输可能需要一段时间。这取决于很多因素像是连接速度，A和B节点的位置，硬件等等。但是你想发送的数据大小是你可以控制的，通过压缩，我们可以把想要发送的数据变小。

对我们有利的是，在.NET Framework包含了压缩相关的内容在[System.IO.Compression](https://msdn.microsoft.com/en-us/library/system.io.compression(v=vs.110).aspx)命名空间下。我们尤其需要注意[GZipStream](https://msdn.microsoft.com/en-us/library/system.io.compression.gzipstream(v=vs.110).aspx)和[DeflateStream](https://msdn.microsoft.com/en-us/library/system.io.compression.deflatestream(v=vs.110).aspx)。这两个流使用[DEFLATE algorithm](https://en.wikipedia.org/wiki/DEFLATE)来无损压缩文件，这是通过流行的[zlib library](http://www.zlib.net/)实现的。

### DefalteStream vs.GZipStream

看着两个流，你可能困惑到底使用哪个。他们都是使用DEFLATE算法，但是有啥区别？`DeflateStream`仅仅压缩你的数据，`GZipStream`也可以做到这一点，但是同时`GZipStream`还可以添加一些额外的数据（比如CRC），这样你就可以直接将数据保存为.gz文件。

请注意，你的程序中并不局限于使用DEFLATE压缩。你也可以基础一些其他算法或者引用外部包。`DeflateStream`和`GZipStream`使用起来很简单。

下面的源代码展示如果在文件的数据中使用这两个流。它也提供了一些压缩数据的信息。

```c#
using System;
using System.Diagnostics;
using System.IO;
using System.IO.Compression;

namespace Compression
{
    class CompressionExample
    {
        // Tiny helper to get a number in MegaBytes
        public static float ComputeSizeInMB(long size)
        {
            return (float)size / 1024f / 1024f;
        }

        public static void Main(string[] args)
        {
            // Our testing file
            string fileToCompress = "image.bmp";
            byte[] uncompressedBytes = File.ReadAllBytes(fileToCompress);

            // Benchmarking
            Stopwatch timer = new Stopwatch();

            // Display some information
            long uncompressedFileSize = uncompressedBytes.LongLength;
            Console.WriteLine("{0} uncompressed is {1:0.0000} MB large.",
                fileToCompress,
                ComputeSizeInMB(uncompressedFileSize));

            // Compress it using Deflate (optimal)
            using (MemoryStream compressedStream = new MemoryStream())
            {
                // init
                DeflateStream deflateStream = new DeflateStream(compressedStream, CompressionLevel.Optimal, true);

                // Run the compression
                timer.Start();
                deflateStream.Write(uncompressedBytes, 0, uncompressedBytes.Length);
                deflateStream.Close();
                timer.Stop();

                // Print some info
                long compressedFileSize = compressedStream.Length;
                Console.WriteLine("Compressed using DeflateStream (Optimal): {0:0.0000} MB [{1:0.00}%] in {2}ms",
                    ComputeSizeInMB(compressedFileSize),
                    100f * (float)compressedFileSize / (float)uncompressedFileSize,
                    timer.ElapsedMilliseconds);

                // cleanup
                timer.Reset();
            }

            // Compress it using Deflate (fast)
            using (MemoryStream compressedStream = new MemoryStream())
            {
                // init
                DeflateStream deflateStream = new DeflateStream(compressedStream, CompressionLevel.Fastest, true);

                // Run the compression
                timer.Start();
                deflateStream.Write(uncompressedBytes, 0, uncompressedBytes.Length);
                deflateStream.Close();
                timer.Stop();

                // Print some info
                long compressedFileSize = compressedStream.Length;
                Console.WriteLine("Compressed using DeflateStream (Fast): {0:0.0000} MB [{1:0.00}%] in {2}ms",
                    ComputeSizeInMB(compressedFileSize),
                    100f * (float)compressedFileSize / (float)uncompressedFileSize,
                    timer.ElapsedMilliseconds);

                // cleanup
                timer.Reset();
            }

            // Compress it using GZip (save it)
            string savedArchive = fileToCompress + ".gz";
            using (MemoryStream compressedStream = new MemoryStream())
            {
                // init
                GZipStream gzipStream = new GZipStream(compressedStream, CompressionMode.Compress, true);

                // Run the compression
                timer.Start();
                gzipStream.Write(uncompressedBytes, 0, uncompressedBytes.Length);
                gzipStream.Close();
                timer.Stop();

                // Print some info
                long compressedFileSize = compressedStream.Length;
                Console.WriteLine("Compressed using GZipStream: {0:0.0000} MB [{1:0.00}%] in {2}ms",
                    ComputeSizeInMB(compressedFileSize),
                    100f * (float)compressedFileSize / (float)uncompressedFileSize,
                    timer.ElapsedMilliseconds);

                // Save it
                using (FileStream saveStream = new FileStream(savedArchive, FileMode.Create))
                {
                    compressedStream.Position = 0;
                    compressedStream.CopyTo(saveStream);
                }

                // cleanup
                timer.Reset();
            }
        }
    }
}
```

下面是我的输出，我同时使用.bmp的图片和一个小的.mp4来测试它。.bmp运行非常良好，因为他是未压缩的。.mp4之前已经被压缩过了，因此我们基本不能压缩更多。

```tex
image.bmp uncompressed is 3.1239 MB large.
Compressed using DeflateStream (Optimal): 0.3496 MB [11.19%] in 32ms
Compressed using DeflateStream (Optimal): 0.3926 MB [12.57%] in 9ms
Compressed using GZipStream: 0.3496 MB [11.19%] in 25ms

film.mp4 uncompressed is 35.7181 MB large.
Compressed using DeflateStream (Optimal): 35.3457 MB [98.96%] in 1155ms
Compressed using DeflateStream (Optimal): 35.3585 MB [98.99%] in 849ms
Compressed using GZipStream: 35.3457 MB [98.96%] in 1004ms
```

这三个程序看起来像是创建了完全相同的压缩并执行了类似的操作，但记住我们截断了打印输出。很有可能实际`Fastest`和`Optimal`是不同的（他们的大小也是）

### So what‘s going on？

代码很简单没啥需要解释的。在顶端我们定义了一个小小的帮助类来计算文件的大小（MB）。在`Main()`看开头，我们从测试文件中读取所有的数据，首先我们使用`Deflate/Optimal`然后使用`Deflate/Fast`。最后我们尝试将其保存为.gz文件（还有一些标记代码）

一开始处理这些流可能有点棘手，但步骤如下：

1. 创建希望数据写入的一个流（比如`MemoryStream`）
2. 创建一个`DeflateStream`/`GZipStream`，将刚刚的流设置为目标流，指定模式/级别，我们可以选择在压缩流关闭之后保持目标流仍然打开。
3. 将数据写入压缩流
4. 然后使用`Close()`或者`Flush()`关闭压缩流
   - 乍一看，这可能很奇怪。在压缩流被刷新之前，他不会向目标流写入任何数据。在读取目标之前一定要这样做（否则会一无所获）。
5. 我们压缩/未压缩的数据已经准备好，我们可以对其做任何操作。

我没有包含解压缩数据的例子，你只需要简单的将[CompressionMode.Decompress](https://msdn.microsoft.com/en-us/library/system.io.compression.compressionmode(v=vs.110).aspx)作为压缩流构造函数的第二个参数就好。

