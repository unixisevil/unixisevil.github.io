+++
title = "atomic-order"
date = 2024-07-31

[taxonomies]
tags = ["rust", "atomic"]
+++

[目前rust atomic的rule大致使用C++20的rule](https://doc.rust-lang.org/std/sync/atomic/index.html),  所以我感觉很多相关的东西是互通的， rust社区有Mara Bos 写的 [Rust Atomics and Locks](https://marabos.nl/atomics/),  C++社区有Anthony Williams 写的[C++ Concurrency in Action](https://www.manning.com/books/c-plus-plus-concurrency-in-action-second-edition), 这两本书都是很好的学习资料.

首先看Relaxed ordering, 对于不同的atomic variable 读写，理论上不强制任何Order.
具体到不同的hardware memory model实现， 这个抽象的software model, relaxed ordering 有不同意思，  对于X86硬件来说, relaxed 意味着处理器会respect Program Order 中的  R || R ,  R || W,  W || W,    不会respect 处理器看见的 W || R .   对于Arm硬件来说,  只要读写指令中没有地址依赖，寄存器依赖， 处理器可以reorder 以上四种读写order组合.  [我发现这个讲解ARM memory model的视频很不错](https://www.youtube.com/watch?v=BciWwv9Z0kE). 

Relaxed Ordering 给程序员唯一的保证是对于单个atomic variable 的Total modification order, 
Anthony Williams 有个很形象的比喻, 想象一下，每个atomic variable是一个坐在封闭电话厅的人，手中有一个记事本，外面的人可以打电话与其通信， 他一次只能与一个人通信，你可以电话他，要求给你一个记事本中的数字，或者你告诉他一个数字，要求他在记事本后面追加上，  当你要求首次查询数字时，他可能从记事本中随机挑选一个告诉你， 不过这个号码记录员有个习惯， 每当有人向他查询数字，他会记录这次从哪个位置挑选的数字告诉客户， 如果有客户重复查询， 他会从上次告诉这位客户的位置和当前号码列表的尾部位置之间随机选择一个数字通知客户. 也就是说每个查询客户都可能观察的不是最新的数字(号码列表中尾部位置的数字), 但是客户对号码变化的观察历史不会倒退(也许客户可以观察到数字历史的倒退，因为可能存在数字的重复，但从号码记录员的角度看，这个变化历史是线型往前). 

```rust
use std::fmt;
use std::sync::atomic::{
    AtomicBool, AtomicUsize,
    Ordering::{Acquire, Relaxed, Release},
};

use std::thread;

static X: AtomicUsize = AtomicUsize::new(0);
static Y: AtomicUsize = AtomicUsize::new(0);
static Z: AtomicUsize = AtomicUsize::new(0);
static GO: AtomicBool = AtomicBool::new(false);

const LOOP_COUNT: usize = 10;

#[derive(Default, Copy, Clone)]
struct Sample {
    x: usize,
    y: usize,
    z: usize,
}

impl fmt::Debug for Sample {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {}, {})", self.x, self.y, self.z)
    }
}

fn increment(var: &AtomicUsize, arr: &mut [Sample]) {
    while !GO.load(Acquire) {
        thread::yield_now();
    }
    for (i, e) in arr.iter_mut().enumerate() {
        e.x = X.load(Relaxed);
        e.y = Y.load(Relaxed);
        e.z = Z.load(Relaxed);
        var.store(i + 1, Relaxed);
        thread::yield_now();
    }
}

fn watch(arr: &mut [Sample]) {
    while !GO.load(Acquire) {
        thread::yield_now();
    }
    for e in arr.iter_mut() {
        e.x = X.load(Relaxed);
        e.y = Y.load(Relaxed);
        e.z = Z.load(Relaxed);
        thread::yield_now();
    }
}

fn main() {
    let mut sample1: [Sample; LOOP_COUNT] = Default::default();
    let mut sample2 = sample1;
    let mut sample3 = sample1;
    let mut sample4 = sample1;
    let mut sample5 = sample1;

    thread::scope(|s| {
        s.spawn(|| increment(&X, &mut sample1[..]));
        s.spawn(|| increment(&Y, &mut sample2[..]));
        s.spawn(|| increment(&Z, &mut sample3[..]));
        s.spawn(|| watch(&mut sample4[..]));
        s.spawn(|| watch(&mut sample5[..]));
        GO.store(true, Release);
    });

    println!("sample1: {:?}", sample1);
    println!("sample2: {:?}", sample2);
    println!("sample3: {:?}", sample3);
    println!("sample4: {:?}", sample4);
    println!("sample5: {:?}", sample5);
}
```

某次执行的输出:

```bash
sample1: [(0, 0, 1), (1, 0, 2), (2, 0, 3), (3, 0, 4), (4, 0, 5), (5, 0, 6), (6, 0, 7), (7, 0, 8), (8, 0, 9), (9, 0, 10)]
sample2: [(10, 0, 10), (10, 1, 10), (10, 2, 10), (10, 3, 10), (10, 4, 10), (10, 5, 10), (10, 6, 10), (10, 7, 10), (10, 8, 10), (10, 9, 10)]
sample3: [(0, 0, 0), (1, 0, 1), (2, 0, 2), (3, 0, 3), (4, 0, 4), (5, 0, 5), (6, 0, 6), (7, 0, 7), (8, 0, 8), (9, 0, 9)]
sample4: [(10, 6, 10), (10, 8, 10), (10, 9, 10), (10, 10, 10), (10, 10, 10), (10, 10, 10), (10, 10, 10), (10, 10, 10), (10, 10, 10), (10, 10, 10)]
sample5: [(10, 0, 10), (10, 10, 10), (10, 10, 10), (10, 10, 10), (10, 10, 10), (10, 10, 10), (10, 10, 10), (10, 10, 10), (10, 10, 10), (10, 10, 10)]
```

三个incrementer线程各自单调自增X,Y,Z， 同时观察这三个atomic variable 的值， 其它两个纯观察者:
incrementer 能够即时观察到自己负责修改的变量， 对于其它两个变量不能保证即时观察到最新的值.

Relaxed ordering 不能保证下面关于z 的断言不会失败:

```rust
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering::Relaxed};
use std::thread;

fn main() {
    let x = AtomicBool::new(false);
    let y = AtomicBool::new(false);
    let z = AtomicUsize::new(0);

    let write_x_then_y = || {
        x.store(true, Relaxed);
        y.store(true, Relaxed);
    };

    let read_y_then_x = || {
        while !y.load(Relaxed) {
            std::hint::spin_loop()
        }
        if x.load(Relaxed) {
            z.fetch_add(1, Relaxed);
        }
    };

    thread::scope(|s| {
        s.spawn(write_x_then_y);
        s.spawn(read_y_then_x);
    });

    assert!(z.load(Relaxed) != 0);
}
```

很多常用的并发原语背后都使用了Release-Acquire ordering, 比如Mutex, Channel 等， Release ordering 使得处理器, 编译器respect Program order 中的  RW || W  ,   Acquire ordering 使得它们respect program order 中的  R || RW.   Release-Acquire 给并发实体之间引入了[Partial Order](https://en.wikipedia.org/wiki/Partially_ordered_set).  这种partial order 不能保证以下代码关于z 的断言不会失败:

```rust
use std::thread;
use std::sync::atomic::{
    AtomicBool, AtomicIsize, 
    Ordering::{Release, Acquire, Relaxed},
};

fn main() {
     let x = AtomicBool::new(false);
     let y = AtomicBool::new(false);
     let z = AtomicIsize::new(0);

     let write_x  = ||  x.store(true, Release);

     let write_y  = ||  y.store(true, Release);

     let read_x_then_y  = ||  {
         while !x.load(Acquire) { std::hint::spin_loop() }
         if y.load(Acquire) {
             z.fetch_add(1, Relaxed);
         }
     };

     let read_y_then_x  = ||  {
         while !y.load(Acquire) { std::hint::spin_loop()  }
         if x.load(Acquire) {
             z.fetch_add(1, Relaxed);
         }
     };

     thread::scope(|s| {
         s.spawn(write_x);
         s.spawn(write_y);
         s.spawn(read_x_then_y);
         s.spawn(read_y_then_x);
     });

     assert!(z.load(Relaxed) != 0);
}
```

如果用上面困在电话厅中号码记录员例子做类比，这段代码相当于有三个记录员分别叫x, y, z,  还有四个客户叫write_x , write_y,  read_x_then_y,  read_y_then_x.  客户write_x 的行为是电话z 请求其在列表上追加true,  客户write_y 电话y, 行为类似.   客户read_x_then_y 不厌其烦地电话x直到x 给出true 为止， 然后去电话y;  客户read_y_than_x 行为类似，反向操作. 初始x, y 手中的列表只有false,
read_x_then_y 一直等到了x 手中的列表是[false, true], 然后电话y，此时y 手中的列表可能是[false] , 或者[false, true],  根据以前的规则， 无论是哪种情况，y 都可以从列表中随机选择一个回应， 也就是y的回应可能是false,  对称情形， read_y_then_x 从x 处得到的回应可能是false. 最终可能是read_x_than_y 从 x 收到true, 从y收到false,  “查询z的最新值并自增后追加" 操作没有发生, read_y_than_x 从 y 收到true, 从x收到false, “查询z的最新值并自增后追加" 操作没有发生, z 保持原来的值0. 

Release-Acquire ordering 的好处是为non-atomic 操作和atomic 操作 确立order:

```rust
use std::thread;
use std::sync::atomic::{
    AtomicBool, AtomicIsize, 
    Ordering::{
        Relaxed,
        Release,
        Acquire,
    },
};

static mut X: bool  = false;
static Y: AtomicBool = AtomicBool::new(false);
static Z: AtomicIsize = AtomicIsize::new(0);

fn main() {
    let write_x_then_y = || {
         unsafe { X = true };
         Y.store(true, Release);
    };

    let read_y_then_x  = ||  {
         while !Y.load(Acquire) { std::hint::spin_loop()  }
         if unsafe { X } {
             Z.fetch_add(1, Relaxed);
         }
     };

     thread::scope(|s| {
         s.spawn(write_x_then_y);
         s.spawn(read_y_then_x);
     });

     assert!(Z.load(Relaxed) != 0);
}
```

```rust
use std::sync::atomic::{
    AtomicBool, AtomicUsize,
    Ordering::{Acquire, Relaxed, Release},
};
use std::thread;

fn main() {
    let x = AtomicBool::new(false);
    let y = AtomicBool::new(false);
    let z = AtomicUsize::new(0);

    let write_x_then_y = || {
        x.store(true, Relaxed);
        y.store(true, Release);
    };

    let read_y_then_x = || {
        while !y.load(Acquire) {
            std::hint::spin_loop()
        }
        if x.load(Relaxed) {
            z.fetch_add(1, Relaxed);
        }
    };

    thread::scope(|s| {
        s.spawn(write_x_then_y);
        s.spawn(read_y_then_x);
    });

    assert!(z.load(Relaxed) != 0);
}
```

这两段代码中关于z 的断言不会失败

Release-Acqire ordering  具有传递性:

```rust
use std::sync::atomic::{
    AtomicIsize,
    Ordering::{AcqRel, Acquire, Relaxed, Release},
};

use std::thread;

fn main() {
    let arr = [const { AtomicIsize::new(0) }; 5];
    let sync = AtomicIsize::new(0);

    let t1 = || {
        arr[0].store(5, Relaxed);
        arr[1].store(18, Relaxed);
        arr[2].store(7, Relaxed);
        arr[3].store(-29, Relaxed);
        arr[4].store(2024, Relaxed);
        sync.store(1, Release);
    };

    let t2 = || {
        while sync.compare_exchange(1, 2, AcqRel, Relaxed).is_err() {
            std::hint::spin_loop()
        }
    };

    let t3 = || {
        while sync.load(Acquire) < 2 {
            std::hint::spin_loop()
        }
        assert!(arr[0].load(Relaxed) == 5);
        assert!(arr[1].load(Relaxed) == 18);
        assert!(arr[2].load(Relaxed) == 7);
        assert!(arr[3].load(Relaxed) == -29);
        assert!(arr[4].load(Relaxed) == 2024);
    };

    thread::scope(|s| {
        s.spawn(t1);
        s.spawn(t2);
        s.spawn(t3);
    });
}
```

如果某个包含Release 语义的原子操作， 后面跟随对同一个atomic variable 的一连串的 atomic RMW(read-modify-write) 操作, 可以形成所谓的Release Sequence.  我感觉Mara Bos在 [More Formally Section](https://marabos.nl/atomics/memory-ordering.html#release-and-acquire-ordering) 解释的很好. 以下是一个Release Sequence 建立happens-before 关系的例子:

```rust
use std::cell::UnsafeCell;
use std::sync::atomic::{
    AtomicIsize,
    Ordering::{Acquire, Release},
};
use std::thread;

struct Queue {
    count: AtomicIsize,
    q: UnsafeCell<Vec<isize>>,
}

unsafe impl Sync for Queue {}

impl Queue {
    fn new() -> Self {
        Self {
            count: AtomicIsize::new(0),
            q: UnsafeCell::new(Vec::<isize>::new()),
        }
    }

    fn populate(&self) {
        let num_of_items: isize = 20;
        for i in 0..num_of_items {
            unsafe { (*self.q.get()).push(i) };
        }
        self.count.store(num_of_items, Release);
    }

    fn consume(&self, id: usize) {
        let mut data_touch = false;
        let mut woken = false;
        loop {
            let remain = self.count.fetch_sub(1, Acquire);
            if remain <= 0 {
                if !data_touch && !woken {
                    std::thread::park();
                    woken = true;
                    continue;
                } else {
                    break;
                }
            }
            println!("c{} got item: {}", id, unsafe {
                (&*self.q.get())[remain as usize - 1]
            });
            data_touch = true;
        }
    }
}

fn main() {
    let q = Queue::new();
    thread::scope(|s| {
        let c1 = s.spawn(|| q.consume(1));
        let c2 = s.spawn(|| q.consume(2));
        q.populate();
        c1.thread().unpark();
        c2.thread().unpark();
    });
}
```

这个Release-Seqence有对count 的release store 和后面的对count 的fetch_sub 构成,  确保了填充队列完成 happens-before 两个队列数据消费者开始使用数据,  同时两个队列消费者之间没有建立order.

Release-Acquire 存在一个特别case， Release-Consume,  是希望能弱化Acquire 端对Order的强制， 在program order 中排在Acquire atomic operation 或者Acquire fence 后面的所有操作都不能reorder 到 Acquire 之前， 人们觉得杀伤范围过大, 能不能把order约束只限制在“相关依赖数据" 上， 其它无关语句仍然可以被编译器和处理器重排， 来获得更好的性能. [尤其是像Paul McKenney等一些专家比较关心这些](https://www.youtube.com/watch?v=ZrNQKpOypqU&t=1803s), 可能是这个概念不好精确定义，编译器开发者很难实现它， [自从C++ 17以来， Release-Consume不在被推荐](https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Consume_ordering).

Rust当前也没有引入Consume Ordering, 不过看起来[crossbeam当前在ARM平台上使用Release Ordering + 一个Acquire语义的compiler_fence 来模拟](https://docs.rs/crossbeam-utils/latest/src/crossbeam_utils/atomic/consume.rs.html#34-48), 使用crossbeam 的consume  demo: 

```rust
#![feature(new_uninit)]

use crossbeam_utils::atomic::AtomicConsume;
use std::ptr::addr_of_mut;
use std::sync::atomic::{
    AtomicIsize, AtomicPtr,
    Ordering::{Relaxed, Release},
};
use std::{thread, time::Duration};

struct SomeStruct {
    i: isize,
    s: String,
}

fn main() {
    let p = AtomicPtr::new(std::ptr::null_mut());
    let a = AtomicIsize::new(0);

    let init_some_struct = || {
        let mut uninit = Box::<SomeStruct>::new_uninit();
        let ss = {
            let pss = uninit.as_mut_ptr();
            unsafe {
                addr_of_mut!((*pss).i).write(42);
                addr_of_mut!((*pss).s).write("Hi".to_string());
                uninit.assume_init()
            }
        };
        a.store(100, Relaxed);
        p.store(Box::into_raw(ss), Release);
    };

    let use_some_struct = || {
        let pss = loop {
            let pss = p.load_consume();
            if !pss.is_null() {
                break pss;
            }
            thread::sleep(Duration::from_micros(1));
        };
        let ss = unsafe { Box::from_raw(pss) };
        //dbg!(ss.i);
        //dbg!(ss.s);
        assert!(ss.s == *"Hi");
        assert!(ss.i == 42);
        assert!(a.load(Relaxed) == 100);
    };

    thread::scope(|s| {
        s.spawn(init_some_struct);
        s.spawn(use_some_struct);
    });
}
```

Release-Consume Ordering 保证use_some_struct 线程观察到SomeStruct被合适地初始化了，仅此而已， 不能保证atomic variable a 的断言不会失败, 尤其是在weakly ordered machine 上，即使是ARM这种宣称weakly order 架构上， 不同的处理器厂商实现处理器时可能又做了order 的强化.

SeqCst 是对Order 保证最强的model， 但也可能是对性能伤害最大的， 尤其是在weakly order 的机器上，给予程序  RW || RW 的ordering 保证, 甚至X86处理器都没实现SeqCst.

```rust
use std::thread;
use std::sync::atomic::{AtomicBool, AtomicIsize, Ordering::SeqCst};


fn main() {
     let x = AtomicBool::new(false);
     let y = AtomicBool::new(false);
     let z = AtomicIsize::new(0);

     let write_x  = ||  x.store(true, SeqCst);

     let write_y  = ||  y.store(true, SeqCst);

     let read_x_then_y  = ||  {
         while !x.load(SeqCst) { std::hint::spin_loop() }
         if y.load(SeqCst) {
             z.fetch_add(1, SeqCst);
         }
     };

     let read_y_then_x  = ||  {
         while !y.load(SeqCst) { std::hint::spin_loop()  }
         if x.load(SeqCst) {
             z.fetch_add(1, SeqCst);
         }
     };

     thread::scope(|s| {
         s.spawn(write_x);
         s.spawn(write_y);
         s.spawn(read_x_then_y);
         s.spawn(read_y_then_x);
     });

     assert!(z.load(SeqCst) != 0);
}
```
由于SeqCst 要求所有并发实体观察到的World View 必须一致, 断言不会失败.
SeqCst 的model 要求所有SeqCst 语义的操作形成一个全局的order，这种global order 仿佛是在保持每个线程的program order 的前提下， 任意交错合并而成的order. 
