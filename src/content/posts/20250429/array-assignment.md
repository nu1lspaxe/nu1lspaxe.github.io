---
title: Array Assignment
published: 2025-04-29
description: "Benchmarks for array assignment in golang."
image: ""
tags: ["Go"]
category: "Go"
draft: false
lang: ""
---

## Background

This is a small piece of my personal journey.

During my first internship, a senior engineer from the team took some time to explain the role of an "Architect" to me.
Perhaps he noticed that I had a strong passion for digging into the principles of programming languages, as well as the patterns and architectures adopted by language designers.

Since then, I decided to embark on a path to become a "Master in Golang," and I'm excited to start sharing
some small insights about Go here.

The code referenced in this article comes from the following examples:

::github{repo="nu1lspaxe/tricky-go"}

## Test data Initialize

We use Go’s native Benchmark for testing:

```go=
const size = 10000

func BenchmarkAssignment(b *testing.B) {
	source := make([]int, size)
	for i := range source {
		source[i] = i
	}
    // ...
}
```

## Test Cases

1. Directly assign `source[i]` into a new slice

```go=
b.Run("direct assignment from index", func(b *testing.B) {
    b.ReportAllocs()
    for b.Loop() {
        target := make([]int, size)
        for j := range source {
            target[j] = source[j]
        }
    }

    b.ReportMetric(b.Elapsed().Seconds(), "s")
})
```

2. Similar to (1), but use a local variable `v` to store each value instead of indexing

```go=
b.Run("direct assignment from value", func(b *testing.B) {
    b.ReportAllocs()
    for b.Loop() {
        target := make([]int, size)
        for j, v := range source {
            target[j] = v
        }
    }

    b.ReportMetric(b.Elapsed().Seconds(), "s")
})
```

3. Use the built-in `copy` function

```go=
b.Run("copy", func(b *testing.B) {
    b.ReportAllocs()
    for b.Loop() {
        target := make([]int, size)
        copy(target, source)
    }

    b.ReportMetric(b.Elapsed().Seconds(), "s")
})
```

4. Initialize a slice with length 0 and a pre-allocated capacity equal to the copy size, then `append` all elements

```go=
b.Run("append slice", func(b *testing.B) {
    b.ReportAllocs()
    for b.Loop() {
        target := make([]int, 0, size)
        target = append(target, source...)
    }

    b.ReportMetric(b.Elapsed().Seconds(), "s")
})
```

5. Initialize a slice with length 0 and a pre-allocated capacity equal to the copy size, then `append` elements one by one

```go=
b.Run("append value with slice capacity", func(b *testing.B) {
    b.ReportAllocs()
    for b.Loop() {
        target := make([]int, 0, size)
        for _, v := range source {
            target = append(target, v)
        }
    }

    b.ReportMetric(b.Elapsed().Seconds(), "s")
})
```

6. Initialize an empty slice `[]int{}` and use `append` inside a `for` loop to add elements.

```go=
b.Run("append value without slice capacity", func(b *testing.B) {
    b.ReportAllocs()
    for b.Loop() {
        target := []int{}
        for _, v := range source {
            target = append(target, v)
        }
    }

    b.ReportMetric(b.Elapsed().Seconds(), "s")
})
```

## Benchmark Result

```text=
BenchmarkAssignment/direct_assignment_from_index-20          112618        13032 ns/op          1.468 s     81920 B/op       1 allocs/op
BenchmarkAssignment/direct_assignment_from_value-20           73483        15496 ns/op          1.139 s     81920 B/op       1 allocs/op
BenchmarkAssignment/copy-20                                  102176        11900 ns/op          1.216 s     81920 B/op       1 allocs/op
BenchmarkAssignment/append_slice-20                          110335        12220 ns/op          1.348 s     81920 B/op       1 allocs/op
BenchmarkAssignment/append_value_with_slice_capacity-20       73024        20067 ns/op          1.465 s     81920 B/op       1 allocs/op
BenchmarkAssignment/append_value_without_slice_capacity-20    14184        84584 ns/op          1.200 s    357627 B/op      19 allocs/op
```

The last approach results in the worst-case performance.
In Go, a slice is composed of a pointer, a length (`len`), and a capacity (`cap`). Since `target` is initialized as `[]int{}`, both its `len` and `cap` are initially 0. As elements are appended, exceeding the current `cap` triggers a reallocation—usually doubling the previous capacity. These frequent reallocations and memory copies can significantly degrade overall system performance.
