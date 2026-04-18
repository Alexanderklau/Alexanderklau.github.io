---
title: 弄懂难缠的BFS算法和相关变种(Python实现)
date: 2021-02-23 16:22:51
tags: ["算法"]
---

## 前言

这段时间频繁刷题，leetcode真的好难啊！！每次都他娘的做不出来，除了刷题，最近还在复习各种架构，或者是完成公司的开发。这些占据了我过多时间，所以blog其实一直想写，但是实在腾不出时间，今天在针对性刷leetcode的时候，对BFS/DFS有了一点别的感悟，所以就写一篇博客，作为自己的笔记，在记录的同时，也帮助其他兄弟少走弯路，希望，能够帮到大家。

## 什么是BFS算法？

这些百度谷歌都搜的到，不过这里还是简单说一下吧

首先，BFS的全名，叫做广度优先搜索算法

搜索，顾名思义，是找寻某个东西，所以叫搜索，在写代码的时候，搜索，其实等同遍历，只是这个遍历是有条件的。

相对于BFS，它的条件是什么呢？

举个例子，有个迷宫，有两个点，你要从A 移动到 B，中间带X的表示墙壁，无法通行

```
A 0 0 0 

X 0 0 X

0 X 0 B
```

现在走出第一步

```
A 1 0 0 

X 0 0 X

0 X 0 B
```

走现在走第二步，发现有两个可以走的地方（分岔路）

```
A 1 2 0 

X 2 0 X

0 X 0 B
```

走第三步，依旧有个分岔路

```
A 1 2 3 

X 2 3 X

0 X 0 B
```

走第四步，只有一条路

```
A 1 2 3 

X 2 3 X

0 X 4 B
```

第五步，走到B点

```
A 1 2 3 

X 2 3 X

0 X 4 B
```
我们画出可行的步骤

```
A-1-2 3 
  | |
X 2-3 X
    |
0 X 4-B
```

BFS（广度优先）算法，就是记录所有可行的路径。

当面临选择和岔路的时候，BFS选择**我全都要**，全都记录下来，然后选择其中一个进入，如果有死路，它选择返回，选另外的岔路，继续重复这样的操作。

可以看出，BFS是逐步求解的，由近到远。这里找路程的步骤，用动态图来展示如下

这个图其实就直接说明了BFS算法的特点

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/bfs/alogrithm_10_2.gif)

这个图真的很棒！我一看就大概明白BFS算法了。
感谢此图的作者，如果你看到，请联系我，我会加上你的名字作为引用！谢谢！

是不是很像钢铁雄心4推进部队的样子？

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/bfs/hoi4.jpg)

持续性突进！

## 树的BFS算法

前面你可以看到，BFS算法，其实就是将分支逐步列出。

其实对于树这个结构来说，BFS算法更像是**特殊的**遍历整个树结构的一种方法。

举个例子

假设我们弄一个树
```
      A
    /   \
   B     C
  / \   / \
 D   E F   G
```
一般的遍历逻辑就是(前序遍历)

A - B - D - E - C - F - G


但是，BFS可不是这么玩的，BFS的遍历逻辑就是
A - B - C - D - E - F - G

这里来个图，你一看就明白！

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/bfs/e129246930f743eea7807c93b2ca1739.gif)

这就是树的BFS遍历，其实说起来也没多难吧？


### 树的BFS算法模版

根据上面的逻辑，我们可以看出来，树的BFS算法是一层层的层次搜索算法，并且是先进先出的逻辑。

如图所示

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/bfs/f799f2d3440f4cddb47d0b1e28d8198d.gif)

那么，遍历树的BFS的模版，应该这么写

首先，定义一个树的结构体

```python
class TreeNode:
    def __init__(self, x):
        self.val = x
        self.left = None
        self.right = None
```

由于**先进先出**的结构特点，这里使用**队列**来进行存储数据，这是因为BFS算法需要保证优先访问顶点的未访问领接点。

```python

def BFS(root):
    if root == None:
        return

    # 队列化root
    queue = deque([root])

    result = []

    # 遍历root
    while queue:
        # 移去并且返回一个元素，queue 最左侧的那一个
        node = queue.popleft()
        # 获取node的详细情况
        result.append(node.val)
        print(node.val)
        # 访问左树
        left = node.left
        if left != None:
            queue.append(left)
        # 访问右树
        right = node.right
        if right != None:
            queue.append(right)
    return result
```

写一个测试的方法，就按着我们刚刚那个A - G的树来一把

```python
if __name__ == "__main__":
    tree = TreeNode("A")
    tree.left = TreeNode("B")
    tree.right = TreeNode("C")
    tree.left.left = TreeNode("D")
    tree.right.right = TreeNode("E")
    tree.right.right.right = TreeNode("F")
    tree.right.right.right = TreeNode("F")
    print(BFS(tree))
```

打印出来

```
A
B
C
D
E
F
['A', 'B', 'C', 'D', 'E', 'F']
```

### 树的BFS算法变种

#### 从上到下打印二叉树 III

leetcode地址：https://leetcode-cn.com/problems/cong-shang-dao-xia-da-yin-er-cha-shu-iii-lcof/

```
请实现一个函数按照之字形顺序打印二叉树，

即第一行按照从左到右的顺序打印，

第二层按照从右到左的顺序打印，

第三行再按照从左到右的顺序打印，其他行以此类推。

 

例如:
给定二叉树: 

[3,9,20,null,null,15,7],

    3
   / \
  9  20
    /  \
   15   7
返回其层次遍历结果：

[
  [3],
  [20,9],
  [15,7]
]
```

**分析题目**

这道题，主要就是用BFS去做遍历，然后遍历的同时判断现在遍历到第几层，然后根据层数转换打印的次序，这道题不算太难

首先先写个BFS模版

```python
queue = deque([root])

result = []

while queue:
    node = queue.popleft()
    result.append(node.val)
    print(node.val)
    left = node.left
    if left != None:
        queue.append(left)
    right = node.right
    if right != None:
        queue.append(right)

return result
```

根据需求，首先先解决输出是

```
[
  [3],
  [20,9],
  [15,7]
]
```
的问题，其实说明内部嵌套多个队列，改写一下，当遍历每一层的时候，添加到新的队列当中

```python
while queue:
    # 定义一个新的队列
    tmp = deque()
    # 判断队列循环到哪里
    for i in range(len(queue)):
        node = queue.popleft()
        # tmp 队列 添加 val数据
        tmp.append(node.val)
        left = node.left
        if left != None:
            queue.append(left)
        right = node.right
        if right != None:
            queue.append(right)
    result.append(list(tmp))
return result
```

接下来就要判断循环的行号是偶数还是计数

```python
# 如果是偶数
if len(result) % 2:
    # 添加到队列左端
    tmp.appendleft(node.val)
else:
    tmp.append(node.val)
```

最终的代码是

```python
from collections import deque

class TreeNode:
    def __init__(self, x):
        self.val = x
        self.left = None
        self.right = None

def BFS(root):
    if root == None:
        return []

    queue = deque([root])

    result = []

    while queue:
        tmp = deque()
        for i in range(len(queue)):
            print(len(result))
            node = queue.popleft()
            if len(result) % 2:
                tmp.appendleft(node.val)
            else:
                tmp.append(node.val)
            if node.left != None:
                queue.append(node.left)
            if node.right != None:
                queue.append(node.right)
        result.append(list(tmp))
    return result
```

## 结尾
我这里只总结了二叉树的几个题目，其实还不是很全，最近实在是太忙啦！后面会补全图算法的BFS和DFS算法，大家多多期待吧！