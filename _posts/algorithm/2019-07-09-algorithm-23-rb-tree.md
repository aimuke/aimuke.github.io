---
title: "浅谈算法和数据结构: 九 平衡查找树之红黑树"
tags: [algorithm, "red-black-tree", "红黑树", "todo"]
---

前面一篇文章介绍了2-3查找树，可以看到，2-3查找树能保证在插入元素之后能保持树的平衡状态，最坏情况下即所有的子节点都是2-node，树的高度为lgN，从而保证了最坏情况下的时间复杂度。但是2-3树实现起来比较复杂，本文介绍一种简单实现2-3树的数据结构，即红黑树（Red-Black Tree）

# 定义
红黑树的主要是想对2-3查找树进行编码，尤其是对2-3查找树中的3-nodes节点添加额外的信息。红黑树中将节点之间的链接分为两种不同类型，红色链接，他用来链接两个2-nodes节点来表示一个3-nodes节点。黑色链接用来链接普通的2-3节点。特别的，使用红色链接的两个2-nodes来表示一个3-nodes节点，并且向左倾斜，即一个2-node是另一个2-node的左子节点。这种做法的好处是查找的时候不用做任何修改，和普通的二叉查找树相同。

![Red black tree](https://images0.cnblogs.com/blog/94031/201403/270024368439888.png)

根据以上描述，红黑树定义如下：

红黑树是一种具有红色和黑色链接的平衡查找树，同时满足：

- 红色节点向左倾斜
- 一个节点不可能有两个红色链接
- 整个书完全黑色平衡，即从根节点到所以叶子结点的路径上，黑色链接的个数都相同。

下图可以看到红黑树其实是2-3树的另外一种表现形式：如果我们将红色的连线水平绘制，那么他链接的两个2-node节点就是2-3树中的一个3-node节点了。

![1-1 correspondence between 2-3 and LLRB](https://images0.cnblogs.com/blog/94031/201403/270024403113529.png)

表示
我们可以在二叉查找树的每一个节点上增加一个新的表示颜色的标记。该标记指示该节点指向其父节点的颜色。

```java
private const bool RED = true;
private const bool BLACK = false;

private Node root;

class Node
{
    public Node Left { get; set; }
    public Node Right { get; set; }
    public TKey Key { get; set; }
    public TValue Value { get; set; }
    public int Number { get; set; }
    public bool Color { get; set; }

    public Node(TKey key, TValue value,int number, bool color)
    {
        this.Key = key;
        this.Value = value;
        this.Number = number;
        this.Color = color;
    }
}

private bool IsRed(Node node)
{
    if (node == null) return false;
    return node.Color == RED;
}
```

![Red black tree representation](https://images0.cnblogs.com/blog/94031/201403/270024427494883.png)

# 实现
## 查找

红黑树是一种特殊的二叉查找树，他的查找方法也和二叉查找树一样，不需要做太多更改。

但是由于红黑树比一般的二叉查找树具有更好的平衡，所以查找起来更快。

```java
//查找获取指定的值
public override TValue Get(TKey key)
{
    return GetValue(root, key);
}

private TValue GetValue(Node node, TKey key)
{
    if (node == null) return default(TValue);
    int cmp = key.CompareTo(node.Key);
    if (cmp == 0) return node.Value;
    else if (cmp > 0) return GetValue(node.Right, key);
    else return GetValue(node.Left, key);
}
```

## 平衡化
在介绍插入之前，我们先介绍如何让红黑树保持平衡，因为一般的，我们插入完成之后，需要对树进行平衡化操作以使其满足平衡化。

### 旋转
旋转又分为左旋和右旋。通常左旋操作用于将一个向右倾斜的红色链接旋转为向左链接。对比操作前后，可以看出，该操作实际上是将红线链接的两个节点中的一个较大的节点移动到根节点上。

左旋操作如下图：

![before left rotation after left rotation](https://images0.cnblogs.com/blog/94031/201403/270024451717710.png)


右旋是左旋的逆操作，过程如下：

![before right rotation after right rotation](https://images0.cnblogs.com/blog/94031/201403/270024587968114.png)

### 颜色反转

当出现一个临时的4-node的时候，即一个节点的两个子节点均为红色，如下图：

![before flip colors after flip colors](https://images0.cnblogs.com/blog/94031/201403/270025071557693.png)

这其实是个`A，E，S 4-node`连接，我们需要将 `E` 提升至父节点，操作方法很简单，就是把 `E` 对子节点的连线设置为黑色，自己的颜色设置为红色。

有了以上基本操作方法之后，我们现在对应之前对2-3树的平衡操作来对红黑树进行平衡操作，这两者是可以一一对应的，如下图：

![RB tree 1-1 correspondence with 2-3 tree](https://images0.cnblogs.com/blog/94031/201403/270025128279460.png)

现在来讨论各种情况：

**Case 1 往一个2-node节点底部插入新的节点**

先热身一下，首先我们看对于只有一个节点的红黑树，插入一个新的节点的操作：

![Insert into a tree with only 1 node](https://images0.cnblogs.com/blog/94031/201403/270025157183445.png)

这种情况很简单，只需要：

- 标准的二叉查找树遍历即可。新插入的节点标记为红色
- 如果新插入的节点在父节点的右子节点，则需要进行左旋操作

**Case 2往一个3-node节点底部插入新的节点**

先热身一下，假设我们往一个只有两个节点的树中插入元素，如下图，根据待插入元素与已有元素的大小，又可以分为如下三种情况：

![Insert into a tree with only 2 nodes](https://images0.cnblogs.com/blog/94031/201403/270025185155457.png)

- 如果带插入的节点比现有的两个节点都大，这种情况最简单。我们只需要将新插入的节点连接到右边子树上即可，然后将中间的元素提升至根节点。这样根节点的左右子树都是红色的节点了，我们只需要调研FlipColor方法即可。其他情况经过反转操作后都会和这一样。

- 如果插入的节点比最小的元素要小，那么将新节点添加到最左侧，这样就有两个连接红色的节点了，这是对中间节点进行右旋操作，使中间结点成为根节点。这是就转换到了第一种情况，这时候只需要再进行一次FlipColor操作即可。

- 如果插入的节点的值位于两个节点之间，那么将新节点插入到左侧节点的右子节点。因为该节点的右子节点是红色的，所以需要进行左旋操作。操作完之后就变成第二种情况了，再进行一次右旋，然后再调用FlipColor操作即可完成平衡操作。

有了以上基础，我们现在来总结一下往一个3-node节点底部插入新的节点的操作步骤，下面是一个典型的操作过程图：

![Insert into 3-node at the bottom](https://images0.cnblogs.com/blog/94031/201403/270025211557283.png)

可以看出，操作步骤如下：

- 执行标准的二叉查找树插入操作，新插入的节点元素用红色标识。
- 如果需要对4-node节点进行旋转操作
- 如果需要，调用FlipColor方法将红色节点提升
- 如果需要，左旋操作使红色节点左倾。
- 在有些情况下，需要递归调用Case1 Case2，来进行递归操作。如下：

![Insert into 3-node at the bottom case 2](https://images0.cnblogs.com/blog/94031/201403/270025244219452.png)

## 代码实现
经过上面的平衡化讨论，现在就来实现插入操作，一般地插入操作就是先执行标准的二叉查找树插入，然后再进行平衡化。对照2-3树，我们可以通过前面讨论的，左旋，右旋，FlipColor这三种操作来完成平衡化。

![Passing a red link up a red-black BST](https://images0.cnblogs.com/blog/94031/201403/270025266714565.png)

具体操作方式如下：

- 如果节点的右子节点为红色，且左子节点位黑色，则进行左旋操作
- 如果节点的左子节点为红色，并且左子节点的左子节点也为红色，则进行右旋操作
- 如果节点的左右子节点均为红色，则执行FlipColor操作，提升中间结点。

# 分析
对红黑树的分析其实就是对2-3查找树的分析，红黑树能够保证符号表的所有操作即使在最坏的情况下都能保证对数的时间复杂度，也就是树的高度。

红黑树在各种情况下都能维护良好的平衡性，从而能够保证最差情况下的查找，插入效率。

下面来详细分析下红黑树的效率：

1. 在最坏的情况下，红黑树的高度不超过 `2lgN`
> 最坏的情况就是，红黑树中除了最左侧路径全部是由3-node节点组成，即红黑相间的路径长度是全黑路径长度的2倍。
>
>下图是一个典型的红黑树，从中可以看到最长的路径(红黑相间的路径)是最短路径的2倍：
>
>a typic red black tree

2. 红黑树的平均高度大约为 `lgN`

> 下图是红黑树在各种情况下的时间复杂度，可以看出红黑树是2-3查找树的一种实现，他能保证最坏情况下仍然具有对数的时间复杂度。
>
>下图是红黑树各种操作的时间复杂度。
>
> ![analysis of red black tree](https://images0.cnblogs.com/blog/94031/201403/270027393273238.png)

# 应用
红黑树这种数据结构应用十分广泛，在多种编程语言中被用作符号表的实现，如：

- Java中的java.util.TreeMap,java.util.TreeSet
- C++ STL中的：map,multimap,multiset
- .NET中的：SortedDictionary,SortedSet 等

下面以.NET中为例，通过Reflector工具，我们可以看到SortedDictionary的Add方法如下：

```java
public void Add(T item)
{
    if (this.root == null)
    {
        this.root = new Node<T>(item, false);
        this.count = 1;
    }
    else
    {
        Node<T> root = this.root;
        Node<T> node = null;
        Node<T> grandParent = null;
        Node<T> greatGrandParent = null;
        int num = 0;
        while (root != null)
        {
            num = this.comparer.Compare(item, root.Item);
            if (num == 0)
            {
                this.root.IsRed = false;
                ThrowHelper.ThrowArgumentException(ExceptionResource.Argument_AddingDuplicate);
            }
            if (TreeSet<T>.Is4Node(root))
            {
                TreeSet<T>.Split4Node(root);
                if (TreeSet<T>.IsRed(node))
                {
                    this.InsertionBalance(root, ref node, grandParent, greatGrandParent);
                }
            }
            greatGrandParent = grandParent;
            grandParent = node;
            node = root;
            root = (num < 0) ? root.Left : root.Right;
        }
        Node<T> current = new Node<T>(item);
        if (num > 0)
        {
            node.Right = current;
        }
        else
        {
            node.Left = current;
        }
        if (node.IsRed)
        {
            this.InsertionBalance(current, ref node, grandParent, greatGrandParent);
        }
        this.root.IsRed = false;
        this.count++;
        this.version++;
    }
}
```

可以看到，内部实现也是一个红黑树，其操作方法和本文将的大同小异，感兴趣的话，您可以使用Reflector工具跟进去查看源代码。

# 总结

前文讲解了自平衡查找树中的2-3查找树，这种数据结构在插入之后能够进行自平衡操作，从而保证了树的高度在一定的范围内进而能够保证最坏情况下的时间复杂度。但是2-3查找树实现起来比较困难，红黑树是2-3树的一种简单高效的实现，他巧妙地使用颜色标记来替代2-3树中比较难处理的3-node节点问题。红黑树是一种比较高效的平衡查找树，应用非常广泛，很多编程语言的内部实现都或多或少的采用了红黑树。

希望本文对您了解红黑树有所帮助，下文将介绍在文件系统以及数据库系统中应用非常广泛的另外一种平衡树结构：B树。

# 参考文献

- [原文地址](https://www.cnblogs.com/yangecnu/p/Introduce-Red-Black-Tree.html)
