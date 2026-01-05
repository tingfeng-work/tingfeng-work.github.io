---
title: 二叉树的层序遍历总结（2026-01-05）
date: 2026-01-05 13:00
author: tingfeng
categories:
  - 算法
  - 二叉树
tags:
  - 层序遍历
  - 二叉树
toc: true
toc_number: true
---

> 在二叉树相关算法中，**层序遍历（Level Order Traversal）** 是 BFS 的典型应用，在涉及「按层处理」「同一层节点关系」「层级统计」等问题时，几乎是第一选择。同时，通过层序遍历序列构造一颗树很好实现。
>
> 本文对我在 2026-01-05 完成的一批层序遍历题目进行一次系统总结，并结合做题过程中补充的 **Java 集合知识点**，形成一份可复盘的学习笔记。

### 基础题目

- 二叉树的层序遍历
- 二叉树的锯齿形层序遍历
- 找树左下角的值

### 扩展题目

- 二叉树的层序遍历 II
- 二叉树的最大深度
- 二叉树的最小深度
- 二叉树中的第 K 大层和
- 二叉树的右视图
-  填充每个节点的下一个右侧节点指针
- 层数最深叶子节点的和
- 奇偶树
- 反转二叉树的奇数层
- 二叉树的堂兄弟节点 II

------

## 层序遍历的核心识别信号

在读题时，只要出现以下关键词，就要高度警惕「层序遍历」：

- “一层一层”
- “同一层节点”
- “第 k 层 / 第 k 大层”
- “左视图 / 右视图”
- “最深一层 / 最浅一层”

**本质：题目要求你按层组织节点，而不是单纯的前序 / 中序 / 后序。**

------

## 层序遍历的两种经典实现方式

### 1. 双数组（当前层 / 下一层）写法（最直观）

思路非常清晰：

1. 用一个数组保存当前层节点
2. 遍历当前层时，把子节点加入“下一层数组”
3. 当前层处理完后，令 `cur = next`

```
cur = [root]
while cur 非空:
    处理 cur
    next = []
    for node in cur:
        next.add(node.left)
        next.add(node.right)
    cur = next
```

优点：

- 思路清楚，适合初学
- 非常适合“同层对称处理”的题目（如 2415）

------

### 队列写法（更通用、面试更常见）

**常用**的写法：

核心技巧：
**在每一轮循环开始时，记录当前队列长度 `size`，它就代表当前层的节点个数**

```
queue.offer(root)
while queue 非空:
    size = queue.size()
    for i in range(size):
        node = queue.poll()
        处理 node
        queue.offer(node.left)
        queue.offer(node.right)
```

优点：

- 不需要额外的“当前层数组”
- 非常适合统计、聚合、视图类问题

------

## 一道典型 Trick：2415. 反转二叉树的奇数层

这道题有一个非常重要但容易忽略的点：

> **题目要求反转的是“奇数层节点的值”，而不是节点本身。**

### 关键 insight

- 二叉树结构 **不能乱改**
- 但交换 `node.val` 是完全合法的
- 对称位置的节点只需要交换值即可

这是一个**“值操作”替代“结构操作”**的经典思路，后续很多树题都会用到。

------

## 做题过程中补充的 Java 集合知识

在实现层序遍历的过程中，我顺带系统补齐了一些 **Java 集合的易混点**。

### 1. `List.of(...)` vs `new ArrayList<>()`

- `List.of(...)`
  - 创建 **不可变 List**
  - 不能增删改
  - 不允许 `null`
  - 引用本身可以重新指向其他 List
- `new ArrayList<>()`
  - 可变集合
  - 支持增删改

常见用法：

```
List<Integer> a = new ArrayList<>(List.of(1, 2, 3));
```

------

### 2. `Collections.reverse(list)` vs `list.reversed()`

- `Collections.reverse(list)`
  - **就地反转**
  - 会修改原 list
  - list 必须是可变的
- `list.reversed()`（Java 21+）
  - 返回一个 **倒序视图（view）**
  - 不修改原 list
  - O(1) 创建视图
  - 原 list 改变，视图会联动变化

------

### 3. `Queue.offer / poll` vs `add / remove`

这是 Queue 接口里非常重要的一组 API 设计差异：

- `offer(e)`：失败返回 `false`
- `poll()`：空队列返回 `null`
- `add(e)`：失败抛异常
- `remove()`：空队列抛异常

实际写算法时，**`offer + poll` 更常用**，避免异常作为控制流。

------

## 总结

总体来看，**层序遍历的思想并不复杂**，但由于它需要我们**显式维护一个队列**，通过不断入队、出队来模拟递归过程，因此：

- 代码相对 DFS 更冗长
- 但在“同层处理”“层级统计”“视图类问题”中几乎不可替代

当题目强调“层”的概念时，应优先考虑层序遍历；而在实现过程中，**对 Java 集合 API 的理解是否扎实，往往直接决定代码是否简洁、健壮**。

## 构造本地测试用例：用层序数组构造二叉树

在刷 LeetCode/写本地 `main` 测试时，经常会遇到题目给出的输入形式是：

- `root = [5,8,9,2,1,3,7,4,6]`
- 或包含缺失节点：`root = [1,2,3,null,4]`

这种数组表示其实就是**层序遍历（level order）的序列化结果**：
 数组从左到右依次给出每一层的节点，`null` 表示该位置没有节点。

为了让本地调试更高效，我补齐了一个通用的建树方法：
**输入 `Integer[] levelOrder`，输出 `TreeNode root`（支持 null）**

### 核心思路

- 用队列保存“等待挂孩子的父节点”
- 指针 `i` 从 1 开始扫描数组
- 每次从队列弹出一个父节点，尝试读取两个位置作为它的 left/right
- 读到 `null` 就跳过，不创建节点

### Java 建树模板（支持 null）

```
public static TreeNode buildTree(Integer[] levelOrder) {
    if (levelOrder == null || levelOrder.length == 0 || levelOrder[0] == null) return null;
    
    TreeNode root = new TreeNode(levelOrder[0]);
    Queue<TreeNode> q = new ArrayDeque<>();
    q.offer(root);

    int i = 1;
    while (i < levelOrder.length && !q.isEmpty()) {
        TreeNode cur = q.poll();

        // left
        if (i < levelOrder.length && levelOrder[i] != null) {
            cur.left = new TreeNode(levelOrder[i]);
            q.offer(cur.left);
        }
        i++;

        // right
        if (i < levelOrder.length && levelOrder[i] != null) {
            cur.right = new TreeNode(levelOrder[i]);
            q.offer(cur.right);
        }
        i++;
    }
    return root;
}
```

## 相关代码

本文涉及的所有代码与笔记，均已同步至我的 GitHub 算法仓库，作为 Java 后端校招过程中的学习记录。