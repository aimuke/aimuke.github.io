---
title: "数据结构与算法： 字典树（Trie）"
tags: [algorithm, "trie", "字典树"]
---

在计算机科学中，`trie` 又称前缀树或字典树，是一种有序树，用于保存关联数组，其中的键通常是字符串。与二叉查找树不同，键不是直接保存在节点中，而是由节点在树中的位置决定。一个节点的所有子孙都有相同的前缀，也就是这个节点对应的字符串，而根节点对应空字符串。一般情况下，不是所有的节点都有对应的值，只有叶子节点和部分内部节点所对应的键才有相关的值。

`Trie` 这个术语来自于 `retrieval`。根据词源学，`trie` 的发明者`Edward Fredkin` 把它读作`/ˈtriː/ "tree"`。但是，其他作者把它读作`/ˈtraɪ/ "try"`。

`trie` 中的键通常是字符串，但也可以是其它的结构。`trie` 的算法可以很容易地修改为处理其它结构的有序序列，比如一串数字或者形状的排列。比如，`bitwise trie`中的键是一串比特，可以用于表示整数或者内存地址。

```js
     ( )
   /  |  \
  t   a   i
 / \       \
o    e      n
   / | \   /
  a  d  n  n 
```

上图是一棵 `Trie` 树，表示了关键字集合 `{“a”, “to”, “tea”, “ted”, “ten”, “i”, “in”, “inn”}` 。从上图可以归纳出 `Trie` 树的基本性质：
- 根节点不包含字符，除根节点外的每一个子节点都包含一个字符。
- 从根节点到某一个节点，路径上经过的字符连接起来，为该节点对应的字符串。
- 每个节点的所有子节点包含的字符互不相同。

通常在实现的时候，会在节点结构中设置一个标志，用来标记该结点处是否构成一个单词（关键字）。

可以看出，Trie树的关键字一般都是字符串，而且Trie树把每个关键字保存在一条路径上，而不是一个结点中。另外，两个有公共前缀的关键字，在Trie树中前缀部分的路径相同，所以Trie树又叫做前缀树（Prefix Tree）。


# 复杂度分析
**时间复杂度：**
- 假设所有字符串长度之和为 `n`，构建字典树的时间复杂度为 `O(n)`
- 假设要查找的字符串长度为 `k`，查找的时间复杂度为 `O(k)`

**空间复杂度：**

字典树每个节点都需要用一个数组来存储子节点的指针，即便实际只有两三个子节点，但依然需要一个完整大小的数组。所以，字典树比较耗内存，空间复杂度较高。

**如何优化？**

- 可以牺牲一点查询的效率，将每个节点的子节点数组用其他数据结构代替，例如有序数组，红黑树，散列表等。例如，当子节点数组采用有序数组时，可以使用二分查找来查找下一个字符。

- **缩点优化:** 将末尾一些只有一个子节点的节点，可以进行合并，但是增加了编码的难度。如下

```js
        ( )                           ()
       /   \                         /  \
     a      s                      a    siab
    / \      \                    /  \
   b   d      i                  b    drf 
  / \   \      \                / \
 c   f   r      a              c   f
          \      \
           f      b
```


# Trie树的优缺点
Trie树的核心思想是空间换时间，利用字符串的公共前缀来减少无谓的字符串比较以达到提高查询效率的目的。

## 优点
插入和查询的效率很高，都为`O(m)`，其中 `m` 是待插入/查询的字符串的长度。

关于查询，会有人说 `hash` 表时间复杂度是`O(1)`不是更快？但是，哈希搜索的效率通常取决于 `hash` 函数的好坏，若一个坏的 `hash` 函数导致很多的冲突，效率并不一定比`Trie`树高。

Trie树中不同的关键字不会产生冲突。

Trie树只有在允许一个关键字关联多个值的情况下才有类似hash碰撞发生。

Trie树不用求 hash 值，对短字符串有更快的速度。通常，求hash值也是需要遍历字符串的。

Trie树可以对关键字按字典序排序。

## 缺点
字典树的缺陷：

- 字符串的`字符集`不能过大，否则存储空间过于浪费，即便是采用优化方案，也是在牺牲部分查询性能的基础上的
- 在字符串前缀重合较多的情况，才有比较好的性能表现
- 没有现成的字典树代码库可以用，如果要使用需要手写
- 字典树中使用到了指针，因此前后节点是不连续的，对 CPU 缓存不友好
- 当 hash 函数很好时，Trie树的查找效率会低于哈希搜索。

**综上：** 在字符串的精确查找场景中，推荐使用红黑树，散列表等数据结构。

而字典树，则适合在查找前缀的场景下，例如，搜索引擎一般在输入部分字符后，会显示一些预选关键字。这些关键字均是以输入的字符为前缀。

# Trie树的应用
## 字符串检索
检索/查询功能是Trie树最原始的功能。思路就是从根节点开始一个一个字符进行比较：

如果沿路比较，发现不同的字符，则表示该字符串在集合中不存在。
如果所有的字符全部比较完并且全部相同，还需判断最后一个节点的标志位（标记该节点是否代表一个关键字）。

```go
struct trie_node
{
    bool isKey;   // 标记该节点是否代表一个关键字
    trie_node *children[26]; // 各个子节点 
};
```

这里子节点定义的是长度为 `26` 的数组，这里对应的是 `26` 个英文字母是英文字符集的数量。假如这里是中文，那么字符集的数量大概是3000左右，那么空间浪费将更加巨大，这里有两点需要注意一下：
- 实际上字符串中可能包含英文字母以外的字符，所以这里只是举例而已
- 声明一个节点的时候，需要同时声明完整的子节点数组，而且是以 `字符集数量` 为底指数上升的，所以在空间上会有浪费，这里可以体现空间换时间。

## 词频统计
Trie树常被搜索引擎系统用于文本词频统计。为了实现词频统计，在节点中提那家一个一个整型变量count来计数。对每一个关键字执行插入操作，若已存在，计数加`1`，若不存在，插入后count置`1`。

```go
struct trie_node
{
    int count;   // 记录该节点代表的单词的个数
    trie_node *children[26]; // 各个子节点 
};
```

注意：字符串检索和词频统计也都可以用 `hash table` 来做。

## 字符串排序
Trie树可以对大量字符串按字典序进行排序，思路也很简单：遍历一次所有关键字，将它们全部插入trie树，树的每个结点的所有儿子很显然地按照字母表排序，然后先序遍历输出Trie树中所有关键字即可。

## 前缀匹配
例如：找出一个字符串集合中所有以ab开头的字符串。我们只需要用所有字符串构造一个trie树，然后输出以 `a->b->` 开头的路径上的关键字即可。

trie树前缀匹配常用于搜索提示。如当输入一个网址，可以自动搜索出可能的选择。当没有完全匹配的搜索结果，可以返回前缀最相似的可能。

在大多手机输入法中, 都会用9格的那种输入法. 这个输入法能够根据用户在9格上的输入,自动匹配出可能的单词。 

填单词游戏 相信大多数人都玩过那种在横竖的格子里填单词的游戏

求某两个串的最长公共前缀的长度是多少?对所有串建立字典树，对于两个串的最长公共前缀的长度即他们所在的结点的公共祖先个数，于是，问题就转化为最近公共祖先问题。
## IP路由表 
在IP路由表中进行路由匹配时, 要按照最长匹配前缀的原则进行匹配。 

## 拼写检查 
例如,在word中输入一个拼写错误的单词, 它能够自动检测出来。 

## 辅助结构
作为其他数据结构和算法的辅助结构如后缀树，AC自动机等。

# demo
## Implement Trie

[208. Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/description/)

Implement a trie with insert, search, and startsWith methods.

Example:

Trie trie = new Trie();

trie.insert("apple");
trie.search("apple");   // returns true
trie.search("app");     // returns false
trie.startsWith("app"); // returns true
trie.insert("app");   
trie.search("app");     // returns true
Note:

You may assume that all inputs are consist of lowercase letters a-z.
All inputs are guaranteed to be non-empty strings.

## Word Search II
[212. Word Search II](https://leetcode.com/problems/word-search-ii/description/)

Given a 2D board and a list of words from the dictionary, find all words in the board.

Each word must be constructed from letters of sequentially adjacent cell, where "adjacent" cells are those horizontally or vertically neighboring. The same letter cell may not be used more than once in a word.

Example:
```
Input: 
board = [
  ['o','a','a','n'],
  ['e','t','a','e'],
  ['i','h','k','r'],
  ['i','f','l','v']
]
words = ["oath","pea","eat","rain"]

Output: ["eat","oath"]
```
Note:

All inputs are consist of lowercase letters a-z.
The values of words are distinct.

`adjacent` [əˈdʒeɪsnt] adj. 与…毗连的;邻近的

## Word Search
[79. Word Search](https://leetcode.com/problems/word-search/description/)

Given a 2D board and a word, find if the word exists in the grid.

The word can be constructed from letters of sequentially adjacent cell, where "adjacent" cells are those horizontally or vertically neighboring. The same letter cell may not be used more than once.

Example:
```
board =
[
  ['A','B','C','E'],
  ['S','F','C','S'],
  ['A','D','E','E']
]

Given word = "ABCCED", return true.
Given word = "SEE", return true.
Given word = "ABCB", return false.
```

# 参考文献

- [维基百科](https://zh.wikipedia.org/wiki/Trie)

- [Trie树（Prefix Tree）介绍](https://blog.csdn.net/lisonglisonglisong/article/details/45584721)

- [Trie 树构造原理、应用场景与复杂度分析](https://blog.csdn.net/m0_37264516/article/details/86028794)

- [Trie (Prefix Tree) 前缀树](https://blog.csdn.net/l947069962/article/details/77650918)

- [Trie Tree(Prefix Tree)](https://blog.csdn.net/wenwen1538/article/details/46557639)
