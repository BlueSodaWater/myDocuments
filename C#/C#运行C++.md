## 1.Windows

1.创建Dynamic-Link Libiary(Dll) 项目

2.新增代码

```c++
#include "pch.h"
#include "iostream"
using namespace std;

extern "C" 
{
    _declspec(dllexport) int Sum(int a, int b)
    {
        if (a - (int)a != 0 || b - (int)b != 0) {
            cout << "请输入整数" << endl;
            return -1;
        }
        return a + b;
    }
    _declspec(dllexport) void Sb()
    {
        cout << "1" << endl;
    }
}
```

**3.运行环境改为x64，然后build，找到生成的dll**

4.新建C# Console application

5.写入代码

```c#
using System;
using System.Runtime.InteropServices;

namespace ConsoleApp3
{
    class Program
    {
        [DllImport("Test.dll", CallingConvention = CallingConvention.Cdecl)]
        extern static int Sum(int a, int b);

        static void Main(string[] args)
        {
            try
            {
                Console.WriteLine("请输入NumberA:");
                int numberA = Convert.ToInt32(Console.ReadLine());

                Console.WriteLine("请输入NumberB:");
                int numberB = Convert.ToInt32(Console.ReadLine());

                Console.WriteLine($"the numberA is:{numberA};numberB is:{numberB},The Sum is:{Sum(numberA, numberB)}");

            }
            catch (Exception ex)
            {
                Console.WriteLine($"ex:{ex}");
            }

            Console.ReadLine();
        }
    }
}

```

6.运行环境改成x64

6.将dll放入bin/x64/debug/net5.0文件夹下，运行成功

参考网址

1. https://www.cnblogs.com/skyfreedom/p/11773597.html
2. https://docs.microsoft.com/zh-cn/cpp/build/walkthrough-creating-and-using-a-dynamic-link-library-cpp?view=msvc-160
3. https://stackoverflow.com/questions/16332701/how-to-call-c-dll-in-c-sharp

## 2.Linux

新建linux C++代码

```c++
#include<iostream>
using namespace std;
extern "C" {
	int JustTest(int a, int b);
}
int JustTest(int a, int b) {
	return a + b;
}
```

**注意linux上不需要_declspec(dllexport)**

新增C#代码

```c#
using System;
using System.Runtime.InteropServices;

namespace DotnetSelector
{
    class Program
    {
        [DllImport("test.so", EntryPoint = "JustTest")]
        static extern int Test(int a, int b);

        static void Main(string[] args)
        {
            var c= Test(1,2);
            Console.WriteLine(c);
        }
    }
}
```

步骤基本和在windows相同