---
title: 删除链表中的元素：从基础操作到 O(n) 优化思路（Java）
date: 2025-12-30 21:00:00
author: tingfeng-work
categories:
  - 算法
  - 链表
tags:
  - 删除链表节点
  - 时间复杂度优化
toc: true
toc_number: true
---

在链表相关题目中，“删除节点”是一个看似简单但非常容易出错的操作。
 本文结合我在刷题过程中的总结，梳理了**链表删除的通用思路**，并通过多个典型场景说明如何避免常见坑点，以及一些特殊处理

------

## 一、链表删除的本质

在**单链表**中，节点只知道自己的 `next`，**并不知道前驱节点**。

因此，删除一个节点的核心操作永远是：

> **找到要删除节点的前一个节点 `prev`，然后：**

```
prev.next = prev.next.next;
```

这也是为什么：

- **头节点删除**
- **倒数第 N 个节点**
- **条件删除多个节点**

几乎都会用到 **哨兵节点（dummy node）**。

------

## 二、为什么要使用哨兵节点（Dummy Node）

以删除头节点为例：

```
1 -> 2 -> 3
```

如果要删除 `1`，你会发现：

- 它没有前驱节点
- 需要单独写 if 判断

引入哨兵节点后：

```
dummy -> 1 -> 2 -> 3
```

这样就能**统一删除逻辑**：

```
ListNode dummy = new ListNode(0);
dummy.next = head;

ListNode cur = dummy;
while (cur.next != null) {
    if (cur.next.val == target) {
        cur.next = cur.next.next;
    } else {
        cur = cur.next;
    }
}
return dummy.next;
```

* 不需要特判头节点
* 删除逻辑始终一致

------

## 三、典型删除场景总结

### 1. 删除倒数第 N 个节点（双指针）

**核心思想：**
 让快指针先走 `N` 步，使快慢指针保持固定距离。

```
ListNode dummy = new ListNode(0);
dummy.next = head;

ListNode fast = dummy;
ListNode slow = dummy;

// fast 先走 N 步
for (int i = 0; i < n; i++) {
    fast = fast.next;
}

// 同时移动
while (fast.next != null) {
    fast = fast.next;
    slow = slow.next;
}

// slow 指向待删除节点的前一个
slow.next = slow.next.next;

return dummy.next;
```

**关键点：**

- 删除的是 `slow.next`
- dummy 能处理删除头节点的情况

------

### 2. 删除排序链表中的重复元素

在删除重复节点时，**一定要注意空指针问题**：

```
while (cur.next != null && cur.next.next != null) {
    if (cur.next.val == cur.next.next.val) {
        cur.next = cur.next.next;
    } else {
        cur = cur.next;
    }
}
```

在操作节点 `node.next`，`node.val` 时一定确保节点不为空

------

### 3. 删除“在数组中出现过”的节点（HashSet）

当删除条件来源于一个数组时，最优解是：

- 先把数组放进 `HashSet`(优化：预分配空间)
- 再遍历链表，O(1) 判断是否需要删除

```
Set<Integer> set = new HashSet<>(nums.length);
for (int x : nums) set.add(x);

ListNode dummy = new ListNode(0);
dummy.next = head;

ListNode cur = dummy;
while (cur.next != null) {
    if (set.contains(cur.next.val)) {
        cur.next = cur.next.next;
    } else {
        cur = cur.next;
    }
}
return dummy.next;
```

**时间复杂度：** O(n)
**空间复杂度：** O(n)

------

## 四、从 O(n²) 到 O(n)：正难则反的思想

在一道题中，我遇到了这样一个需求：

> 删除链表中 **右侧存在比当前节点值更大元素的节点**

###  直觉做法（超时）

- 对每个节点向右扫描
- 判断是否存在更大的值
   👉 **O(n²)**，直接超时

------

### 优化思路：正难则反

1. **反转链表**

2. 问题转化为：

   > 删除“左侧存在比当前节点更大的节点”

3. 删除比最大值小的节点

4. 再反转回来

**最终复杂度：**

- 时间：O(n)
- 空间：O(1)

------

## 五、总结

- 链表删除的核心永远是：**操作前驱节点**
- 需要操作头节点时，哨兵节点可以极大简化代码逻辑
- 双指针是处理“倒数第 N 个节点”的标准解法
- 当正向思路复杂时，要学会 **反转链表 + 条件转换**，运用之前写过的函数，拆解问题

------

## 六、相关代码

本文涉及的所有代码与笔记，均已同步至我的 GitHub 算法仓库，作为 Java 后端校招过程中的学习记录。