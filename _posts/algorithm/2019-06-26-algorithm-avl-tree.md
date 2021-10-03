---
title: "平衡二叉树 AVL树"
tags: [algorithm, "avl tree", "平衡二叉树"]
---

学习过了二叉查找树，想必大家有遇到一个问题。例如，将一个数组 `{1,2,3,4}` 依次插入树的时候，形成了如下(a)的情况。有建立树与没建立树对于数据的增删查改已经没有了任何帮助，反而增添了维护的成本。而只有建立的树如下（b），才能够最大地体现二叉树的优点。

```js
 1              2
  \            / \
   2          1   3
    \              \
     3              4
      \
       4
  (a)           (b) 
```

在上述的例子中，(b)就是一棵平衡二叉树。科学家们提出平衡二叉树，就是为了让树的查找性能得到最大的体现。下面进入今天的正题，平衡二叉树。

# AVL的定义

**平衡二叉查找树**：简称平衡二叉树。由前苏联的数学家 `Adelse-Velskil` 和 `Landis` 在1962年提出的高度平衡的二叉树，根据科学家的英文名也称为 `AVL树`。其平均和最坏情况下的查找时间都是O(logn)。它具有如下几个性质：
- 可以是空树。
- 假如不是空树，任何一个结点的左子树与右子树都是平衡二叉树，并且高度之差的绝对值不超过1

那么在建立树的过程中，我们如何知道左右子树的高度差呢？在这里我们采用了平衡因子进行记录。

**平衡因子：** 左子树的高度减去右子树的高度。由平衡二叉树的定义可知，平衡因子的取值只可能为 `0,1,-1`.分别对应着左右子树等高，左子树比较高，右子树比较高。

对于给定结点数为 `n` 的AVL树，最大高度为 \\( O(\mathbf{log}\_{2}n) \\).

如下:
- (a) 中节点 3的左子树高度为 2， 右子树高度为0，相差2，不平衡。 节点5 相反也不平衡。
- (b) 各节点的平衡因子都为 `0,1,-1`之间，平衡。
- (c) 节点 7 的左子树高度为2， 右子树高度为1，也不平衡

```js
       4             6               7
      / \           / \            /  \
     3   5         4   8          2    8 
    /     \       / \   \        / \
   2       6     1   5   9      1   5
  /         \     \                /  \
 1           7     2              4    6
    (a)              (b)            (c)
```

# 失衡与调整

说了这么久，我们开始进入今天的重点，如何将一棵不平衡的二叉树变成平衡二叉树。平衡二叉树的失衡调整主要是通过旋转最小失衡子树来实现的。

**最小失衡子树**：在新插入的结点向上查找，以第一个平衡因子的绝对值超过1的结点为根的子树称为最小不平衡子树。

也就是说，一棵失衡的树，是有可能有多棵子树同时失衡的，这个时候，我们只要调整最小的不平衡子树，就能够将不平衡的树调整为平衡的树。


## 左旋(Left Rotation)

当新插入的节点为`右子树的右子节点`时，我们需要进行左旋操作来保证此部分子树继续处于平衡状态。
```js
A (2)     A  
 \         \  
  B (1)     B  ↖        B(0)
   \         \         /    \
    C(0)      C      A(0)   C(0)
右侧不平衡   左旋     平衡后的树

```

比如:
```js
   50 (非平衡节点)
  /  \
40    60                         60 
      / \                      /    \
    55   70                   50     70
          \                  /  \      \
           80(新)           40   55     80 

```

## 右旋(Right Rotation)
当新插入的结点为左子树的左子结点时，我们需要进行右旋操作来保证此部分子树继续处于平衡状态。

```js
     A (2)     A  
    /         /  
   B (1)   ↗ B         B(0)
  /         /         /    \
 C(0)      C        A(0)   C(0)
左侧不平衡   右旋     平衡后的树

```


比如:
```js
        50 (非平衡节点)
       /  \
     40    60                    40 
    /  \                       /    \
  30    45                   30     50
 /                          /       /  \
20(新)                   (新)20   45    60

```

## 左右旋(Left-Right Rotation)
新节点被插入到左子树的有节点时，需要先左旋在右旋

```js
   A(2)
  /
B(1)
  \
   C(0)
```

此时需要先以B为轴进行左旋操作

```js
    A(2)                A(2)
   /                    /
↙ B(1)          -->   C(1)  
    \                  /
     C(0)  ↖         B(0)
```

然后在以A为轴进行右旋操作

```js
     A(2)  ↘
    /
   C(1)      -->    C(0)
  /                 /  \
B(0)              B(0)  A(0)
```

比如:
```js
        50 (不平衡点)         50   ↘            
       /  \                 / \
  ↙  40    60              45  60                45
    /  \                 /    \                /    \
  30    45              40     47(新)         40     50
         \              /                   /      /    \
          47(新)       30                   30   47(新)  60     

```
上例中， 节点`50` 为最小失衡子树的根节点。新节点 `47` 是 节点 `50` 的左子节点的右子节点。先以节点`40` 为轴进行左旋，然后在以节点 `50`为轴进行右旋转。

需要注意的是，以 `50`为轴进行旋转的时候，做了两件事： 1） 先将节点`50`的左节点置为 `45`的右节点； 2） 再将节点 `45`的右节点置为 节点 `50`.

## 右左旋(Right-Left Rotation)
同左右的情况，插入节点为不平衡节点的右节点的左节点, 其处理方法为先右旋在左旋。

```js
A(2)
  \
   B(1) 
   / 
C(0)
```

```js
A           A  
  \          \
   B ↘        C  ↖          C(0)
   /           \            /   \
↗ C             B         A(0)  B(0)
```


比如:
```js
        50 (不平衡点)         50   ↖            
       /  \                 /   \
     40    60  ↘          40     55                 55
          /  \                 /    \             /    \
         55   70            53(新)   60         50      60
         /                            \         / \      \
       53(新)                          70     40  53(新)  70     
```

# 实现

## 左旋
```java
    /**
     * @param n
     * @return
     * @function 左旋操作
     */
    private AVLNode rotateLeft(AVLNode n) {

        //指向当前节点的右孩子
        AVLNode top = n.right;

        //将当前节点的右孩子挂载到当前节点的父节点
        top.parent = n.parent;

        //如果当前节点的父节点不为空
        if (top.parent != null) {
            if (top.parent.right == n) {
                top.parent.right = top;
            } else {
                top.parent.left = top;
            }
        }

        //将原本节点的右孩子挂载到新节点的左孩子
        n.right = top.left;

        if (n.right != null)
            n.right.parent = n;

        //将原本节点挂载到新节点的左孩子上
        top.left = n;

        //将原本节点的父节点设置为新节点
        n.parent = top;

        //重新计算每个节点的平衡度
        setBalance(n, top);

        return top;
    }
```

## 右旋

```java
    private AVLNode rotateRight(AVLNode n) {
        // 设置新的top节点
        AVLNode top = n.left;
        top.parent = n.parent;

        if (top.parent != null) {
            if (top.parent.right == n) {
                top.parent.right = top;
            } else {
                top.parent.left = top;
            }
        }

        // 设置右旋后节点的左节点
        n.left = top.right;

        if (n.left != null)
            n.left.parent = n;

        // 设置新top节点的右节点
        top.right = n;
        n.parent = top;

        setBalance(n, top);

        return top;
    }
```

## 左右旋

```java
    private AVLNode rotateLeftThenRight(AVLNode n) {
        n.left = rotateLeft(n.left);
        return rotateRight(n);
    }
```

## 右左旋
```java
    private AVLNode rotateRightThenLeft(AVLNode n) {
        n.right = rotateRight(n.right);
        return rotateLeft(n);
    }
```

## 重平衡
对节点进行重平衡操作，从传入节点开始，对所有不平衡子节点递归进行平衡操作
```java
    /**
     * @param n
     * @function 重平衡该树
     */
    private void rebalance(AVLNode n) {

        //为每个节点设置相对高度
        setBalance(n);

        //如果左子树高于右子树
        if (n.balance == -2) {

            //如果挂载的是左子树的左孩子
            if (height(n.left.left) >= height(n.left.right))

                //进行右旋操作
                n = rotateRight(n);
            else

                //如果挂载的是左子树的右孩子,则先左旋后右旋
                n = rotateLeftThenRight(n);

        }
        //如果左子树高于右子树
        else if (n.balance == 2) {

            //如果挂载的是右子树的右孩子
            if (height(n.right.right) >= height(n.right.left))

                //进行左旋操作
                n = rotateLeft(n);
            else

                //否则进行先右旋后左旋
                n = rotateRightThenLeft(n);
        }

        if (n.parent != null) {

            //如果当前节点的父节点不为空,则平衡其父节点
            rebalance(n.parent);
        } else {
            root = n;
        }
    }
```

## 完整代码
```java

package wx.algorithm.search.avl;

/**
 * Created by apple on 16/7/30.
 */
public class AVLTree {

    //指向当前AVL树的根节点
    private AVLNode root;

    /**
     * @param key
     * @return
     * @function 插入函数
     */
    public boolean insert(int key) {

        //如果当前根节点为空,则直接创建新节点
        if (root == null)
            root = new AVLNode(key, null);
        else {

            //设置新的临时节点
            AVLNode n = root;

            //指向当前的父节点
            AVLNode parent;

            //循环直至找到合适的插入位置
            while (true) {

                //如果查找到了相同值的节点
                if (n.key == key)

                    //则直接报错
                    return false;

                //将当前父节点指向当前节点
                parent = n;

                //判断是移动到左节点还是右节点
                boolean goLeft = n.key > key;
                n = goLeft ? n.left : n.right;

                //如果左孩子或者右孩子为空
                if (n == null) {
                    if (goLeft) {
                        //将节点挂载到左孩子上
                        parent.left = new AVLNode(key, parent);
                    } else {
                        //否则挂载到右孩子上
                        parent.right = new AVLNode(key, parent);
                    }

                    //重平衡该树
                    rebalance(parent);
                    break;
                }

                //如果不为空,则以n为当前节点进行查找
            }
        }
        return true;
    }

    /**
     * @param delKey
     * @function 根据关键值删除某个元素, 需要对树进行再平衡
     */
    public void delete(int delKey) {
        if (root == null)
            return;
        AVLNode n = root;
        AVLNode parent = root;
        AVLNode delAVLNode = null;
        AVLNode child = root;

        while (child != null) {
            parent = n;
            n = child;
            child = delKey >= n.key ? n.right : n.left;
            if (delKey == n.key)
                delAVLNode = n;
        }

        if (delAVLNode != null) {
            delAVLNode.key = n.key;

            child = n.left != null ? n.left : n.right;

            if (root.key == delKey) {
                root = child;
            } else {
                if (parent.left == n) {
                    parent.left = child;
                } else {
                    parent.right = child;
                }
                rebalance(parent);
            }
        }
    }

    /**
     * @function 打印节点的平衡度
     */
    public void printBalance() {
        printBalance(root);
    }

    /**
     * @param n
     * @function 重平衡该树
     */
    private void rebalance(AVLNode n) {

        //为每个节点设置相对高度
        setBalance(n);

        //如果左子树高于右子树
        if (n.balance == -2) {

            //如果挂载的是左子树的左孩子
            if (height(n.left.left) >= height(n.left.right))

                //进行右旋操作
                n = rotateRight(n);
            else

                //如果挂载的是左子树的右孩子,则先左旋后右旋
                n = rotateLeftThenRight(n);

        }
        //如果左子树高于右子树
        else if (n.balance == 2) {

            //如果挂载的是右子树的右孩子
            if (height(n.right.right) >= height(n.right.left))

                //进行左旋操作
                n = rotateLeft(n);
            else

                //否则进行先右旋后左旋
                n = rotateRightThenLeft(n);
        }

        if (n.parent != null) {

            //如果当前节点的父节点不为空,则平衡其父节点
            rebalance(n.parent);
        } else {
            root = n;
        }
    }

    
    /**
     * @param n
     * @return
     * @function 左旋操作
     */
    private AVLNode rotateLeft(AVLNode n) {

        //指向当前节点的右孩子
        AVLNode top = n.right;

        //将当前节点的右孩子挂载到当前节点的父节点
        top.parent = n.parent;

        //如果当前节点的父节点不为空
        if (top.parent != null) {
            if (top.parent.right == n) {
                top.parent.right = top;
            } else {
                top.parent.left = top;
            }
        }

        //将原本节点的右孩子挂载到新节点的左孩子
        n.right = top.left;

        if (n.right != null)
            n.right.parent = n;

        //将原本节点挂载到新节点的左孩子上
        top.left = n;

        //将原本节点的父节点设置为新节点
        n.parent = top;

        //重新计算每个节点的平衡度
        setBalance(n, top);

        return top;
    }

    private AVLNode rotateRight(AVLNode n) {
        // 设置新的top节点
        AVLNode top = n.left;
        top.parent = n.parent;

        if (top.parent != null) {
            if (top.parent.right == n) {
                top.parent.right = top;
            } else {
                top.parent.left = top;
            }
        }

        // 设置右旋后节点的左节点
        n.left = top.right;

        if (n.left != null)
            n.left.parent = n;

        // 设置新top节点的右节点
        top.right = n;
        n.parent = top;

        setBalance(n, top);

        return top;
    }

    private AVLNode rotateLeftThenRight(AVLNode n) {
        n.left = rotateLeft(n.left);
        return rotateRight(n);
    }

    private AVLNode rotateRightThenLeft(AVLNode n) {
        n.right = rotateRight(n.right);
        return rotateLeft(n);
    }

    /**
     * @param n
     * @return
     * @function 计算某个节点的高度
     */
    private int height(AVLNode n) {
        if (n == null)
            return -1;
        return 1 + Math.max(height(n.left), height(n.right));
    }

    /**
     * @param AVLNodes
     * @function 重设置每个节点的平衡度
     */
    private void setBalance(AVLNode... AVLNodes) {
        for (AVLNode n : AVLNodes)
            n.balance = height(n.right) - height(n.left);
    }

    private void printBalance(AVLNode n) {
        if (n != null) {
            printBalance(n.left);
            System.out.printf("%s ", n.balance);
            printBalance(n.right);
        }
    }
}
```

# 参考文献

- [平衡二叉树,AVL树之图解篇](https://www.cnblogs.com/suimeng/p/4560056.html)

- [AVL平衡二叉树详解与实现](https://segmentfault.com/a/1190000006123188)

- [6天通吃树结构—— 第二天 平衡二叉树](https://www.cnblogs.com/huangxincheng/archive/2012/07/22/2603956.html)
