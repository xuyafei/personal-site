---
title: "C++堆结构详解：优先队列与heap算法完全指南"
date: 2025-08-05
draft: false
description: "深入解析C++中堆数据结构、优先队列、STL heap算法，包括make_heap、push_heap、pop_heap的使用方法和应用场景"
tags: ["C++", "堆结构", "优先队列", "STL", "算法"]
categories: ["C++"]
author: "徐亚飞"
showToc: true
TocOpen: true
weight: 18
math: true
---

# C++ 堆结构详解：优先队列与 heap 算法完全指南

## 一、什么是堆（Heap）？

堆是一种**完全二叉树**，具有以下特性：
- **大根堆（Max Heap）**：任意一个节点的值 ≥ 其左右孩子的值。
- **小根堆（Min Heap）**：任意一个节点的值 ≤ 其左右孩子的值。

这使得：
- 大根堆的**堆顶元素**是最大值。
- 小根堆的**堆顶元素**是最小值。

---

## 二、C++ 中如何表示堆？

在 C++ 中，一般使用 `std::vector` 或数组来表示堆结构。

假设数组下标从 0 开始，则有以下关系：
- **左孩子**：`2 * i + 1`
- **右孩子**：`2 * i + 2`
- **父节点**：`(i - 1) / 2`

**例子（最大堆）：**
```cpp
std::vector<int> heap = {10, 9, 8, 7, 6, 5, 4};
```

结构如下：
```
        10
       /  \
      9    8
     / \  / \
    7  6 5  4
```

---

## 三、C++ STL 中的堆操作函数

STL 提供了一组处理堆的算法，定义在 `<algorithm>` 中：

### 1. `std::make_heap(begin, end)`

将一段无序区间转换成堆结构（默认大根堆）。

```cpp
std::vector<int> v = {3, 1, 5, 2, 4};
std::make_heap(v.begin(), v.end());
```

### 2. `std::push_heap(begin, end)`

在堆尾部插入一个新元素之后，重新调整使其保持堆结构。

```cpp
v.push_back(6);
std::push_heap(v.begin(), v.end());
```

### 3. `std::pop_heap(begin, end)`

将堆顶元素与末尾元素交换，并重新调整剩下部分为堆结构。

```cpp
std::pop_heap(v.begin(), v.end());  // 堆顶到最后
v.pop_back();  // 移除最大元素
```

---

## 四、堆示例代码（最大堆）

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> heap = {3, 1, 4, 1, 5, 9};
    
    std::make_heap(heap.begin(), heap.end());
    std::cout << "堆顶元素（最大值）: " << heap.front() << std::endl;

    heap.push_back(6);
    std::push_heap(heap.begin(), heap.end());
    std::cout << "插入6后堆顶: " << heap.front() << std::endl;

    std::pop_heap(heap.begin(), heap.end());
    std::cout << "弹出堆顶后最大值: " << heap.back() << std::endl;
    heap.pop_back();

    std::sort_heap(heap.begin(), heap.end());
    std::cout << "堆排序后: ";
    for (int n : heap) std::cout << n << " ";
    std::cout << std::endl;

    return 0;
}
```

---

## 五、自定义小根堆

默认是大根堆，要创建小根堆，可以传入 `std::greater<>()` 比较函数：

```cpp
std::priority_queue<int, std::vector<int>, std::greater<int>> minHeap;
```

对于 `std::make_heap` 等函数：
```cpp
std::make_heap(v.begin(), v.end(), std::greater<int>());
```

---

## 六、应用场景

- **优先队列（priority_queue）**
- **Top-K 问题**（维护前 K 大或前 K 小）
- **堆排序**
- **Dijkstra 最短路径算法**
- **Huffman 编码树构建**

---

## 七、补充说明

### ✅ C++ 中的堆结构（vector + make_heap）是不是完全二叉树？

**答案是：是的！**

即使你看到的 `std::vector<int> v = {3, 1, 5, 2, 4}` 看起来不是一个树，但它在内存中的组织方式就是完全二叉树的数组表示形式。

#### 一、完全二叉树的定义

一个完全二叉树（Complete Binary Tree）是除了最底层外，每一层的节点都达到最大数，并且最底层的节点从左到右依次排列。

比如 5 个节点，它的完全二叉树结构是：
```
      3        （下标 0）
     / \
    1   5     （下标 1, 2）
   / \
  2   4       （下标 3, 4）
```

用数组存储这棵树是完全合理的：
```cpp
std::vector<int> v = {3, 1, 5, 2, 4};
```

只要元素是按照层序（从上到下、从左到右）排列的，这个 vector 天然就是完全二叉树的结构。堆（Heap）正是以这种数组方式来模拟树的。

#### 二、堆和完全二叉树的关系（数组建堆）

`std::make_heap` 会把这个 vector 的内容重新排列，满足最大堆（或最小堆）的结构，但不会破坏它作为完全二叉树的数组表示这一特性。

比如：
```cpp
std::vector<int> v = {3, 1, 5, 2, 4};
std::make_heap(v.begin(), v.end()); 
```

结果会变成（最大堆）：
```cpp
v = {5, 4, 3, 2, 1};  // 举例，make_heap 可能有多个合法结果
```

对应的完全二叉树结构：
```
       5
     /   \
    4     3
   / \
  2   1
```

**总结：** 你问的"能构成完全二叉树吗？"

是的，只要你用 vector 存储元素，下标从 0 开始，它总是表示一棵完全二叉树。
至于这些值是否满足堆的性质，那就得用 `make_heap` 来重排了，但结构上它一直是完全二叉树。

---

## 八、堆（Heap）与二叉搜索树（BST）的关键区别

| 特性 | 堆（Heap） | 二叉搜索树（BST） |
|------|------------|------------------|
| 树的结构 | 必须是完全二叉树 | 可以是不完整的二叉树 |
| 节点大小关系 | 父节点 ≥（或 ≤）任意子节点 | 左子树 < 根节点 < 右子树 |
| 节点排序 | 局部有序（只保证每个子树根最大/最小） | 全局有序（中序遍历为升序） |
| 查找效率 | 不能快速查找任意值 | 可以高效查找任意值（O(log n)） |
| 最大/最小值获取 | O(1) 获取根节点 | O(log n)（必须走到最左或最右） |
| 插入/删除 | O(log n)（调整堆结构） | O(log n)（平衡性好时） |

### 二、举个例子说明

我们以最大堆为例：
```cpp
std::vector<int> v = {3, 1, 5, 2, 4};
std::make_heap(v.begin(), v.end());
```

最终可能变成（最大堆）：
```
       5
     /   \
    4     3
   / \
  2   1
```

观察这个结构：
- 每个节点都 ≥ 它的孩子，这是堆的要求。
- 但并不满足左 < 根 < 右（BST 的要求）。
- 比如：左子树 4 < 5，但右子树是 3 < 5，没有任何关于左右子树的大小顺序要求。
- 所以它不是一个二叉排序树。

### 三、你可以记住这样一句话：

**"堆是为了快速找到最大/最小值；BST 是为了快速查找任意值。"**

---

## 九、`std::pop_heap(begin, end)` 只是重排元素，并不删除！

🔍 **它的行为是：**

将堆顶元素（最大或最小）移到序列的最后一个位置，并把前面的部分重新整理为一个有效的堆。

🚫 **它不会删除任何元素！**

你需要再手动调用 `v.pop_back()` 来真正删除。

### 二、总结

| 操作 | 是否修改容器大小 | 是否删除堆顶元素 | 是否保持堆结构 |
|------|------------------|------------------|----------------|
| `std::pop_heap(begin, end)` | ❌ 否 | ❌ 否 | ✅ 是（除了最后一个） |
| `v.pop_back()` | ✅ 是 | ✅ 是（配合上面使用） | ❌ 无关堆结构 |

### 使用套路

完整的"从堆中取出堆顶元素"的写法是：
```cpp
std::pop_heap(v.begin(), v.end()); // 堆顶交换到末尾
v.pop_back();                      // 删除末尾元素（原堆顶）
```

---

## 总结

C++ 中的堆结构通过 vector 和 STL 算法提供了高效的最大/最小值操作。理解堆与完全二叉树的关系，以及堆与二叉搜索树的区别，对于正确使用这些数据结构至关重要。掌握 `make_heap`、`push_heap`、`pop_heap` 等算法的使用方式，能够解决很多实际问题。 
