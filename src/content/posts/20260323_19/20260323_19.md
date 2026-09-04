---
title: "System Design: LFU Cache"
published: 2026-03-23
description: "https://leetcode.com/problems/lfu-cache"
image: ""
tags: ["Leetcode"]
category: "Leetcode"
draft: false
lang: "en"
---

## LRU Cache vs LFU Cache

In [my previous post](./20260322.md), we discussed LRU (Least Recently Used), which evicts the "oldest" data. However, LRU has a weakness, **Cache Pollution**. If a large one-time scan occurs, it kicks out "hot" data for data that will never be used again.

LFU (Least Frequently Used) solves this by counting how many times a key is accessed. It prioritizes data with higher frequency.

## Discussion

To keep time complexity in _O(1)_ just like LRU cache, we take advantage of **Doubly Linked List**.

We need to combine three structures :

- **Frequency Map** : `frequency` -> `Doubly Linked List`

- **Hash Map** : `key` -> `Node` for _O(1)_ lookup

- **`min_freq` Pointer** : an integer tracking the current lowest frequency in the cache to find eviction instantly.

```python
from collections import defaultdict

class Node:
    def __init__(self, key, val):
        self.key, self.val = key, val
        self.freq = 1
        self.prev = self.next = None

class DoublyLinkedList:
    """Standard DLL with O(1) add/remove operations."""
    def __init__(self):
        self.head, self.tail = Node(0, 0), Node(0, 0)
        self.head.next, self.tail.prev = self.tail, self.head
        self.size = 0

    def add_to_head(self, node):
        node.next, node.prev = self.head.next, self.head
        self.head.next.prev = node
        self.head.next = node
        self.size += 1

    def remove(self, node):
        node.prev.next, node.next.prev = node.next, node.prev
        self.size -= 1
        return node

class LFUCache:
    def __init__(self, capacity: int):
        self.cap = capacity
        self.cache = {} # key -> Node
        # a defaultdict to automatically create DoublyLinkedList structure
        self.freq_map = defaultdict(DoublyLinkedList) # freq -> DLL
        self.min_freq = 0

    def _update(self, node):
        """
        A difference from LRU cache, updating a node should remove from old frequency doubly linked list and move to new frequency doubly linked list.
        """
        f = node.freq
        self.freq_map[f].remove(node)

        # If the floor is empty and it was the min_freq floor, update min_freq
        if f == self.min_freq and self.freq_map[f].size == 0:
            self.min_freq += 1

        node.freq += 1
        self.freq_map[node.freq].add_to_head(node)

    def get(self, key: int) -> int:
        if key not in self.cache: return -1
        node = self.cache[key]
        self._update(node)
        return node.val

    def put(self, key: int, value: int) -> None:
        if self.cap == 0: return

        if key in self.cache:
            node = self.cache[key]
            node.val = value
            self._update(node)
        else:
            if len(self.cache) >= self.cap:
                # Evict the LRU node from the min_freq floor
                lru_node = self.freq_map[self.min_freq].tail.prev
                self.freq_map[self.min_freq].remove(lru_node)
                del self.cache[lru_node.key]

            new_node = Node(key, value)
            self.cache[key] = new_node
            self.freq_map[1].add_to_head(new_node)
            # A newly inserted node always starts with frequency 1.
            self.min_freq = 1
```
