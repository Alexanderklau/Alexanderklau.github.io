---
title: 弄懂难缠的DFS算法和相关变种(Python实现)
date: 2021-04-28 17:21:07
tags: ["算法"]
---

## 前言

这次不废话，直接接上次的BFS，直接来看DFS。

## 什么是DFS算法

DFS，全名深度优先搜索

用大白话来说，其实就是 **一条路走到黑，走不通再回来，直到无路可走**

举个简单的例子，现在我们有一个树，就像下面这样

```
     A
   /   \
  B     C
 / \   / \
D   E  F  G
```

假设我们要用DFS算法来进行遍历／搜索

它的步骤如下

1. 从初始节点**A**出发，并且将**A**标记为已访问
2. 查找**A**的一个临接顶点**B**。
3. 如果**B**存在，继续执行访问，否则回退到上一步，继续查找临接顶点
4. 将**B**标记为可访问，继续执行查找临界点**D**的操作
5. 重复操作，直到所有的值都访问完了，无值可以访问为止。

那么，遍历树的顺序应该是

A -> B -> D -> E - > C -> F -> G

是不是有点像树的前序遍历，其实差不了多少

它的遍历步骤用图来表示，就像下面这样


![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/dfs/7c01d5c3efe54b0f9f6ec5af938a8df9.png)


![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/dfs/be7a3d1472f345119c2b863e16fc27ea.png)


![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/dfs/3881d7eacb3e490fa55ff67811b069aa.png)


![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/dfs/c704cf1ebe154115bf3c0dee0dc5758e.png)


其实换一种走迷宫的说法，就是把所有能走的路都给走了。

来个图，一看你就明白了

很感谢这个图的作者！我找不到您的出处了，如果您看到，请联系我，我会加上引用！感恩！感恩！

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/dfs/alogrithm_10_3.gif)


这就是俗称的一条路走到黑，撞墙了要么回去，要么继续找路。

## DFS的实现

现在弄明白原理了，问题就在实践上了

其实看下上面的步骤，基于DFS

首先，我们需要标记一个已经访问的对象，并且持续性访问子节点，直到无法访问为止。

这种一般呼之欲出的方法就是**递归**，我们可以通过递归实现这些操作。

或者我们也可以利用**栈**实现，先将根入栈，再将根出栈，并将根的右子树，左子树存入栈，按照栈的先进后出规则来实现DFS

**栈的方法**

```python
def DFS(root):
    if root:
        res = []
        stack = [root]
        # 当 stack 有值
        while stack:
            currentNode = stack.pop()
            res.append(currentNode.val)
            if currentNode.right:
                stack.append(currentNode.right)
            if currentNode.left:
                stack.append(currentNode.left)
    return res
```

**递归的方法**

```python
def DFS(root):
    if root is not None:
        print(root.key)
        if root.left is not None:
            return DFS(root.left)
        if root.right is not None:
            return DFS(root.right)
```

## DFS的几道变种算法题

### [94. 二叉树的中序遍历](https://leetcode-cn.com/problems/binary-tree-inorder-traversal/)

给定一个二叉树的根节点 root ，返回它的 中序 遍历。

```
输入：root = [1,null,2,3]
输出：[1,3,2]
```

这是一道基础的DFS算法题，中序遍历，直接递归走你

```python
class Solution(object):
    def inorderTraversal(self, root):
        """
        :type root: TreeNode
        :rtype: List[int]
        """
        # 搞个全局的列表
        z = []

        if root == None:
            return []
        
        # 这边写个递归
        def dfs(root):
            if root:
                dfs(root.left)
                z.append(root.val)
                dfs(root.right)
        
        dfs(root)

        return z
```

### [112. 路径总和](https://leetcode-cn.com/problems/path-sum/)

给你二叉树的根节点 root 和一个表示目标和的整数 targetSum ，判断该树中是否存在 根节点到叶子节点 的路径，这条路径上所有节点值相加等于目标和 targetSum 。

叶子节点 是指没有子节点的节点。

```
输入：root = [5,4,8,11,null,13,4,7,2,null,null,null,1], targetSum = 22
输出：true
```

这是一道典型的DFS题，需要一条路走到黑那种

并且你需要记录下来到底走了那些路，能不能综合成一个总数

这边我用的方法是，所有路线走遍，分别记录，一条条再去匹配targetSum，没有匹配到，直接返回false

把上面那个模版拿过来，加两个参数

第一个，加一个stact_val，这个是用来记录和的

第二个，在每一步遍历的的时候，都去做一步加法，最后统计总量对比

```python
class Solution(object):
    def hasPathSum(self, root, targetSum):
        """
        :type root: TreeNode
        :type targetSum: int
        :rtype: bool
        """
        if not root:
            return False 
        if root:
            stack = [root]
            stack_val = [root.val]
            # 当 stack 有值
            while stack:
                currentNode = stack.pop()
                temp = stack_val.pop()
                if not currentNode.left and not currentNode.right:
                    if temp == targetSum:
                        return True
                    continue
                if currentNode.right:
                    stack.append(currentNode.right)
                    stack_val.append(currentNode.right.val + temp)
                if currentNode.left:
                    stack.append(currentNode.left)
                    stack_val.append(currentNode.left.val + temp)
        return False
```


### [129. 求根节点到叶节点数字之和](https://leetcode-cn.com/problems/sum-root-to-leaf-numbers/)

```
给你一个二叉树的根节点 root ，树中每个节点都存放有一个 0 到 9 之间的数字。
每条从根节点到叶节点的路径都代表一个数字：

例如，从根节点到叶节点的路径 1 -> 2 -> 3 表示数字 123 。
计算从根节点到叶节点生成的 所有数字之和 。

叶节点 是指没有子节点的节点。

输入：root = [1,2,3]
输出：25
解释：
从根到叶子节点路径 1->2 代表数字 12
从根到叶子节点路径 1->3 代表数字 13
因此，数字总和 = 12 + 13 = 25


输入：root = [4,9,0,5,1]
输出：1026
解释：
从根到叶子节点路径 4->9->5 代表数字 495
从根到叶子节点路径 4->9->1 代表数字 491
从根到叶子节点路径 4->0 代表数字 40
因此，数字总和 = 495 + 491 + 40 = 1026
```

这道题的解法，其实就是遍历所有子树，到底，然后把每个子树的数字组成新数字，最后返回一个和。

分析一下，按照1，2，3这个来看，上层子树永远是下层的10倍数

也就得出

(1 * 10 + 2) + (1 * 10 + 3) = 25

根据这个逻辑，开始写代码

首先把那个DFS的遍历模版拿出来

```python
def DFS(root):
    if root is not None:
        print(root.key)
        if root.left is not None:
            return DFS(root.left)
        if root.right is not None:
            return DFS(root.right)
```

改一下，先写个伪代码

```python
def dfs(root):
    if not root:
        return 0
    num = 上一个子树的节点 * 10 + root.val
    if not root.left and not root.right:
        return num
    else:
        return 左节点num + 右节点num
```

现在我们要获取到的数据就是，上一个子树的节点，这个我们决定，以参数的形式传输进去，因为第一个节点树，都是0

改一下代码

```python
def dfs(root, nodeval):
    if not root:
        return 0
    num = nodeval + root.val
    if not root.left and not root.right:
        return num
    else:
        return dfs(root.left, num) + dfs(root.right, num)
```

把我们的代码嵌入到主代码当中

```python
class Solution(object):
    def sumNumbers(self, root):
        """
        :type root: TreeNode
        :rtype: int
        """
        def dfs(root, nodeval):
            if not root:
                return 0
            total = nodeval * 10 + root.val
            if not root.left and not root.right:
                return total
            else:
                return dfs(root.left, total) + dfs(root.right, total)
        return dfs(root, 0) 
```

这样就完成了这道题。

## 总结

这段时间觉得自己越来越不努力了，我不喜欢自己这样，我一点也不喜欢。

我觉得每天我都要努力前进，后续我会持续性输出算法和其他的东西，请期待吧。

今天的freestyle说点什么呢？说点开心的事儿吧。

突然想到的小样，不代表任何事物。哈哈。

```
check it~

初见的记忆浮现

是不是爱神射错了离弦的箭？

曾几何时你我只是互相平行的线

是谁在中间接了莫名其妙的姻缘？

似乎曾经相见

是石头记下宝玉渊源？
```

最近好累，感觉真的挺累，学习不会停止，你，我，我们，都要在春暖花开的地方相见。