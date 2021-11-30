### 设置连接远程计算机调试

debug->options

![image-20211130162228106](..\pics\image-20211130162153098.png)



![image-20211130162431785](..\pics\image-20211130162228106.png)

下载程序集

应该就ok了

### attach to process

现在linux上运行程序

```shell
dotnet build
dotnet run
```

VS上

![image-20211130162735116](..\pics\image-20211130162735116.png)

**注意：选择和程序名相同的进程！！!**

断点不起作用可能是因为上面的原因