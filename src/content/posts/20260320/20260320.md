---
title: 380. Insert Delete GetRandom O(1)
published: 2026-03-20
description: "https://leetcode.com/problems/insert-delete-getrandom-o1"
image: ""
tags: ["Leetcode"]
category: "Leetcode"
draft: false
lang: "zh_TW"
---

## Discussion

題目需要我們實作 function :

- `bool insert(int val)`
- `bool remove(int val)`
- `int getRandom()`

其實題目在提到 `RandomizedSet` 就有給了一點提示，或許能用 `set` 解決。那我們就先來看看 `set` 的特性。

在 Python 中， `set` 的底層 CPython 是透過 hashtable 實作的。因此在執行 `find` / `search` 操作可以在複雜度 _O(1)_ 內解決。在執行 `pop()` 與 `add` 操作也都是 _O(1)_。

這邊是 `set` 的原始碼結構位於 [`cpython/Include/cpython/setobject.h`](https://github.com/python/cpython/blob/main/Include/cpython/setobject.h) :

```cpp
/* There are three kinds of entries in the table:

1. Unused:  key == NULL and hash == 0
2. Dummy:   key == dummy and hash == -1
3. Active:  key != NULL and key != dummy and hash != -1

The hash field of Unused slots is always zero.

The hash field of Dummy slots are set to -1
meaning that dummy entries can be detected by
either entry->key==dummy or by entry->hash==-1.
*/

#define PySet_MINSIZE 8

typedef struct {
    PyObject *key;
    Py_hash_t hash;             /* Cached hash code of the key */
} setentry;

typedef struct {
    PyObject_HEAD

    Py_ssize_t fill;            /* Number active and dummy entries*/
    Py_ssize_t used;            /* Number active entries */

    /* The table contains mask + 1 slots, and that's a power of 2.
     * We store the mask instead of the size because the mask is more
     * frequently needed.
     */
    Py_ssize_t mask;

    /* The table points to a fixed-size smalltable for small tables
     * or to additional malloc'ed memory for bigger tables.
     * The table pointer is never NULL which saves us from repeated
     * runtime null-tests.
     */
    setentry *table;
    Py_hash_t hash;             /* Only used by frozenset objects */
    Py_ssize_t finger;          /* Search finger for pop() */

    setentry smalltable[PySet_MINSIZE];
    PyObject *weakreflist;      /* List of weak references */
} PySetObject;
```

不會 C/C++ 的讀者沒關係，筆者也不熟，但我們可以從上面看到這段 :

```cpp
#define PySet_MINSIZE 8
```

其實 Python 在實作 `set` 時，有給它預設的大小。根據上面的註解，可能會找到空值的情況就，就代表其實 `set` 實際包含的大小會比我們指派給它的還要多。

讓我們再回頭思考 `getRandom()`，會發現如果透過 `table[rand() % size]` 這種方式就無法在 _O(1)_ 的條件下取回數值 (因為取得的位置可能是空的)。

那可能是 `set` 這個選擇不對嗎 ? 那我們換個資料結構，從 `List` 的角度思考。

`List` 在進行隨機取值 (e.g., `list[i]`) 時的複雜度是 _O(1)_，那我們就可以透過以下方式來執行 `getRandom()` :

```python
import random

return random.choice(self.list)
```

現在處理完 `getRandom()`，讓我們來看到 `remove(int val)`。

`List` 的 `append()` 與 `pop()` 都是 _O(1)_，但如果要刪除某個特定的值 (`remove()` 操作)，複雜度會變成 _O(N)_。

如果我們仔細拆解 `remove()`，應該可以發現它是先透過線性搜尋來做移除的。原始碼位於 [`cpython/Objects/listobject.c`](https://github.com/python/cpython/blob/main/Objects/listobject.c) :

```cpp
static PyObject *
list_remove_impl(PyListObject *self, PyObject *value)
/*[clinic end generated code: output=b9b76a6633b18778 input=26c813dbb95aa93b]*/
{
    Py_ssize_t i;

    for (i = 0; i < Py_SIZE(self); i++) {
        PyObject *obj = self->ob_item[i];
        Py_INCREF(obj);
        int cmp = PyObject_RichCompareBool(obj, value, Py_EQ);
        Py_DECREF(obj);
        if (cmp > 0) {
            if (list_ass_slice_lock_held(self, i, i+1, NULL) == 0)
                Py_RETURN_NONE;
            return NULL;
        }
        else if (cmp < 0)
            return NULL;
    }
    PyErr_SetString(PyExc_ValueError, "list.remove(x): x not in list");
    return NULL;
}
```

既然我們已經知道了執行刪除所需要的步驟，讓我們來重新設計一下 `RandomizedSet`。在每次執行 `insert()` 時，使用 `Dict` 紀錄下該數值對應的 index，就能以 _O(1)_ 來省略線性搜索這個步驟。

但是呢，即便我們已經知道了刪除值的 index，`List` 還是需要 _O(N)_ 來執行 `del[i]`。

讓我們聯想到前面提過的 `pop()` 操作只需要 _O(1)_，而隨機取值 `list[i]` 也只需要 _O(1)_。

有一種豁然開朗的感覺嗎? 既然已知要刪除元素的 index，是不是就可以透過 `swap` 來與 `List` 最後一個元素進行交換，然後再透過 `pop()` 把該元素刪除。這樣就達到 _O(1)_ 了！

既然我們已經找到了方法，就來看一下解答吧！

## Solution

```python
import random

class RandomizedSet:

    def __init__(self):
        self.lis = []
        self.dic = {}

    def insert(self, val: int) -> bool:
        if val in self.dic:
            return False

        self.lis.append(val)
        self.dic[val] = len(self.lis) - 1
        return True

    def remove(self, val: int) -> bool:
        if val not in self.dic:
            return False

        del_idx = self.dic[val]
        last_idx = len(self.lis) - 1
        last_val = self.lis[-1]

        # swap & pop
        self.lis[del_idx], self.lis[last_idx] = last_val, val
        self.lis.pop()

        # update indices
        # Note: update last_val index before deletion
        self.dic[last_val] = del_idx
        del self.dic[val]
        return True

    def getRandom(self) -> int:
        return random.choice(self.lis)
```

本文結束，恭喜你們走到這一步，給自己一點鼓勵！
