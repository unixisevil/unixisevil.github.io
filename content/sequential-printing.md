+++
title = "approaching a popular interview question from different angles"
date = 2026-02-06

[taxonomies]
tags = ["golang"]
+++

遇过几次golang面试要求写程序: 多个goroutines 顺序地打印数字输出到终端. 好像已经成为了golang面试八股文之一了？这问题从何而来？
想必是出题者意图通过这题来考察应聘者对golang并发的基本掌握情况. 面试官脑海中已有标准答案了？ 如果应聘者通过刷题或者强记所谓标准答案，能够答对，就说明对golang并发有了基本掌握？ 

现在让我化身为孔乙己，展示回字有几种写法,希望以后没人再问我这个问题. 可能标准的方式是通过channel来解决此题, 使用channel也有通过channel发送数字，或者利用channel向下一个goroutine发送"signal":

seqprint_chan0_val.go:
```golang 
wg := sync.WaitGroup{}
chs := make([]chan int, n)
for i := range chs {
	chs[i] = make(chan int)
}
for i := range n {
	wg.Go(func() {
		val := <-chs[i]
		fmt.Println(val)
		if i < n-1 {
			chs[i+1] <- val + 1
		}
	})
}
chs[0] <- 1
wg.Wait()
```

seqprint_chan1_val.go:
```golang
wg := sync.WaitGroup{}
chs := make([]chan int, n)
for i := range chs {
		chs[i] = make(chan int, 1)
}
chs[0] <- 1
for i := range n {
	wg.Go(func() {
		val := <-chs[i]
		fmt.Println(val)
		if i < n-1 {
			chs[i+1] <- val + 1
		}
	})
}
wg.Wait()
```
seqprint_chan_close.go:
```golang
wg := sync.WaitGroup{}
chs := make([]chan struct{}, n)
for i := range chs {
	chs[i] = make(chan struct{})
}
close(chs[0])
for i := range n {
	wg.Go(func() {
		<-chs[i]
		fmt.Println(i + 1)
		if i < n-1 {
			close(chs[i+1])
		}
	})
}
wg.Wait()
```

对于使用unbuffered chan 来发送val来说, 启动信号: "chs[0] <- 1" 在loop 之后是必须; 然而使用1 size chan 或者通过close chan来广播信号来说，启动信号: "chs[0] <- 1" or "close(chs[0])" 在loop前或者loop后，只是性能的差异.

以下是可能算是非标准答案了，使用sync.Cond:

seqprint_broadcast.go:
```golang

var (
		mu   sync.Mutex
		wg   sync.WaitGroup
		cond = sync.NewCond(&mu)
		turn = 0
)

for i := range n {
	wg.Go(func() {
		mu.Lock()
		for turn != i {
			cond.Wait()
		}
		fmt.Println(i + 1)
		turn++
		cond.Broadcast()
		mu.Unlock()
	})
}
wg.Wait()
```
seqprint_cond.go:
```golang

var (
	wg sync.WaitGroup
	mu sync.Mutex
)
cvs := make([]sync.Cond, n)

for i := range n {
	cvs[i].L = &mu
}
turn := 0
for i := range n {
	wg.Go(func() {
		mu.Lock()
		for turn != i {
			cvs[i].Wait()
		}
		turn++
		fmt.Println(i + 1)
		mu.Unlock()
		if i < n-1 {
			cvs[i+1].Signal()
		}
	})
}
wg.Wait()
```
seqprint_broadcast.go 和 seqprint_cond.go 区别是使用一个cond variable 还是多个，在这个问题上使用broadcast有thundering-herd 效果，所以性能较差. 感觉不少golang 程序员对condition variable 并不太熟悉, 可能与官方文档建议少用, 对比channel不能较容易正确地使用有关.
跟c, c++, rust 的condition variable api 比较, golang 的缺少超时wait api, 不过golang的实现，文档指示golang 的cv 没有虚假唤醒问题.

仅使用lock或者wait group可能也不是标准答案

 seqprint_lock.go:  
```golang
wg := sync.WaitGroup{}
locks := make([]sync.Mutex, n)
for i := 1; i < n; i++ {
	locks[i].Lock()
}
for i := range n {
	wg.Go(func() {
		locks[i].Lock()
		fmt.Println(i + 1)
		if i < n-1 {
			locks[i+1].Unlock()
		}
	})
}
wg.Wait()
```
seqprint_wg.go:
```golang
starts := make([]sync.WaitGroup, n)
for i := 1; i < n; i++ {
	starts[i].Add(1)
}
wg := sync.WaitGroup{}
for i := range n {
	wg.Go(func() {
		starts[i].Wait()
		fmt.Println(i + 1)
		if i < n-1 {
			starts[i+1].Done()
		}
	})
}
wg.Wait()
```

仅使用atomic:

seqprint_one_atomic.go:
```golang
var (
	turn atomic.Uintptr
	wg   sync.WaitGroup
)
for i := uintptr(0); i < uintptr(n); i++ {
	wg.Go(func() {
		for turn.Load() != i {
			runtime.Gosched()
		}
		fmt.Println(i + 1)
		turn.Store(i + 1)
	})
}
wg.Wait()
```
seqprint_multi_atomics.go:
```golang
flags := make([]struct {
	_pad [60]byte
	flag atomic.Bool
}, n)
flags[0].flag.Store(true)
wg := sync.WaitGroup{}
for i := range n {
	wg.Go(func() {
		for !flags[i].flag.Load() {
			runtime.Gosched()
		}
		fmt.Println(i + 1)
		if i < n-1 {
			flags[i+1].flag.Store(true)
		}
	})
}
wg.Wait()
```
使用类似spin loop的方案来解决关键区包含慢速IO任务问题并不是好主意.

尝试理解[parking crate](https://docs.rs/parking/latest/parking/)的过程中, 顺便移植了它的逻辑来解决这个问题:

seqprint_parker.go:
```golang
wg := sync.WaitGroup{}
parkers := make([]*Parker, n)
for i := range n {
	parkers[i] = New()
}
parkers[0].Unpark()
for i := range n {
	wg.Go(func() {
		parkers[i].Park()
		fmt.Println(i + 1)
		if i < n-1 {
			parkers[i+1].Unpark()
		}
	})
}
wg.Wait()
```
一个parker对象基本是一个三状态的原子变量 + 条件变量(仅用于提供休眠和唤醒), [full source code](https://gist.github.com/unixisevil/3c66a6549ef48803ffd78b47dd29556d#file-seqprint_chan0_val-go).

