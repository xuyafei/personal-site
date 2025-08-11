---
title: "C++ 优先队列（std::priority_queue）完全指南：从基础到高级应用"
description: "深入解析C++中优先队列的使用方法，从基础语法到高级应用，包含实际项目中的最佳实践和性能优化技巧"
keywords: ["C++", "优先队列", "priority_queue", "堆", "heap", "STL", "算法", "数据结构"]
author: "C++开发者指南"
date: "2025-08-07"
tags: ["C++", "数据结构", "算法", "STL", "优先队列", "堆"]
categories: ["C++"]
toc: true
---

# C++ 优先队列（std::priority_queue）完全指南：从基础到高级应用

## 概述

C++ 中的优先队列（std::priority_queue）是一个非常常用的容器适配器，它封装了堆（heap）结构，默认是最大堆，也可以配置成最小堆或者自定义排序。

## 1. 什么是优先队列？

优先队列是一种特殊的队列结构，队首元素总是优先级最高的元素。

不像普通队列是先进先出（FIFO），优先队列会根据你设置的优先级来出队。C++ 中的 std::priority_queue 默认使用最大堆（大顶堆）。

### 1.1 优先队列的特点

- **自动排序**：元素按照优先级自动排序
- **快速访问**：O(1) 时间复杂度访问最高优先级元素
- **高效插入删除**：O(log n) 时间复杂度进行插入和删除操作
- **内存效率**：基于数组实现，内存使用紧凑

### 1.2 堆结构基础

优先队列底层使用堆（Heap）数据结构：
- **最大堆**：父节点总是大于等于子节点
- **最小堆**：父节点总是小于等于子节点
- **完全二叉树**：除了最后一层，其他层都是满的

## 2. 基本使用方法

### 2.1 头文件
```cpp
#include <queue>
```

### 2.2 默认用法（最大堆）
```cpp
#include <iostream>
#include <queue>
using namespace std;

int main() {
    priority_queue<int> pq;
    pq.push(5);
    pq.push(1);
    pq.push(10);

    while (!pq.empty()) {
        cout << pq.top() << " "; // 每次输出最大元素
        pq.pop();
    }
    // 输出：10 5 1
}
```

### 2.3 最小堆的写法

默认是最大堆。想变成最小堆，可以用**greater<T>**：
```cpp
#include <queue>
#include <vector>
#include <functional> // for greater
using namespace std;

priority_queue<int, vector<int>, greater<int>> minHeap;
```

示例：
```cpp
priority_queue<int, vector<int>, greater<int>> pq;
pq.push(5);
pq.push(1);
pq.push(10);

while (!pq.empty()) {
    cout << pq.top() << " "; // 每次输出最小元素
    pq.pop();
}
// 输出：1 5 10
```

## 3. 自定义类型的优先队列

当你放入的是结构体（比如任务、节点等），你需要自定义排序规则。

### 3.1 方法一：重载 < 操作符（适用于最大堆）
```cpp
struct Node {
    int val;
    int priority;

    // 注意：默认 priority_queue 是最大堆
    bool operator<(const Node& other) const {
        return priority < other.priority; // priority大的在前
    }
};

priority_queue<Node> pq;
```

### 3.2 方法二：使用比较器（更灵活）
```cpp
struct Node {
    int val;
    int priority;
};

struct Compare {
    bool operator()(const Node& a, const Node& b) {
        return a.priority > b.priority; // 小的优先（最小堆）
    }
};

priority_queue<Node, vector<Node>, Compare> pq;
```

## 4. 常用操作总结

| 操作 | 含义 | 时间复杂度 |
|------|------|------------|
| `pq.push(x)` | 插入元素 | O(log n) |
| `pq.top()` | 查看当前优先级最高的元素（不删除） | O(1) |
| `pq.pop()` | 删除当前优先级最高的元素 | O(log n) |
| `pq.empty()` | 判断队列是否为空 | O(1) |
| `pq.size()` | 返回元素个数 | O(1) |

## 5. 高级用法

### 5.1 pair 类型的优先队列（常用于图/最短路径）

默认是按照 first 排序，再按 second
```cpp
#include <iostream>
#include <queue>
using namespace std;

int main() {
    priority_queue<pair<int, string>> pq;

    pq.push({3, "low"});
    pq.push({5, "medium"});
    pq.push({10, "high"});

    while (!pq.empty()) {
        auto top = pq.top();
        cout << "Priority: " << top.first << ", Value: " << top.second << endl;
        pq.pop();
    }
    // 输出顺序：10 -> 5 -> 3（按 first 降序）
}
```

如果要按 first 升序（即最小堆）
```cpp
priority_queue<pair<int, string>, vector<pair<int, string>>, greater<pair<int, string>>> pq;
```

### 5.2 使用 lambda 表达式定义优先级（最灵活）
```cpp
#include <iostream>
#include <queue>
#include <vector>
using namespace std;

int main() {
    // lambda 比较器：小的优先（最小堆）
    auto cmp = [](int a, int b) {
        return a > b; // a 优先级高就放后面
    };

    priority_queue<int, vector<int>, decltype(cmp)> pq(cmp);

    pq.push(10);
    pq.push(2);
    pq.push(7);

    while (!pq.empty()) {
        cout << pq.top() << " "; // 输出：2 7 10
        pq.pop();
    }
}
```

例子：对 pair<int, string> 进行自定义排序（按字符串升序）
```cpp
#include <iostream>
#include <queue>
#include <string>
using namespace std;

int main() {
    auto cmp = [](const pair<int, string>& a, const pair<int, string>& b) {
        return a.second > b.second; // 字符串字典序升序
    };

    priority_queue<pair<int, string>, vector<pair<int, string>>, decltype(cmp)> pq(cmp);

    pq.push({1, "banana"});
    pq.push({2, "apple"});
    pq.push({3, "cherry"});

    while (!pq.empty()) {
        cout << pq.top().second << " ";
        pq.pop();
    }
    // 输出：apple banana cherry
}
```

### 5.3 实战例子：找出前 K 大的元素（Top-K）
```cpp
#include <iostream>
#include <vector>
#include <queue>
using namespace std;

vector<int> topK(vector<int>& nums, int k) {
    // 最小堆，堆顶是当前最小的那个
    priority_queue<int, vector<int>, greater<int>> pq;

    for (int num : nums) {
        pq.push(num);
        if (pq.size() > k)
            pq.pop(); // 保持堆的大小为 k
    }

    vector<int> res;
    while (!pq.empty()) {
        res.push_back(pq.top());
        pq.pop();
    }
    return res;
}

int main() {
    vector<int> nums = {4, 1, 7, 3, 8, 5};
    int k = 3;

    vector<int> res = topK(nums, k);
    for (int x : res)
        cout << x << " ";
    // 输出：5 7 8（顺序可能略有不同）
}
```

## 6. 实际应用场景

### 6.1 任务调度系统
```cpp
struct Task {
    int id;
    int priority;
    string description;
    
    bool operator<(const Task& other) const {
        return priority < other.priority; // 高优先级先执行
    }
};

class TaskScheduler {
private:
    priority_queue<Task> taskQueue;
    
public:
    void addTask(int id, int priority, const string& desc) {
        taskQueue.push({id, priority, desc});
    }
    
    Task getNextTask() {
        if (taskQueue.empty()) {
            throw runtime_error("No tasks available");
        }
        Task task = taskQueue.top();
        taskQueue.pop();
        return task;
    }
    
    bool hasTasks() const {
        return !taskQueue.empty();
    }
};
```

### 6.2 数据流中位数
```cpp
class MedianFinder {
private:
    priority_queue<int> maxHeap; // 存储较小的一半
    priority_queue<int, vector<int>, greater<int>> minHeap; // 存储较大的一半
    
public:
    void addNum(int num) {
        maxHeap.push(num);
        minHeap.push(maxHeap.top());
        maxHeap.pop();
        
        if (maxHeap.size() < minHeap.size()) {
            maxHeap.push(minHeap.top());
            minHeap.pop();
        }
    }
    
    double findMedian() {
        if (maxHeap.size() > minHeap.size()) {
            return maxHeap.top();
        }
        return (maxHeap.top() + minHeap.top()) / 2.0;
    }
};
```

### 6.3 合并K个有序链表
```cpp
struct ListNode {
    int val;
    ListNode *next;
    ListNode(int x) : val(x), next(NULL) {}
};

class Solution {
public:
    ListNode* mergeKLists(vector<ListNode*>& lists) {
        auto cmp = [](ListNode* a, ListNode* b) {
            return a->val > b->val;
        };
        
        priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);
        
        // 将所有链表的头节点加入优先队列
        for (ListNode* list : lists) {
            if (list) pq.push(list);
        }
        
        ListNode dummy(0);
        ListNode* current = &dummy;
        
        while (!pq.empty()) {
            ListNode* node = pq.top();
            pq.pop();
            
            current->next = node;
            current = current->next;
            
            if (node->next) {
                pq.push(node->next);
            }
        }
        
        return dummy.next;
    }
};
```

## 7. 性能优化技巧

### 7.1 预分配内存
```cpp
// 如果知道大概的元素数量，可以预分配
priority_queue<int> pq;
pq.reserve(1000); // 预分配1000个元素的空间
```

### 7.2 使用emplace避免拷贝
```cpp
struct ComplexObject {
    int id;
    string name;
    vector<int> data;
    
    ComplexObject(int i, const string& n, const vector<int>& d) 
        : id(i), name(n), data(d) {}
};

// 使用emplace直接构造，避免拷贝
priority_queue<ComplexObject> pq;
pq.emplace(1, "obj1", vector<int>{1, 2, 3});
```

### 7.3 自定义分配器
```cpp
// 使用自定义分配器优化内存使用
priority_queue<int, vector<int, custom_allocator<int>>> pq;
```

## 8. 常见陷阱和注意事项

### 8.1 比较器设计错误
```cpp
// 错误：比较器逻辑反了
struct WrongCompare {
    bool operator()(int a, int b) {
        return a < b; // 这样会得到最大堆，不是最小堆
    }
};

// 正确：最小堆比较器
struct CorrectCompare {
    bool operator()(int a, int b) {
        return a > b; // 这样才是最小堆
    }
};
```

### 8.2 空队列访问
```cpp
priority_queue<int> pq;
// 错误：在空队列上调用top()或pop()
// int val = pq.top(); // 未定义行为

// 正确：先检查是否为空
if (!pq.empty()) {
    int val = pq.top();
    pq.pop();
}
```

### 8.3 迭代器失效
```cpp
// priority_queue 不支持迭代器，不能像其他容器那样遍历
// 错误：priority_queue<int>::iterator it = pq.begin(); // 编译错误

// 正确：只能通过pop()来访问元素
while (!pq.empty()) {
    cout << pq.top() << " ";
    pq.pop();
}
```

## 9. 与其他容器的比较

| 特性 | priority_queue | set/multiset | vector + sort |
|------|----------------|--------------|----------------|
| 插入复杂度 | O(log n) | O(log n) | O(n) |
| 删除复杂度 | O(log n) | O(log n) | O(n) |
| 查找最大/最小 | O(1) | O(1) | O(1) |
| 内存使用 | 紧凑 | 较高 | 紧凑 |
| 支持重复元素 | 是 | multiset是 | 是 |
| 支持迭代器 | 否 | 是 | 是 |

## 10. 总结

| 方式 | 用法场景 | 示例关键点 |
|------|----------|------------|
| 默认 `priority_queue<int>` | 最大堆 | 输出最大值优先 |
| `greater<T>` | 最小堆 | `priority_queue<int, vector<int>, greater<>>` |
| `pair` | 图算法，带权重元素 | 默认按 first 降序排序 |
| 自定义 struct + 比较器 | 多维权重，复杂逻辑排序 | `struct Compare { bool operator()... }` |
| lambda 表达式 | 临时、简洁地定义排序 | `auto cmp = [](T a, T b) { ... };` |

### 10.1 选择建议

- **简单数值排序**：使用默认或 `greater<T>`
- **复杂对象排序**：使用自定义比较器
- **临时排序需求**：使用 lambda 表达式
- **图算法**：使用 `pair` 类型
- **需要迭代器**：考虑使用 `set` 或 `multiset`

### 10.2 最佳实践

1. **明确比较逻辑**：确保比较器逻辑正确
2. **检查空队列**：在访问元素前检查是否为空
3. **合理使用**：优先队列适合需要频繁访问最大/最小元素的场景
4. **内存考虑**：对于大数据集，考虑内存使用情况
5. **性能测试**：在关键路径上进行性能测试

通过掌握这些技巧，你就能在各种场景中灵活运用C++的优先队列，解决复杂的算法问题。 
