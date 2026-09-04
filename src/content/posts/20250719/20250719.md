---
title: Inside Go sync.Map
published: 2025-07-19
description: "Learn how sync.Map balances performance and safety in high-concurrency scenarios."
image: ""
tags: ["Go"]
category: "Go"
draft: false
lang: ""
---

## Background

最近有人問我，在需要保持順序狀態性的情況下，高併發的 goroutine 如何設計和運作。

而 `sync.Map` 剛好可以用來解決這個問題，因此有了這篇文章的誕生，來介紹 `sync.Map` 的設計理念。

## Memory Model

[The Go Memory Model](https://go.dev/ref/mem#model) 參考自 [_Foundations of the C++ Concurrency Memory Model_](https://dl.acm.org/doi/10.1145/1375581.1375591)，提到 **data-race-free** 的程式會有 **sequential consistency** (順序一致性) 的執行結果，稱為 DRF-SC。

### Memory Operation

每個 memory operation 包含以下 4 個要素 :

- kind : ordinary write, ordinary read, or synchronizing (e.g, atomic, mutex, channel)
- program location
- memory location
- 被執行的值 (value)

在分類上，

- read-like : e.g., read, atomic read, mutex lock and channel receive
- write-like : e.g., write, atomic write, mutex unlock, channel send and channel close
- both : e.g., atomic compare-and-swap

### Goroutine Execution

1. 每個 goroutine 的 memory operations 必須符合[正確的順序邏輯](https://go.dev/ref/spec#Order_of_evaluation)
2. 在同步操作中，所有讀取的值必須合理
   - _synchronized before_ 關係是一個同步記憶體操作的 partial order，稱作 `W`
   - `W` 對每個 synchronizing read-like (同步讀取) 記憶體操作 `r` 建立對應關係到 synchronizing write-like (同步寫入) 記憶體操作`w`
   - 寫作 `W(r) = w`
3. 當執行普通 (ordinary) 讀取操作 `r` 時，讀取的值 `W(r)` 必須滿足兩個條件 :
   - `w happens-before r`
   - `w` 必須是所有會發生在 `r` 之前的寫入中**最後發生**的那個; 如果有其他 `w'` 發生在 `r` 之前，`w'` 一樣要發生在 `w` 之前

## `sync.Map`

```go
// src/sync/map.go

// Map is like a Go map[any]any but is safe for concurrent use
// by multiple goroutines without additional locking or coordination.
// Loads, stores, and deletes run in amortized constant time.
//
// The Map type is specialized. Most code should use a plain Go map instead,
// with separate locking or coordination, for better type safety and to make it
// easier to maintain other invariants along with the map content.
//
// The Map type is optimized for two common use cases: (1) when the entry for a given
// // key is only ever written once but read many times, as in caches that only grow,
// or (2) when multiple goroutines read, write, and overwrite entries for disjoint
// sets of keys. In these two cases, use of a Map may significantly reduce lock
// contention compared to a Go map paired with a separate [Mutex] or [RWMutex].
//
// The zero Map is empty and ready for use. A Map must not be copied after first use.
//
// In the terminology of [the Go memory model], Map arranges that a write operation
// “synchronizes before” any read operation that observes the effect of the write
type Map struct {
    _ noCopy

    mu Mutex

    read atomic.Pointer[readOnly]

    dirty map[any]*entry

    misses int
}
```

首先看到匿名欄位 `noCopy`，其目的是用來防止使用者進行 (值) 複製。因為 `sync.Map` 含有同步狀態結構 `sync.Mutex`、`atomic.Value` 以及非同步安全的內部結構。

:::note

`noCopy` source code :

```go
// src/sync/cond.go

// noCopy may be added to structs which must not be copied after the first use.
//
// Note that it must not be embedded, due to the Lock and Unlock methods.
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock() {}
func (*noCopy) Unlock() {}
```

`noCopy` 的 `Lock` 跟 `Unlock` 並不是真的提供 lock ，而是給 `go vet` 靜態分析工具做檢測。

:::

再來看到 `read` 欄位，以下是針對該欄位的註解 :

```go
// read contains the portion of the map's contents that are safe for
// concurrent access (with or without mu held).
//
// The read field itself is always safe to load, but must only be stored with mu held.
//
// Entries stored in read may be updated concurrently without mu, but updating
// a previously-expunged entry requires that the entry be copied to the dirty
// map and unexpunged with mu held.
```

可以整理為以下 :

- `sync.Map` 為了效能，把資料分為兩個部分 : `read` 跟 `dirty`
- `read` 是唯讀快取區域，無論是否持有 `mu` 鎖都可以被 goroutine 併發訪問
- 儘管讀取是安全的，但寫入時，必須持有 `mu` 鎖
- 如果某筆資料已經被標記為 expunged (已刪除)，就必須持有 `mu` 鎖，將它複製到 `dirty` map 並取消 expunge 才能更新

然後是 `dirty` 欄位，以下是針對該欄位的註解 :

```go
// dirty contains the portion of the map's contents that require mu to be
// held. To ensure that the dirty map can be promoted to the read map quicly,
// it also includes all of the non-expunged entries in the read map.
//
// Expunged entries are not stored in the dirty map. An expunged entry in the
// clean map must be unexpunged and added to the dirty map before a new value
// can be stored to it.
//
// If the dirty map is nil, the next write to the map will initialize it by
// making a shallow copy of the clean map, omitting stale entries.
```

大致上為下列觀念 :

- `dirty` map 的任何操作都必須持有 `mu` 才能進行
- 為了讓 `dirty` map 可以快速升級為 `read` map (稱為 promotion)，`dirty` map 也會包含 `read` 裡面尚未被標記為 expunged 的資料 (也就是說，被 expunged 的資料不會出現在 `dirty` 中)
- `dirty` 是 lazy-init，在第一次寫入動作才初始化，淺拷貝 `read` map 的資料

最後是針對 `missess` 欄位，同樣，我們來看它的註解 :

```go
// misses counts the number of loads since the read map was last updated that
// needed to lock mu to determine whether the key was present.
//
// Once enough misses have occurred to cover the cost of copying the dirty
// map, the dirty map will be promoted to the read map (in the unamended
// state) and the next store to the map will make a new dirty copy.
```

把這段歸納成以下重點 :

- `misses` 紀錄上次更新 `read` map 之後，有多少次讀取找不到 key，而需要配合 `mu` 查看 `dirty` map 才能知道該 key 是否存在
- 當 `misses` 次數達到與複製 `dirty` map 相同的成本時，`dirty` map 就會被 promote 成 `read` map，並清空為 `unamended` 狀態。於下次寫入時，會重新創建新的 `dirty` map

以下是其他在 `sync.Map` 中的欄位使用到的結構與變數，可以更好地了解前面提到的概念 :

```go
// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
    m        map[any]*entry
    ammended bool // true if the dirty map contains some key not in m.
}

// expunged is an arbitrary pointer that marks entries which ahve been deleted
// from the dirty map.
var expunged = new(any)

// An entry is a slot in the map corresponding to a particular key.
type entry struct {
    // p points to the interface{} value stored for the entry.
    //
    // If p == nil, the entry has been deleted, and either m.dirty == nil or
    // m.dirty[key] is e.
    //
    // If p == expunged, the entry has been deleted, m.dirty != nil, and the entry
    // is missing from m.dirty.
    //
    // Otherwise, the entry is valid and recorded in m.read.m[key] and, if m.dirty != nil,
    // in m.dirty[key].
    //
    // An entry can be deleted by atomic replacement with nil: when m.dirty is
    // next created, it will atomically replace nil with expunged and leave
    // m.dirty[key] unset.
    //
    // An entry's associated value can be updated by atomic replacement, provided
    // p != expunged. If p == expunged, an entry's associated value can be updated
    // only after first setting m.dirty[key] = e so taht lookups using the dirty
    // map find the entry.
    p atomic.Pointer[any]
}
```
