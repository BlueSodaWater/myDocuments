**索引和范围的语言支持
**

此语言支持依赖于两个新类型和两个新运算符：

- System.Index 表示一个序列索引。
- 来自末尾运算符 ^ 的索引，指定一个索引与序列末尾相关。
- System.Range 表示序列的子范围。
- 范围运算符 ..，用于指定范围的开始和末尾，就像操作数一样。

让我们从索引规则开始。 请考虑数组 `sequence`。 `0` 索引与 `sequence[0] `相同。 `^0` 索引与 `sequence[sequence.Length]` 相同。 表达式 `sequence[^0] `不会引发异常，就像 `sequence[sequence.Length]` 一样。 对于任何数字 `n`，索引` ^n` 与 `sequence[sequence.Length - n] `相同。

```C#
string[] words = new string[]
{
    // index from start index from end
 "The",  // 0     ^9
 "quick", // 1     ^8
 "brown", // 2     ^7
 "fox",  // 3     ^6
 "jumped", // 4     ^5
 "over",  // 5     ^4
 "the",  // 6     ^3
 "lazy",  // 7     ^2
 "dog"  // 8     ^1
};    // 9 (or words.Length) ^0
```

可以使用 `^1` 索引检索最后一个词。 在初始化下面添加以下代码：

```C#
Console.WriteLine($"The last word is {words[^1]}");
```

范围指定范围的开始和末尾。 范围是排除的，也就是说“末尾”不包含在范围内。 范围 [0..^0] 表示整个范围，就像 `[0..sequence.Length] `表示整个范围。
以下代码创建了一个包含单词“quick”、“brown”和“fox”的子范围。 它包括 words[1] 到 words[3]。 元素 words[4] 不在该范围内。 将以下代码添加到同一方法中。 将其复制并粘贴到交互式窗口的底部。

```c#
string[] quickBrownFox = words[1..4];
foreach (var word in quickBrownFox)
 Console.Write($"< {word} >");
Console.WriteLine();
```

以下代码使用“lazy”和“dog”返回范围。 它包括 `words[^2]` 和 `words[^1]`。 结束索引 `words[^0]` 不包括在内。 同样添加以下代码：

```C#
string[] lazyDog = words[^2..^0];
foreach (var word in lazyDog)
 Console.Write($"< {word} >");
Console.WriteLine();
```

下面的示例为开始和/或结束创建了开放范围：

```c#
string[] allWords = words[..]; // contains "The" through "dog".
string[] firstPhrase = words[..4]; // contains "The" through "fox"
string[] lastPhrase = words[6..]; // contains "the, "lazy" and "dog"
foreach (var word in allWords)
 Console.Write($"< {word} >");
Console.WriteLine();
foreach (var word in firstPhrase)
 Console.Write($"< {word} >");
Console.WriteLine();
foreach (var word in lastPhrase)
 Console.Write($"< {word} >");
Console.WriteLine();
```

还可以将范围或索引声明为变量。 然后可以在 [ 和 ] 字符中使用该变量：

```c#
Index the = ^3;
Console.WriteLine(words[the]);
Range phrase = 1..4;
string[] text = words[phrase];
foreach (var word in text)
 Console.Write($"< {word} >");
Console.WriteLine();
```

下面的示例展示了使用这些选项的多种原因。 请修改 x、y 和 z 以尝试不同的组合。 在进行实验时，请使用 x 小于 y且 y 小于 z 的有效组合值。 在新方法中添加以下代码。 尝试不同的组合：

```C#
int[] numbers = Enumerable.Range(0, 100).ToArray();
int x = 12;
int y = 25;
int z = 36;
 
Console.WriteLine($"{numbers[^x]} is the same as {numbers[numbers.Length - x]}");
Console.WriteLine($"{numbers[x..y].Length} is the same as {y - x}");
 
Console.WriteLine("numbers[x..y] and numbers[y..z] are consecutive and disjoint:");
Span<int> x_y = numbers[x..y];
Span<int> y_z = numbers[y..z];
Console.WriteLine($"\tnumbers[x..y] is {x_y[0]} through {x_y[^1]}, numbers[y..z] is {y_z[0]} through {y_z[^1]}");
 
Console.WriteLine("numbers[x..^x] removes x elements at each end:");
Span<int> x_x = numbers[x..^x];
Console.WriteLine($"\tnumbers[x..^x] starts with {x_x[0]} and ends with {x_x[^1]}");
 
Console.WriteLine("numbers[..x] means numbers[0..x] and numbers[x..] means numbers[x..^0]");
Span<int> start_x = numbers[..x];
Span<int> zero_x = numbers[0..x];
Console.WriteLine($"\t{start_x[0]}..{start_x[^1]} is the same as {zero_x[0]}..{zero_x[^1]}");
Span<int> z_end = numbers[z..];
Span<int> z_zero = numbers[z..^0];
Console.WriteLine($"\t{z_end[0]}..{z_end[^1]} is the same as {z_zero[0]}..{z_zero[^1]}");
```

**索引和范围的类型支持**

索引和范围提供清晰、简洁的语法来访问序列中的单个元素或元素的范围。 索引表达式通常返回序列元素的类型。 范围表达式通常返回与源序列相同的序列类型。
若任何类型提供带 Index 或 Range 参数的索引器，则该类型可分别显式支持索引或范围。 采用单个 Range 参数的索引器可能会返回不同的序列类型，如 System.Span<T>。

> **重要**
>
> 使用范围运算符的代码的性能取决于序列操作数的类型。
> 范围运算符的时间复杂度取决于序列类型。 例如，如果序列是 string 或数组，则结果是输入中指定部分的副本，因此，时间复杂度为 O(N)（其中 N 是范围的长度）。 另一方面，如果它是 `System.Span<T> `或 `System.Memory<T>`，则结果引用相同的后备存储，这意味着没有副本且操作为 O(1)。
> 除了时间复杂度外，这还会产生额外的分配和副本，从而影响性能。 在性能敏感的代码中，考虑使用 `Span<T>` 或 `Memory<T>` 作为序列类型，因为不会为其分配范围运算符。



若类型包含名称为 `Length` 或` Count `的属性，属性有可访问的 Getter 并且其返回类型为 `int`，则此类型为可计数类型。**** 不显式支持索引或范围的可计数类型可能为它们提供隐式支持。 有关详细信息，请参阅功能建议说明的隐式索引支持和隐式范围支持部分。 使用隐式范围支持的范围将返回与源序列相同的序列类型。
例如，以下 .NET 类型同时支持索引和范围：String、Span<T> 和 ReadOnlySpan<T>。 List<T> 支持索引，但不支持范围。
Array 具有更多的微妙行为。 单个维度数组同时支持索引和范围。 多维数组则不支持。 多维数组的索引器具有多个参数，而不是一个参数。 交错数组（也称为数组的数组）同时支持范围和索引器。 下面的示例演示如何循环访问交错数组的矩形子节。 它循环访问位于中心的节，不包括前三行和后三行，以及每个选定行中的前两列和后两列：

```C#
var jagged = new int[10][]
{
 new int[10] { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9},
 new int[10] { 10,11,12,13,14,15,16,17,18,19},
 new int[10] { 20,21,22,23,24,25,26,27,28,29},
 new int[10] { 30,31,32,33,34,35,36,37,38,39},
 new int[10] { 40,41,42,43,44,45,46,47,48,49},
 new int[10] { 50,51,52,53,54,55,56,57,58,59},
 new int[10] { 60,61,62,63,64,65,66,67,68,69},
 new int[10] { 70,71,72,73,74,75,76,77,78,79},
 new int[10] { 80,81,82,83,84,85,86,87,88,89},
 new int[10] { 90,91,92,93,94,95,96,97,98,99},
};
 
var selectedRows = jagged[3..^3];
 
foreach (var row in selectedRows)
{
 var selectedColumns = row[2..^2];
 foreach (var cell in selectedColumns)
 {
  Console.Write($"{cell}, ");
 }
 Console.WriteLine();
}
```

在所有情况下，Array 的范围运算符都会分配一个数组来存储返回的元素。

**索引和范围的应用场景**

要分析较大序列的一部分时，通常会使用范围和索引。 在准确读取所涉及的序列部分这一方面，新语法更清晰。 本地函数 `MovingAverage` 以 `Range `为参数。 然后，该方法在计算最小值、最大值和平均值时仅枚举该范围。 在项目中尝试以下代码：

```C#
int[] sequence = Sequence(1000);
 
for(int start = 0; start < sequence.Length; start += 100)
{
 Range r = start..(start+10);
 var (min, max, average) = MovingAverage(sequence, r);
 Console.WriteLine($"From {r.Start} to {r.End}: \tMin: {min},\tMax: {max},\tAverage: {average}");
}
 
for (int start = 0; start < sequence.Length; start += 100)
{
 Range r = ^(start + 10)..^start;
 var (min, max, average) = MovingAverage(sequence, r);
 Console.WriteLine($"From {r.Start} to {r.End}: \tMin: {min},\tMax: {max},\tAverage: {average}");
}
 
(int min, int max, double average) MovingAverage(int[] subSequence, Range range) =>
 (
  subSequence[range].Min(),
  subSequence[range].Max(),
  subSequence[range].Average()
 );
 
int[] Sequence(int count) =>
 Enumerable.Range(0, count).Select(x => (int)(Math.Sqrt(x) * 100)).ToArray();
```