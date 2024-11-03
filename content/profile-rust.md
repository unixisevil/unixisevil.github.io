+++
title = "Profiling Rust code using Samply"
date = 2024-10-31

[taxonomies]
tags = ["rust", "profile", "samply"]
+++

之前提到的[Elixir String Processing Optimization](https://blog.jola.dev/posts/elixir-string-processing-optimization)文章中使用测试文本来自[这里](http://ptrace.fefe.de/wp/timings2019.txt), 这个链接张贴了多种语言实现的测试结果, 看了看其中rust 的实现, 一个是单线程的实现[wpopt](https://ptrace.fefe.de/wp/wpopt.rs) , 另一个多线程的实现[wp-rs](https://ptrace.fefe.de/wp/wp-rs.tar.xz).

同时下载了C的实现[wp2.c](https://ptrace.fefe.de/wp/wp2.c),  编译后(gcc 11.4.0 , rustc 1.82.0), 测试对比了一下:

wp2: 

```bash
Benchmark 1: cat ./words-llvm8.0.txt | ./wp2
  Time (mean ± σ):      1.635 s ±  0.109 s    [User: 1.531 s, System: 0.194 s]
  Range (min … max):    1.505 s …  1.845 s    10 runs
```

wpopt: 

```bash
Benchmark 1: cat ../words-llvm8.0.txt | ./target/release/wpopt
  Time (mean ± σ):      2.065 s ±  0.018 s    [User: 1.732 s, System: 0.328 s]
  Range (min … max):    2.045 s …  2.105 s    10 runs
```

[samply](https://github.com/mstange/samply/) 的使用十分简单:

```bash
samply record ./my-application my-arguments
```

```bash
cat ../words-llvm8.0.txt | samply record ./target/release/wpopt
```

命令执行后自动打开firefox 浏览器窗口: 
<img src="/imgs/wpopt.samply.png">

从截图可以看出， 主要时间花费在hash table 相关， stdout 打印输出相关code.

当前rust的hash table 默认hash 算法是siphash, 有一定的防dos能力， 可是速度比较慢， 可以换成别的hash 算法, 比如[rustc_hash](https://crates.io/crates/rustc-hash), wp2 的实现也是使用比较普通的hash算法，wp2的实现缓冲stdout 的输出， wpopt.rs 也可以加上,  wp2 的实现已经假设文本是ascii 编码, rust 的String 类型是UTF-8 编码，带来少许开销, 可以使用Vec<u8>, &[u8] 等替代.

wpopt_profile:

```rust
use rustc_hash::FxHashMap;
use std::fs::File;
use std::io::Write;
use std::io::{self, Read};
use std::iter::FromIterator;
use std::os::fd::AsFd;

fn main() -> io::Result<()> {
    let mut buffer = Vec::new();
    io::stdin().read_to_end(&mut buffer)?;
    let buffer = unsafe { String::from_utf8_unchecked(buffer) };

    // Primitive Tokenize
    let mut frequency: FxHashMap<&str, u32> = FxHashMap::default();
    for word in buffer.split_ascii_whitespace() {
        *frequency.entry(word).or_insert(0) += 1;
    }

    // Sort by value
    let mut frequency = Vec::from_iter(frequency);
    frequency.sort_by(|&(_, a), &(_, b)| b.cmp(&a));

    let mut stdout = io::BufWriter::with_capacity(
        4096,
        File::from(io::stdout().as_fd().try_clone_to_owned().unwrap()),
    );

    // Push to stdout
    for (word, count) in frequency {
        write!(&mut stdout, "{}", format_args!("{} {}\n", count, word))?;
    }
    Ok(())
}
```

bench 结果:

```bash
  Benchmark 1: cat ../words-llvm8.0.txt | ./target/release/wpopt_profile
  Time (mean ± σ):      1.275 s ±  0.006 s    [User: 1.003 s, System: 0.265 s]
  Range (min … max):    1.266 s …  1.285 s    10 runs
```

 检查多线程的实现wp-rs 的时候，发现了其中的bug, 导致了单词被撕裂, 估计是不小心:

```rust
.map(|mut pos| {
          while pos < buffer_as_bytes.len() && buffer_as_bytes[pos] == ' ' as u8 {
              pos += 1;
          }
          pos
      })
```

修复后的，bench 结果:

```bash
   Benchmark 1: cat ../words-llvm8.0.txt | ./target/release/wp-rs
  Time (mean ± σ):     678.2 ms ±   4.6 ms    [User: 2010.8 ms, System: 295.0 ms]
  Range (min … max):   673.4 ms … 687.8 ms    10 runs                               
```                                           
                                           
应用上面针对wpopt的优化策略后, 不过升级了hashbrown到0.15.0，从这个版本开始默认hash算法从siphash 切换成foldhash, 移除对itertools 的依赖, 标准库的reduce就可以实现foldl 操作了.
 
```rust
use std::fs::File;
use std::io::{self, Read, Write};
use std::iter::{FromIterator, Iterator};
use std::os::fd::AsFd;
use std::str;
use std::thread;

use hashbrown::HashMap;

type Frequencies<'a> = HashMap<&'a str, usize>;

fn main() -> io::Result<()> {
    // Read string from stdio
    let mut buffer = Vec::new();
    io::stdin().read_to_end(&mut buffer)?;

    // Figure out optimal chunk size
    let num_threads = num_cpus::get();
    let chunk_size = buffer.len() / num_threads;

    // Figure out positions where we can split string into chunks
    // We only split on whitespaces
    let mut split_positions: Vec<_> = (0..num_threads).map(|i| i * chunk_size).collect();
    split_positions.push(buffer.len());

    let split_positions: Vec<_> = split_positions
        .into_iter()
        .map(|mut pos| {
            while pos < buffer.len() && buffer[pos] != b' ' && buffer[pos] != b'\n' {
                pos += 1;
            }
            pos
        })
        .collect();

    // Create references to seperate chunks
    let chunks = (0..num_threads).map(|i| {
        buffer
            .get(split_positions[i]..split_positions[i + 1])
            .unwrap()
    });

    // Tokenize each chunk on a seperate thread
    let frequencies: Result<Vec<_>, _> = thread::scope(|s| {
        let threads: Vec<_> = chunks
            .map(|chunk| {
                s.spawn(move || {
                    let mut frequencies = Frequencies::new();
                    let chunk = unsafe { str::from_utf8_unchecked(chunk) };
                    for word in chunk.split_ascii_whitespace() {
                        *frequencies.entry(word).or_insert(0) += 1;
                    }
                    frequencies
                })
            })
            .collect();

        threads.into_iter().map(|t| t.join()).collect()
    });

    // Merge results from each thread into a single HashMap
    let frequencies = frequencies.unwrap();
    let frequencies = frequencies
        .into_iter()
        .reduce(merge_frequencies)
        .unwrap_or_default();

    // Sort by value
    let mut frequencies = Vec::from_iter(frequencies);
    frequencies.sort_by(|&(_, a), &(_, b)| b.cmp(&a));

    let stdout = io::stdout();
    let mut stdout = io::BufWriter::with_capacity(
        65536,
        File::from(stdout.as_fd().try_clone_to_owned().unwrap()),
    );

    // Push to stdout
    for (word, count) in frequencies {
        write!(&mut stdout, "{}", format_args!("{} {}\n", count, word))?;
    }

    Ok(())
}

fn merge_frequencies<'a>(mut a: Frequencies<'a>, b: Frequencies<'a>) -> Frequencies<'a> {
    for (word, count) in b {
        *a.entry(word).or_insert(0) += count
    }
    a
}                                                                                  
```
  
bench 结果:
  
```bash
 Benchmark 1: cat ../words-llvm8.0.txt | ./target/release/wp-rs-profile
  Time (mean ± σ):     594.6 ms ±   3.2 ms    [User: 1529.0 ms, System: 289.9 ms]
  Range (min … max):   590.5 ms … 599.8 ms    10 runs 
```
Update:  rust的数据并行处理库[rayon](https://docs.rs/rayon/latest/rayon/), 可能和elixir的[flow](https://hexdocs.pm/flow/Flow.html) 比较类似，尝试使用rayon写同样的测试code:
```rust
use std::fs::File;
use std::io::{self, Read, Write};
use std::os::fd::AsFd;
use std::str;

use hashbrown::HashMap;
use rayon::prelude::*;

type Frequencies<'a> = HashMap<&'a str, usize>;

fn main() -> io::Result<()> {
    let mut buffer = Vec::new();
    io::stdin().read_to_end(&mut buffer)?;

    let positions = even_parts_positions(buffer.len(), rayon::current_num_threads());

    let mut wcs = positions
        .into_par_iter()
        .map(|mut pos| {
            while pos < buffer.len() && buffer[pos] != b' ' && buffer[pos] != b'\n' {
                pos += 1;
            }
            pos
        })
        .collect::<Vec<_>>()
        .par_windows(2)
        .fold(
            //Frequencies::new,
            || Frequencies::with_capacity(50021),
            |mut m: Frequencies, r: &[usize]| {
                let chunk = unsafe { str::from_utf8_unchecked(&buffer[r[0]..r[1]]) };
                for word in chunk.split_ascii_whitespace() {
                    *m.entry(word).or_insert(0) += 1;
                }
                m
            },
        )
        .reduce(
            //Frequencies::new,
            || Frequencies::with_capacity(557733),
            |mut acc: Frequencies, m: Frequencies| {
                for (word, count) in m {
                    *acc.entry(word).or_insert(0) += count
                }
                acc
            },
        )
        .into_par_iter()
        .collect::<Vec<(_, _)>>();

    wcs.par_sort_unstable_by(|&(_, a), &(_, b)| b.cmp(&a));

    let mut stdout = io::BufWriter::with_capacity(
        65536,
        File::from(io::stdout().as_fd().try_clone_to_owned().unwrap()),
    );

    for (w, c) in wcs {
        write!(&mut stdout, "{}", format_args!("{} {}\n", c, w))?;
    }
    Ok(())
}

fn even_parts_positions(n: usize, p: usize) -> Vec<usize> {
    assert!(n >= p);
    let r = n % p;
    let d = (n - r) / p;
    let mut ret = vec![0usize; p + 1];

    for i in 1..=p - r {
        ret[i] = i * d
    }
    for i in p - r + 1..p + 1 {
        ret[i] = ret[i - 1] + (d + 1)
    }
    ret
}
```
bench 结果:
```bash
Benchmark 1: cat ../words-llvm8.0.txt | ./target/release/wf-rayon
  Time (mean ± σ):     580.6 ms ±   2.1 ms    [User: 1881.9 ms, System: 411.3 ms]
  Range (min … max):   578.0 ms … 583.2 ms    10 runs
```
