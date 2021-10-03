---
title: "算法的时间复杂度和空间复杂度"
tags: [algorithm, complexity, "复杂度", todo]
---

# 为什么要进行算法分析？

- 预测算法所需的资源
   - 计算时间（CPU 消耗）
   - 内存空间（RAM 消耗）
   - 通信时间（带宽消耗）
- 预测算法的运行时间
   - 在给定输入规模时，所执行的基本操作数量。
   - 或者称为算法复杂度（Algorithm Complexity）

# 如何衡量算法复杂度？

- 内存（Memory）
- 时间（Time）
- 指令的数量（Number of Steps）
- 特定操作的数量
   - 磁盘访问数量
   - 网络包数量
- 渐进复杂度（Asymptotic Complexity）

# 算法的运行时间与什么相关？

- 取决于输入的数据。（例如：如果数据已经是排好序的，时间消耗可能会减少。）
- 取决于输入数据的规模。（例如：`6` 和 `6 * 109`）
- 取决于运行时间的上限。（因为运行时间的上限是对使用者的承诺。）

# 算法分析的种类

- 最坏情况（Worst Case）：任意输入规模的最大运行时间。（Usually）
- 平均情况（Average Case）：任意输入规模的期待运行时间。（Sometimes）
- 最佳情况（Best Case）：通常最佳情况不会出现。（Bogus）

例如，在一个长度为 n 的列表中顺序搜索指定的值，则

- 最坏情况：n 次比较
- 平均情况：n/2 次比较
- 最佳情况：1 次比较

而实际中，我们一般仅考量算法在最坏情况下的运行情况，也就是对于规模为 n 的任何输入，算法的最长运行时间。这样做的理由是：

- 一个算法的最坏情况运行时间是在任何输入下运行时间的一个上界（Upper Bound）。
- 对于某些算法，最坏情况出现的较为频繁。
- 大体上看，平均情况通常与最坏情况一样差。

算法分析要保持大局观（Big Idea），其基本思路：

- 忽略掉那些依赖于机器的常量。
- 关注运行时间的增长趋势。

比如：\\( T(n) = 73n^3 + 29n^3 + 8888 \\) 的趋势就相当于 \\( T(n) = Θ(n^3) \\)。

渐近记号（Asymptotic Notation）通常有 `O`、 `Θ` 和 `Ω` 记号法。`Θ` 记号渐进地给出了一个函数的上界和下界，当只有渐近上界时使用 `O` 记号，当只有渐近下界时使用 `Ω` 记号。尽管技术上 `Θ` 记号较为准确，但通常仍然使用 `O` 记号表示。

使用 `O` 记号法（`Big O Notation`）表示最坏运行情况的上界。例如，

- 线性复杂度 O(n) 表示每个元素都要被处理一次。
- 平方复杂度 \\( O(n^2) \\) 表示每个元素都要被处理 n 次。

|Notation|	Intuition|	Informal Definition|
|:---|:---|:--|
|f(n)∈O(g(n))|f is bounded above by g asymptotically||
|f(n)∈Ω(g(n))|Two definitions :<br>Number theory: f is not dominated by g asymptotically <br>Complexity theory: f is bounded below by g asymptotically| f(n) ≥ g(n)*k|
|f(n) ∈  Θ(g(n))| f is bounded both above and below by g asymptotically| g(n)*k1 ≤ f(n) ≤ g(n)*k2|

例如：
- \\( T(n) = O(n^3) \\) 等同于 \\( T(n) ∈ O(n^3) \\)
- \\( T(n) = Θ(n^3) \\) 等同于 \\( T(n) ∈ Θ(n^3) \\).

相当于:
- T(n) 的渐近增长不快于 \\( n^3 \\)。
- T(n) 的渐近增长与 \\( n^3 \\) 一样快。

|复杂度|	标记符号|	描述|
|:--|:--|:--|
|常量（Constant）	|O(1)| 操作的数量为常数，与输入的数据的规模无关。<br>n = 1,000,000 -> 1-2 operations |
|对数（Logarithmic）	| \\(  O(\mathbf{log}\_{2} n) \\)| 操作的数量与输入数据的规模 n 的比例是 \\( \mathbf{log}\_{2} (n) \\)。<br>n = 1,000,000 -> 30 operations|
|线性（Linear）	| O(n)|	操作的数量与输入数据的规模 n 成正比。<br>n = 10,000 -> 5000 operations|
|平方（Quadratic）|	\\( O(n^2) \\) |操作的数量与输入数据的规模 n 的比例为二次平方。<br>n = 500 -> 250,000 operations|
|立方（Cubic）	| \\( O(n^3) \\)	|操作的数量与输入数据的规模 n 的比例为三次方。<br>n = 200 -> 8,000,000 operations|
|指数（Exponential）	| \\( O(2^n) \\) <br> \\( O(k^n) \\) <br> \\( O(n!) \\)|指数级的操作，快速的增长。<br>n = 20 -> 1048576 operations|

**注1：**快速的数学回忆，\\( \mathbf{log}\_{a}b = y \\) 其实就是 \\( a^y = b \\)。所以，\\( \mathbf{log}\_{2}4 = 2 \\)，因为 \\( 2^2 = 4 \\)。同样 \\( \mathbf{log}\_{2}8 = 3 \\)，因为 \\( 2^3 = 8 \\)。我们说，\\( \mathbf{log}\_{2}n \\) 的增长速度要慢于 `n`，因为当 `n = 8 `时，\\( \mathbf{log}\_{2}n = 3 \\)。

**注2：** 通常将以 10 为底的对数叫做常用对数。为了简便，N 的常用对数 \\( \mathbf{log}\_{10}N \\) 简写做 `lg N`，例如 \\( \mathbf{log}\_{10}5 \\) 记做 `lg 5`。

**注3：** 通常将以无理数 `e` 为底的对数叫做自然对数。为了方便，N 的自然对数 \\( \mathbf{log}\_{e}N \\) 简写做 `ln N`，例如 \\( \mathbf{log}\_{e}3 \\) 记做 `ln 3`。

**注4：** 在算法导论中，采用记号 \\( lg n = \mathbf{log}\_{2}n \\) ，也就是以 2 为底的对数。改变一个对数的底只是把对数的值改变了一个常数倍，所以当不在意这些常数因子时，我们将经常采用 "`lg n`"记号，就像使用 `O` 记号一样。计算机工作者常常认为对数的底取 2 最自然，因为很多算法和数据结构都涉及到对问题进行二分。

![big-O complexity](https://images0.cnblogs.com/blog/175043/201412/151255023901167.png)

而通常时间复杂度与运行时间有一些常见的比例关系：


|复杂度	|10	|20|50|	100	|1000|	10000|	100000|
|:--|:--|:--|:--|:--|:--|:--|:--|
|O(1)	|<1s|<1s|<1s|<1s|<1s|<1s|<1s|
|\\( O(\mathbf{log}\_{2}(n))	\\) |1s|<1s|<1s|<1s|<1s|<1s|<1s|
|O(n)	|<1s|<1s|<1s|<1s|<1s|<1s|<1s|
|\\( O(n*\mathbf{log}\_{2}(n))	\\)|<1s|<1s|<1s|<1s|<1s|<1s|<1s|
|\\( O(n2)	\\)|<1s|<1s|<1s|<1s|<1s|2s|3-4 min|
|\\( O(n3)	\\)|<1s|<1s|<1s|<1s|20s| 5 hours | 231 days| 
|\\( O(2^n)	\\)|<1s|<1s| 260 days | hangs | hangs |hangs|hangs|
|O(n!)	|<1s| hangs |hangs| hangs |hangs|hangs|hangs|
|\\( O(n^n)	\\)| 3-4 min |hangs|hangs| hangs |hangs|hangs|hangs|


计算代码块的渐进运行时间的方法有如下步骤：

1. 确定决定算法运行时间的组成步骤。
2. 找到执行该步骤的代码，标记为 1。
3. 查看标记为 1 的代码的下一行代码。如果下一行代码是一个循环，则将标记 1 修改为 1 倍于循环的次数 `1 * n`。如果包含多个嵌套的循环，则将继续计算倍数，例如 `1 * n * m`。
4. 找到标记到的最大的值，就是运行时间的最大值，即算法复杂度描述的上界。

# 示例
示例代码（1）：
```js
     decimal Factorial(int n)
     {
       if (n == 0)
         return 1;
       else
         return n * Factorial(n - 1);
     }
```

阶乘（factorial），给定规模 n，算法基本步骤执行的数量为 `n`，所以算法复杂度为 `O(n)`。

示例代码（2）：

```js
      int FindMaxElement(int[] array)
      {
        int max = array[0];
        for (int i = 0; i < array.Length; i++)
        {
          if (array[i] > max)
          {
            max = array[i];
          }
       }
       return max;
     }
```
这里，n 为数组 array 的大小，则最坏情况下需要比较 n 次以得到最大值，所以算法复杂度为 `O(n)`。

示例代码（3）：
```js
     long FindInversions(int[] array)
     {
       long inversions = 0;
       for (int i = 0; i < array.Length; i++)
         for (int j = i + 1; j < array.Length; j++)
           if (array[i] > array[j])
             inversions++;
       return inversions;
     }
```
这里，n 为数组 array 的大小，则基本步骤的执行数量约为 `n*(n-1)/2`，所以算法复杂度为 \\( O(n^2) \\)。

示例代码（4）：

```js
1     long SumMN(int n, int m)
2     {
3       long sum = 0;
4       for (int x = 0; x < n; x++)
5         for (int y = 0; y < m; y++)
6           sum += x * y;
7       return sum;
8     }
```
给定规模 n 和 m，则基本步骤的执行数量为 `n*m`，所以算法复杂度为 \\( O(n^2) \\)。

示例代码（5）：

```js
     decimal Sum3(int n)
     {
       decimal sum = 0;
       for (int a = 0; a < n; a++)
         for (int b = 0; b < n; b++)
           for (int c = 0; c < n; c++)
             sum += a * b * c;
       return sum;
     }
```
这里，给定规模 n，则基本步骤的执行数量约为 `n*n*n` ，所以算法复杂度为 \\( O(n^3) \\)。

示例代码（6）：

```js
     decimal Calculation(int n)
     {
       decimal result = 0;
       for (int i = 0; i < (1 << n); i++)
         result += i;
       return result;
     }
```
这里，给定规模 n，则基本步骤的执行数量为 \\( 2^n \\)，所以算法复杂度为 \\( O(2^n) \\)。

示例代码（7）：

斐波那契数列：
```js
Fib(0) = 0
Fib(1) = 1
Fib(n) = Fib(n-1) + Fib(n-2)
F() = 0, 1, 1, 2, 3, 5, 8, 13, 21, 34 ...
```

```js
     int Fibonacci(int n)
     {
       if (n <= 1)
         return n;
       else
         return Fibonacci(n - 1) + Fibonacci(n - 2);
     }
```
这里，给定规模 n，计算 `Fib(n)` 所需的时间为计算 `Fib(n-1)` 的时间和计算 `Fib(n-2)` 的时间的和。

`T(n<=1) = O(1)`

`T(n) = T(n-1) + T(n-2) + O(1)`

                     fib(5)   
                 /             \     
           fib(4)                fib(3)   
         /      \                /     \
     fib(3)      fib(2)         fib(2)    fib(1)
    /     \        /    \       /    \  

通过使用递归树的结构描述可知算法复杂度为 \\( O(2^n) \\)。

示例代码（8）：

```js
      int Fibonacci(int n)
      {
        if (n <= 1)
          return n;
        else
        {
          int[] f = new int[n + 1];
          f[0] = 0;
          f[1] = 1;
 
         for (int i = 2; i <= n; i++)
         {
           f[i] = f[i - 1] + f[i - 2];
         }
 
         return f[n];
       }
     }
```
同样是斐波那契数列，我们使用数组 `f` 来存储计算结果，这样算法复杂度优化为 `O(n)`。

示例代码（9）：

```js
     int Fibonacci(int n)
     {
       if (n <= 1)
         return n;
       else
       {
         int iter1 = 0;
         int iter2 = 1;
         int f = 0;
 
         for (int i = 2; i <= n; i++)
         {
           f = iter1 + iter2;
           iter1 = iter2;
           iter2 = f;
         }
 
         return f;
       }
     }
```
同样是斐波那契数列，由于实际只有前两个计算结果有用，我们可以使用中间变量来存储，这样就不用创建数组以节省空间。同样算法复杂度优化为 `O(n)`。

示例代码（10）：

通过使用矩阵乘方的算法来优化斐波那契数列算法。

```js
     static int Fibonacci(int n)
     {
       if (n <= 1)
         return n;
 
       int[,] f = { { 1, 1 }, { 1, 0 } };
       Power(f, n - 1);
 
       return f[0, 0];
     }
 
     static void Power(int[,] f, int n)
     {
       if (n <= 1)
         return;
 
       int[,] m = { { 1, 1 }, { 1, 0 } };
 
       Power(f, n / 2);
       Multiply(f, f);
 
       if (n % 2 != 0)
         Multiply(f, m);
     }
 
     static void Multiply(int[,] f, int[,] m)
     {
       int x = f[0, 0] * m[0, 0] + f[0, 1] * m[1, 0];
       int y = f[0, 0] * m[0, 1] + f[0, 1] * m[1, 1];
       int z = f[1, 0] * m[0, 0] + f[1, 1] * m[1, 0];
       int w = f[1, 0] * m[0, 1] + f[1, 1] * m[1, 1];
 
       f[0, 0] = x;
       f[0, 1] = y;
       f[1, 0] = z;
       f[1, 1] = w;
     }
```
优化之后算法复杂度为 \\( O(\mathbf{log}\_{2}n) \\)。

示例代码（11）：

在 C# 中更简洁的代码如下。

```c#
     static double Fibonacci(int n)
     {
       double sqrt5 = Math.Sqrt(5);
       double phi = (1 + sqrt5) / 2.0;
       double fn = (Math.Pow(phi, n) - Math.Pow(1 - phi, n)) / sqrt5;
       return fn;
     }
```

示例代码（12）：

插入排序的基本操作就是将一个数据插入到已经排好序的有序数据中，从而得到一个新的有序数据。算法适用于少量数据的排序，时间复杂度为 \\( O(n^2) \\)。

```java
     private static void InsertionSortInPlace(int[] unsorted)
     {
       for (int i = 1; i < unsorted.Length; i++)
       {
         if (unsorted[i - 1] > unsorted[i])
         {
           int key = unsorted[i];
           int j = i;
           while (j > 0 && unsorted[j - 1] > key)
           {
             unsorted[j] = unsorted[j - 1];
             j--;
           }
           unsorted[j] = key;
         }
       }
     }
```


# 参考文献

- [原文地址](https://blog.csdn.net/zolalad/article/details/11848739)

- [参考1](http://www.cnblogs.com/songQQ/archive/2009/10/20/1587122.html)

- [参考2](ttp://www.cppblog.com/85940806/archive/2011/03/12/141672.html)
