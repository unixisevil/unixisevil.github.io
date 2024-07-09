+++
title = "fizzer-buzzer"
date = 2024-07-09

[taxonomies]
tags = ["rust", "atomic", "channel"]
+++

[最近读了一篇关于多线程实现FizzBuzz的blog](https://firedbg.sea-ql.org/blog/2024-06-30-fizzbuzz-multithread/),  作者用unbuffered channel的两次握手来实现线程之间的同步，一次握手由配对的send, recv 调用构成:

```rust
use std::sync::mpsc::{sync_channel, Receiver, SyncSender};

fn fizzer(n : i32,  sender: SyncSender<()>) {
    for i in 1..= n {
        if i % 15 == 0 {
            print!("Fizz");
            sender.send(()).unwrap();
            sender.send(()).unwrap(); // <-
        } else if i % 3 == 0 {
            println!("Fizz");
        } else if i % 5 == 0 {
            sender.send(()).unwrap();
            sender.send(()).unwrap(); // <-
        } else {
            println!("{i}");
        }
    }
}

fn buzzer(receiver: Receiver<()>) {
    while let Ok(()) = receiver.recv() {
        println!("Buzz");
        receiver.recv().unwrap(); // <-
    }
}

fn main() {
     let n = std::env::args()
        .skip(1)
        .take(1)
        .next()
        .map_or(100, |a| a.parse().map_or(100, |v| v));

    let (sender, receiver) = sync_channel(0);
    std::thread::spawn(move || buzzer(receiver));

    fizzer(n,  sender);
}
```

第一次握手确保fizzer 线程的打印语句完成事件 "happens before"  buzzer 线程的打印语句启动事件.
第二次握手确保buzzer 线程的打印语句完成事件 "happens before"  fizzer 线程的下一轮打印语句启动事件.

能否使用atomic 原语实现同样的同步协议?   yes: 

```rust
use std::sync::atomic::AtomicBool;
use std::sync::atomic::Ordering;

fn fizzer(n: i32, buzzer_turn: &AtomicBool, fizzer_done: &AtomicBool) {
    for i in 1..=n {
        if i % 15 == 0 {
            print!("Fizz");
            buzzer_turn.store(true, Ordering::Release);
            while buzzer_turn.load(Ordering::Acquire) {}
        } else if i % 3 == 0 {
            println!("Fizz");
        } else if i % 5 == 0 {
            buzzer_turn.store(true, Ordering::Release);
            while buzzer_turn.load(Ordering::Acquire) {}
        } else {
            println!("{i}");
        }
    }
    fizzer_done.store(true, Ordering::Relaxed);
}

fn buzzer(buzzer_turn: &AtomicBool, fizzer_done: &AtomicBool) {
    'outer: loop {
        while !buzzer_turn.load(Ordering::Acquire) {
            if fizzer_done.load(Ordering::Relaxed) {
                break 'outer;
            }
        }
        println!("Buzz");
        buzzer_turn.store(false, Ordering::Release);
    }
}

fn main() {
    let n = std::env::args()
        .skip(1)
        .take(1)
        .next()
        .map_or(100, |a| a.parse().map_or(100, |v| v));

    let buzzer_turn = AtomicBool::new(false);
    let fizzer_done = AtomicBool::new(false);

    std::thread::scope(|s| {
        s.spawn(|| buzzer(&buzzer_turn, &fizzer_done));
        s.spawn(|| fizzer(n, &buzzer_turn, &fizzer_done));
    });
}
```

性能(throughout) 会好一些 ? 

channel 实现的 throughout:
<img src="/imgs/chan-fizzer.gif">
atomic 实现的 throughout:
<img src="/imgs/atomic-fizzer.gif">

[不过这些自然是没法跟使用AVX2汇编高度优化的实现相比](https://codegolf.stackexchange.com/questions/215216/high-throughput-fizz-buzz/236630#236630),  throughout 根本不在一个数量级别,  根据论坛中描述，作者花费了数个月时间去优化代码，amazing !  amazing ! amazing ! 
