+++
title = "link list sorting"
date = 2023-10-24

[taxonomies]
tags = ["golang", "sorting", "link list"]
+++


偶然读到了Putty作者[Simon Tatham的这篇blog](https://www.chiark.greenend.org.uk/~sgtatham/algorithms/listsort.html), 他这个自底向上，
只需要常量空间的链表merge sort 比较有意思, 跟我之前了解的自顶向下和自底向上的两个版本不太一样, 于是把他这个C的实现移植到golang，
写了简单test和benchmark, 跟之前了解的版本做下对比

正好golang 1.21 发布后，在标准库引入了[slice](https://pkg.go.dev/slices), [cmp](https://pkg.go.dev/cmp)等便利工具可以使用

sortlist.go:
```golang

package main

import (
	"cmp"
	"fmt"
	"math/rand"
	"slices"
)

type node[E cmp.Ordered] struct {
	next *node[E]
	data E
}

func prependList[E cmp.Ordered](href **node[E], data E) {
	new := &node[E]{data: data}
	new.next = *href
	*href = new
}

func appendList[E cmp.Ordered](href **node[E], data E) {
	new := &node[E]{data: data}
	tail := href
	for ; *tail != nil; tail = &(*tail).next {
	}
	*tail = new
}

func listIsSorted[E cmp.Ordered](h *node[E]) bool {
	sorted := true
	for c := h; c != nil; c = c.next {
		if c.next != nil && cmp.Compare(c.data, c.next.data) == 1 {
			sorted = false
			break
		}
	}
	return sorted
}

func walk[E cmp.Ordered](h *node[E]) {
	for c := h; c != nil; c = c.next {
		fmt.Printf("%v ", c.data)
	}
	fmt.Println()
}

func inPlaceMergeSort[E cmp.Ordered](list *node[E]) *node[E] {
	var (
		p     *node[E]
		q     *node[E]
		e     *node[E]
		tail  *node[E]
		psize int
		qsize int
	)
	if list == nil {
		return nil
	}
	insize := 1
	for {
		p = list
		list = nil
		tail = nil
		nmerges := 0
		for p != nil {
			nmerges++
			q = p
			psize = 0
			for i := 0; i < insize; i++ {
				psize++
				q = q.next
				if q == nil {
					break
				}
			}
			qsize = insize
			for psize > 0 || (qsize > 0 && q != nil) {
				if psize == 0 {
					e = q
					q = q.next
					qsize--
				} else if qsize == 0 || q == nil {
					e = p
					p = p.next
					psize--
				} else if cmp.Compare(p.data, q.data) <= 0 {
					e = p
					p = p.next
					psize--
				} else {
					e = q
					q = q.next
					qsize--
				}
				if tail != nil {
					tail.next = e
				} else {
					list = e
				}
				tail = e
			}
			p = q
		}
		tail.next = nil
		if nmerges <= 1 {
			return list
		}
		insize *= 2
	}
}

func insertSort[E cmp.Ordered](c *node[E]) *node[E] {
	insert := func(href **node[E], new *node[E]) {
		if *href == nil || !cmp.Less((*href).data, new.data) {
			new.next = *href
			*href = new
		} else {
			c := *href
			for c.next != nil && cmp.Less(c.next.data, new.data) {
				c = c.next
			}
			new.next = c.next
			c.next = new
		}
	}

	var sorted *node[E]
	for c != nil {
		next := c.next
		insert(&sorted, c)
		c = next
	}
	return sorted
}

func merge[E cmp.Ordered](a, b *node[E]) *node[E] {
	var dummy node[E]
	c := &dummy
	for a != nil && b != nil {
		if cmp.Less(a.data, b.data) {
			c.next = a
			c = a
			a = a.next
		} else {
			c.next = b
			c = b
			b = b.next
		}
	}
	if a == nil {
		c.next = b
	} else {
		c.next = a
	}
	c = dummy.next
	dummy.next = nil
	return c
}

func topDownMergeSort[E cmp.Ordered](c *node[E]) *node[E] {
	var (
		a *node[E]
		b *node[E]
	)
	// base case(empty list, or single elem list)
	if c == nil || c.next == nil {
		return c
	}
	a = c
	b = c.next
	for b != nil && b.next != nil {
		c = c.next
		b = b.next.next
	}
	b = c.next
	c.next = nil
	return merge(topDownMergeSort(a), topDownMergeSort(b))
}

func bottomUpMergeSort[E cmp.Ordered](c *node[E]) *node[E] {
	var (
		queue []*node[E]
		t     *node[E]
	)
	for ; c != nil; c = t {
		t = c.next
		c.next = nil
		queue = append(queue, c)
	}
	c = queue[0]
	queue = queue[1:]
	for len(queue) > 0 {
		queue = append(queue, c)
		a := queue[0]
		b := queue[1]
		queue = queue[2:]
		c = merge(a, b)
	}
	return c
}

func makeRndSlice(len, min, max int) []int {
	rs := make([]int, len)
	for i := 0; i < len; i++ {
		rs[i] = min + rand.Intn(max-min+1)
	}
	return rs
}

func makeAscList(len, min, max int) *node[int] {
	var list *node[int]
	rs := makeRndSlice(len, min, max)
	slices.Sort(rs)
	slices.Reverse(rs)
	for _, e := range rs {
		prependList(&list, e)
	}
	return list
}

func makeDescList(len, min, max int) *node[int] {
	var list *node[int]
	rs := makeRndSlice(len, min, max)
	slices.Sort(rs)
	for _, e := range rs {
		prependList(&list, e)
	}
	return list
}

func makeRandList(len, min, max int) *node[int] {
	var list *node[int]
	rs := makeRndSlice(len, min, max)
	for _, e := range rs {
		prependList(&list, e)
	}
	return list
}

func main() {
	list1 := makeRandList(10, 10, 99)
	walk(list1)
	list1 = insertSort(list1)
	walk(list1)

	list2 := makeRandList(10, 10, 99)
	walk(list2)
	list2 = topDownMergeSort(list2)
	walk(list2)

	list3 := makeRandList(10, 10, 99)
	walk(list3)
	list3 = bottomUpMergeSort(list3)
	walk(list3)

	list4 := makeRandList(10, 10, 99)
	walk(list4)
	list4 = inPlaceMergeSort(list4)
	walk(list4)
}
```

上述代码中insertSort 是链表的插入排序, topDownMergeSort 是自顶向下递归的merge sort, bottomUpMergeSort是自底向上迭代的merge sort,
inPlaceMergeSort 是Simon Tatham 的版本

sortlist_test.go:
```golang

package main

import (
	"testing"
)

func testHelper(t *testing.T, sortFn func(*node[int]) *node[int]) {
	ascList := makeAscList(10, 1, 100)
	ascList = sortFn(ascList)
	if !listIsSorted(ascList) {
		t.Fail()
	}

	dscList := makeDescList(10, 1, 100)
	dscList = sortFn(dscList)
	if !listIsSorted(dscList) {
		t.Fail()
	}

	rndList := makeRandList(10, 100, 500)
	rndList = sortFn(rndList)
	if !listIsSorted(rndList) {
		t.Fail()
	}
}

func TestInsertSort(t *testing.T) {
	testHelper(t, insertSort[int])
}

func TestTopDownMergeSort(t *testing.T) {
	testHelper(t, topDownMergeSort[int])
}

func TestBottomUpMergeSort(t *testing.T) {
	testHelper(t, bottomUpMergeSort[int])
}

func TestInPlaceMergeSort(t *testing.T) {
	testHelper(t, inPlaceMergeSort[int])
}

var Sink *node[int]

func benchHelper(b *testing.B, sortFn func(*node[int]) *node[int]) {
	for _, bcase := range []struct {
		name         string
		inputGenFunc func() *node[int]
	}{
		{"smallAscList", func() *node[int] { return makeAscList(50, 10, 100) }},
		{"smallDescList", func() *node[int] { return makeDescList(50, 10, 100) }},
		{"smallRndList", func() *node[int] { return makeRandList(50, 10, 100) }},
		{"bigAscList", func() *node[int] { return makeAscList(1e5, 1, 1e4) }},
		{"bigDescList", func() *node[int] { return makeDescList(1e5, 1, 1e4) }},
		{"bigRndList", func() *node[int] { return makeRandList(1e5, 1, 1e5) }},
	} {
		b.Run(bcase.name, func(b *testing.B) {
			b.ReportAllocs()
			var s *node[int]
			for i := 0; i < b.N; i++ {
				b.StopTimer()
				input := bcase.inputGenFunc()
				b.StartTimer()
				s = sortFn(input)
			}
			Sink = s
		})
	}
}

func BenchmarkInsertSort(b *testing.B) {
	benchHelper(b, insertSort[int])
}

func BenchmarkTopDownMergeSort(b *testing.B) {
	benchHelper(b, topDownMergeSort[int])
}

func BenchmarkBottomUpMergeSort(b *testing.B) {
	benchHelper(b, bottomUpMergeSort[int])
}

func BenchmarkInPlaceMergeSort(b *testing.B) {
	benchHelper(b, inPlaceMergeSort[int])
}
```

一次benchmark 结果:
```bash
BenchmarkInsertSort/smallAscList-12         	  834589	      1442 ns/op	       0 B/op	       0 allocs/op
BenchmarkInsertSort/smallDescList-12        	 6429494	       186.8 ns/op	       0 B/op	       0 allocs/op
BenchmarkInsertSort/smallRndList-12         	 1000000	      1145 ns/op	       0 B/op	       0 allocs/op
BenchmarkInsertSort/bigAscList-12           	       1	7555973814 ns/op	       0 B/op	       0 allocs/op
BenchmarkInsertSort/bigDescList-12          	    4516	    266626 ns/op	       0 B/op	       0 allocs/op
BenchmarkInsertSort/bigRndList-12           	       1	20278468050 ns/op	       0 B/op	       0 allocs/op
BenchmarkTopDownMergeSort/smallAscList-12   	  442106	      2426 ns/op	     784 B/op	      49 allocs/op
BenchmarkTopDownMergeSort/smallDescList-12  	  557928	      2297 ns/op	     784 B/op	      49 allocs/op
BenchmarkTopDownMergeSort/smallRndList-12   	  353748	      3175 ns/op	     784 B/op	      49 allocs/op
BenchmarkTopDownMergeSort/bigAscList-12     	     135	   8902250 ns/op	 1599991 B/op	   99999 allocs/op
BenchmarkTopDownMergeSort/bigDescList-12    	     147	   8189749 ns/op	 1599990 B/op	   99999 allocs/op
BenchmarkTopDownMergeSort/bigRndList-12     	      79	  14935899 ns/op	 1599990 B/op	   99999 allocs/op
BenchmarkBottomUpMergeSort/smallAscList-12  	  352664	      3374 ns/op	    2376 B/op	      57 allocs/op
BenchmarkBottomUpMergeSort/smallDescList-12 	  358518	      3300 ns/op	    2376 B/op	      57 allocs/op
BenchmarkBottomUpMergeSort/smallRndList-12  	  285111	      4122 ns/op	    2376 B/op	      57 allocs/op
BenchmarkBottomUpMergeSort/bigAscList-12    	      73	  14356156 ns/op	 9057681 B/op	  100042 allocs/op
BenchmarkBottomUpMergeSort/bigDescList-12   	      86	  13416789 ns/op	 9057660 B/op	  100042 allocs/op
BenchmarkBottomUpMergeSort/bigRndList-12    	      68	  21171545 ns/op	 9057652 B/op	  100042 allocs/op
BenchmarkInPlaceMergeSort/smallAscList-12   	 1286823	       928.9 ns/op	       0 B/op	       0 allocs/op
BenchmarkInPlaceMergeSort/smallDescList-12  	 1000000	      1107 ns/op	       0 B/op	       0 allocs/op
BenchmarkInPlaceMergeSort/smallRndList-12   	  646136	      1865 ns/op	       0 B/op	       0 allocs/op
BenchmarkInPlaceMergeSort/bigAscList-12     	     285	   4114096 ns/op	       0 B/op	       0 allocs/op
BenchmarkInPlaceMergeSort/bigDescList-12    	     232	   5074325 ns/op	       0 B/op	       0 allocs/op
BenchmarkInPlaceMergeSort/bigRndList-12     	      84	  13954904 ns/op	       8 B/op	       0 allocs/op
```
