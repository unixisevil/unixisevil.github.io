+++
title = "footguns in async rust"
date = 2024-06-26

[taxonomies]
tags = ["rust", "async", "footgun"]
+++

最近读到一篇关于[async rust痛点的blog](https://skepfyr.me/blog/futures-liveness-problem/), 其中用两个例子展示了如果对future和stream(async iterator) 处理不当可能引起的一些问题, stream 的例子来自[Tyler Mandry的这篇blog](https://tmandry.gitlab.io/blog/posts/for-await-buffered-stream) 

首先是这篇blog作者自己描述的一个dead lock 的例子:

```rust
#[allow(unused_mut)]

#[tokio::main]
async fn main() {
    let mutex = tokio::sync::Mutex::new(());
    let lock = mutex.lock().await;

    let mut a = Box::pin(async {
        println!("A1");
        let _ = mutex.lock().await;
        println!("A2");
    });

    let mut b = Box::pin(async {
        println!("B1");
        drop(lock);
        println!("B2");
    });

    let _keep = futures::future::select(a, b).await;

    println!("Main 1");
    let _ = mutex.lock().await;
    println!("Main 2");
}
```

使用[futures crate 的select  function](https://docs.rs/futures/0.3.30/futures/future/fn.select.html) 并发等待future a 和 future b，  await select function 的返回结果, 意味着[这个新的future需要poll a 和 b](https://docs.rs/futures-util/0.3.30/src/futures_util/future/select.rs.html#94-124)
由于future b 自身没有引入await point， 被一下poll 到完成， future a 打印完"A1"后就让出了执行线程,  在future b 得到执行机会时， 顺便把future main 持有的lock 释放了， 然后future a 应当从获取lock 的await 点被唤醒接着执行了， 但是select 函数返回的future: 我们称为future Select， future Select 只会poll 传入的参数 a, b 各自一次， a, b 中的“winner” 返回时， future Select返回， future Select 既不继续poll "loser"  ， 也不drop "loser"； future main 打印完”Main 1" ,再次尝试抢lock, 但是future a 已经在mutex 的等待队列中排在前面， 所以main future在此处的await 点以pending 状态返回，  类似队列前面的future a， 没人去继续poll 它们. 

解决deadlock 可以在main future 第二次尝试获取lock 之前， drop future Select， 由于future Select “own"  future a，b ，  drop _keep  递归地 drop a, b , 让future main 在await 点处以ready 状态返回:  

```rust
 drop(_keep);
```

这样输出就是: 

```bash
A1
B1
B2
Main 1
Main 2
```

更好是把future a 也poll 到完成:

```rust
    let _keep = futures::future::select(a, b).await;

    match _keep {
        futures::future::Either::Left((_, r)) => r.await,
        futures::future::Either::Right((_, r)) => r.await,
    };
```

tokio::select! 管理并发分支的方式是直接drop "loser", 替换futures::future::select:

```rust
    tokio::select! {
        _ = a => {},
        _ = b => {},
    }
```

但是如果tokio::select! 没有接管future a, b 的ownership的话， 同样存在死锁风险:

```rust
    tokio::select! {
        _ = &mut a => {},
        _ = &mut b => {},
    }
```

<img src="/imgs/dk-pb-bad.gif">

使用tokio::select! 也可以把future a, b poll到完成吗?  
不让tokio::select! 接管a,b 的ownership, 我们手动去poll 竞争中的落后一拍的future:

```rust
    let winner;
    tokio::select! {
        _ = &mut a => { winner = Some('a') },
        _ = &mut b => { winner = Some('b') },
    }

    if matches!(winner, Some(w) if w  == 'a') {
        b.await;
    }else {
        a.await;
    }
```

<img src="/imgs/dk-good.gif">

async stream 的一些拓展方法也可能产生意外效果, 比如[buffered](https://docs.rs/futures/0.3.30/futures/stream/trait.StreamExt.html#method.buffered),   上面链接的Tyler Mandry 的blog中详细描述了原因, 我提供一个简化可以运行的版本:

```rust
use futures::stream::{self, StreamExt};
use std::error::Error;
use std::result::Result;
use tokio::time::{Duration, Instant, sleep};

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let c = std::env::args().skip(1).take(1).next().is_some();

    let mut stream = stream::iter(1..=10)
        .map(|item| async move {
            let now = Instant::now();
            sleep(Duration::from_secs(1)).await;
            println!("compute: {}, delay: {:?}", item, now.elapsed());
            item * 2
        })
        .buffered(3);

    let star_fn = |item: i32| async move {
        sleep(Duration::from_secs(5)).await;
        format!("***** {:?} *****", item)
    };

    if c {
        stream
            .for_each_concurrent(3, |item| async move {
                star_fn(item).await;
            })
            .await;
    } else {
        while let Some(item) = stream.next().await {
            star_fn(item).await;
        }
    }

    Ok(())
}

```

else branch 版本， 可能导致stream 中的某些future 得不到即时被poll而超时

<img src = "/imgs/asyncit-bad.gif"> 

if  branch 版本， 使用Tyler的尽量“fan out" buffer 中的future，让buffer中的futures 与 loop body 中的future 并发起来 (如果我没有理解出错的话)


<img src = "/imgs/asyncit-good.gif"> 

