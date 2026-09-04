---
title: 2906. Construct Product Matrix
published: 2026-03-24
description: "https://leetcode.com/problems/construct-product-matrix"
image: ""
tags: ["Leetcode"]
category: "Leetcode"
draft: false
lang: "zh_TW"
---

## Discussion

### 模運算

在執行模運算時，要特別注意 :

- `(A / B) (mod M) != (A (mod M)) / (B (mod M))`

- `(A * B) (mod M) = [(A (mod M)) * B (mod M)] (mod M)`

### 模逆元 (Modular Multiplicative Inverse)

以這題需要 `mod 12345` 為例說明，在模 $M$ 的運算中，如果想要做到除以 $x$ 的操作，等於要找到一個數字 $y$，使得 $x \times y \equiv 1 \pmod M$。而 $y$ 就被稱作 $x$ 的模逆元。

**只有當 $x$ 與 $M$ 互質 (Greatest Common Divisor = 1) 時，模逆元 $y$ 才存在。**

而 $12345 = 3 \times 5 \times 823$，因此遇到這幾個數字都會破壞除法邏輯的有效性。

因此最好是使用**乘法**。

## Solution

在以下的解答中，會發現每一步都執行 modulo 操作，是為了防止整數溢位並維持運算效能。當數字乘積越來越大，運算會變得極慢且耗費大量記憶體。運用模運算特性，將數字限制在 0 ~ 12344 之間。

```python
class Solution:
    def constructProductMatrix(self, grid: List[List[int]]) -> List[List[int]]:
        m, n = len(grid), len(grid[0])
        MOD = 12345
        res = [[0] * n for _ in range(m)]

        # Forward (prefix product)
        prefix = 1
        for i in range(m):
            for j in range(n):
                res[i][j] = prefix
                prefix = (prefix * grid[i][j]) % MOD

        # Backward (suffix product)
        suffix = 1
        for i in range(m-1, -1, -1):
            for j in range(n-1, -1, -1):
                res[i][j] = (res[i][j] * suffix) % MOD
                suffix = (suffix * grid[i][j]) % MOD

        return res
```
