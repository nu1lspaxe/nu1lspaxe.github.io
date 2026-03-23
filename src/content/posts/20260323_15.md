---
title: 1594. Maximum Non Negative Product in a Matrix
published: 2026-03-23
description: "https://leetcode.com/problems/maximum-non-negative-product-in-a-matrix"
image: ""
tags: ["Leetcode"]
category: "Leetcode"
draft: false
lang: "zh_TW"
---

## Preface

目前是 2026， AI 氾濫的又一年，無論是網路或現實中都充斥著軟工焦慮的聲音。

筆者想在今日進入主題前先聊一下，為什麼在現在這個時機，還在寫 Leetcode 文章呢 ? 而且 Leetcode 解題在網路上已經很多了，AI 也會寫，搞不好還寫得更好 ?

這些說法不能說有錯，只是我們可以從不同角度看待問題。

**「輸出」** 是一項檢視你是否能夠將所學運用的一種方式，而且相當有用。

筆者算是比較晚才接受專業的學術教育，大學才進入資管系。在天賦方面不敢說有，只能在這片大海中持續努力，也就是說，其實筆者是在寫 Leetcode 一陣子之後，才慢慢領悟那種透過「演算法」解決問題，根據問題場景尋找最合適的解法的感覺。

而這種突然領悟的感覺，筆者想來分享幾個觀點，希望幫助如果有興趣的人，可以找到自己的方向。

1. 刷題
   這件事不建議視為在學校寫作業那樣，以達到作業目的為終點。
   更好的方式，是將題目內容視為今天公司產品的其中一個功能，或是想解決的一種問題。如此一來，更容易將整個題目視為日常開發的一環，無論是在工作上還是在解題中，都能遵循問題解決的模式，去思考。

2. 了解問題 = 了解 spec
   透過明確的問題限制，對比開發功能時需要考量到的功能限制，可以更了解你在解題中所使用到的資料結構或是演算法適合用在哪種情境。

稍微說明完雜談了，讓我們進入正題吧！

## Discussion

題目要求找到 maximum non-negative product (最大非負數乘積)。

問題很像 DP，因為要找到從 top-left 到 bottom-right 的 max product，過程中只能選擇 `right` or `down`。可以用 DP table `dp[i][j]` 紀錄每一個抵達的格子累積乘積是多少。

但要注意的是，輸入的 `grid` 中含有**負數**，也就是說我們後面可能會在某一格達到負負得正的相乘結果，比一路都選擇正整數的格子還大。

因此，我們可以選擇透過兩個 DP table，一個 `max_dp` 用於紀錄最大乘積、一個 `min_dp` 用於紀錄最小乘積。

遇到每個格子，新的 `dp[i][j]` 可能會是從左邊 (left) 或上面 (top) 乘過來 (因為路徑是從 `right` or `down` 擴散的)。因為有 2 個 DP table，所以可能的相乘值有 4 個 :

- `max_dp[i-1][j] * grid[i][j]`
- `max_dp[i][j-1] * grid[i][j]`
- `min_dp[i-1][j] * grid[i][j]`
- `min_dp[i][j-1] * grid[i][j]`

那再根據 4 個中的最大值與最小值更新 `max_dp[i][j]` 與 `min_dp[i][j]` 就好。

另外因為第一個 row 只能從左邊相乘過來，第一個 column 只可能從上面相乘下來，因此初始化單獨針對這兩條作處理。

## Solution

```python
class Solution:
    def maxProductPath(self, grid: List[List[int]]) -> int:
        m, n = len(grid), len(grid[0])

        # Initialize dp table
        # There has negative integer in grid matrix,
        # should use to table to keep both max positive and min negative integers
        max_dp = [[0] * n for _ in range(m)]
        min_dp = [[0] * n for _ in range(m)]

        max_dp[0][0] = min_dp[0][0] = grid[0][0]

        # Initialize first row and first column
        for i in range(1, m):
            max_dp[i][0] = min_dp[i][0] = max_dp[i-1][0] * grid[i][0]
        for j in range(1, n):
            max_dp[0][j] = min_dp[0][j] = max_dp[0][j-1] * grid[0][j]

        for i in range(1, m):
            for j in range(1, n):
                curr = grid[i][j]
                vals = [
                    max_dp[i-1][j] * curr,
                    max_dp[i][j-1] * curr,
                    min_dp[i-1][j] * curr,
                    min_dp[i][j-1] * curr,
                ]

                max_dp[i][j] = max(vals)
                min_dp[i][j] = min(vals)

        res = max_dp[-1][-1]
        return res % (10**9 + 7) if res >= 0 else -1
```

以上的時間複雜度是 _O(M\*N)_，空間複雜度是 _O(M\*N)_。

## Optimization

有些讀者可能發現，空間複雜度可以優化乘 _O(N)_。

讓我們觀察一下 transition function : `dp[i][j]` 只依賴上方格子 `dp[i-1][j]` 與左方格子 `dp[i][j-1]`。

代表當我們計算第 `i` 列時，完全不需要第 `i-2` 列以前的資料。

那如何從 2D 矩陣降維到 1D 呢 ? 我們可以只維護 **當前列** 就好。

- 上方格子 : 代表 `dp[j]`，當我們進入下一列時，`dp[0]` 會根據上一列相乘取得 `dp[0] * grid[i][0]`
- 左方格子 : 代表 `dp[j-1]`

程式碼會變成以下 :

```python
class Solution:
    def maxProductPath(self, grid: List[List[int]]) -> int:
        m, n = len(grid), len(grid[0])

        # Initialize dp table
        # There has negative integer in grid matrix,
        # should use to table to keep both max positive and min negative integers
        max_dp = [0 for _ in range(n)]
        min_dp = [0 for _ in range(n)]

        max_dp[0] = min_dp[0] = grid[0][0]

        # Initialize first row and first column
        for j in range(1, n):
            max_dp[j] = min_dp[j] = max_dp[j-1] * grid[0][j]

        for i in range(1, m):
            # for each row, first block must product from the
            # top block
            max_dp[0] = min_dp[0] = max_dp[0] * grid[i][0]

            for j in range(1, n):
                curr = grid[i][j]

                vals = [
                    max_dp[j] * curr,   # top
                    max_dp[j-1] * curr, # left
                    min_dp[j] * curr,   # top
                    min_dp[j-1] * curr, # left
                ]

                max_dp[j] = max(vals)
                min_dp[j] = min(vals)

        res = max_dp[-1]
        return res % (10**9 + 7) if res >= 0 else -1
```

恭喜你走到這裡，給自己一點鼓勵！
