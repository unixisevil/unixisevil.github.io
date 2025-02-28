+++
title = "Traps of Laziness"
date = 2025-02-28

[taxonomies]
tags = ["haskell", "elixir", "kotlin", "rust"]
+++
最近学到关于haskell如何处理IO的部分，作为一门纯函语言, 需要使用Monad来追踪side effet, 感觉大致可以理解,  但默认的laziness对于习惯了使用strict language 的我感觉有些不太适应.

看这段问题代码: 
```Haskell
makeAndReadFile :: Int -> IO String
makeAndReadFile fnumber =
  let fname = "/tmp/test/" <> show fnumber
   in writeFile fname fname >> readFile fname

unsafe :: IO ()
unsafe =
  let files = mapM makeAndReadFile [1 .. 50000] :: IO [String]
   in files >>= print

main = unsafe
```
当你运行时, 会发现遇到报类似错误, 超过linux 系统默认的最大打开文件描述符限制:
```bash
unsafeiotest: /tmp/test/1021: withFile: resource exhausted (Too many open files)
```
主要是因为haskell默认是lazy的，通常表达式的值不会被立即计算, 而是被推迟到将来某个不得不计算的时刻. 比如上面的unsafe函数中let
binding 中的files 一开始并没有被计算，直到遇到IO action print， 在print的驱动下，整个计算才开始运作, mapM 相当于制造了一个loop,
把字符串fname 写入新建文件fname, 然后读取新文件的内容并返回. 
可是[readFile](https://hackage.haskell.org/package/ghc-internal-9.1201.0/docs/src/GHC.Internal.System.IO.html#readFile)的实现并没有关闭文件句柄, 在loop中快速堆积了大量未关闭的句柄, 以至超过系统默认文件句柄限制. 

使用non-lazy 的readFile'替换readFile 可以避免这个问题; 或者使用strict类型Text 替换String 类型; 或者把unsafe 中的print挪到readFile fname 后面, 结合unsafe中使用mapM_,  这样程序语义稍有改变， 不再是打印一个包裹在 IO 上下文的 String List, 而是每轮loop的末尾是一个print, readFile仍然在泄漏文件句柄, 为什么没问题， 不太清楚，我猜测是慢速的print给ghc runtime 更多的时间来回收readFile 中托管被弃用文件句柄临时变量, gc finalizer 起作用了? 

同样有问题的模式不只是Haskell独有，只是其默认的laziness 与IO 交互显得比较突出. 不少其它语言也提供了一定程度的laziness
比如Elixir 的Stream， Rust的iterator,  Kotlin 的Sequence.  通过不恰当地使用某些API 结合语言的laziness 都可以制造类似的bug.

Elixir: 
```elixir
defmodule Test do
  def make_and_read_file(fnumber) when is_integer(fnumber) do
    fname = "/tmp/test/" <> Integer.to_string(fnumber)
    File.write!(fname, fname)
    fh = File.open!(fname, [:raw, :read])

    case IO.binread(fh, :line) do
      :eof -> "eof"
      {:error, reason} -> ["encounter some error:", reason]
      data -> data
    end
    |> IO.puts()

    fh
  end

  def unsafe() do
    Stream.map(1..50000, &make_and_read_file/1)
    |> Enum.to_list()

    # |> Enum.map(&File.close/1)
  end
end

Test.unsafe()
```

Elixir 标准库中一些便利的high level io/file  api 一般是通过消息传递的方式代理给beam process 处理， 文件句柄等系统资源由其owner process 管理, 这里使用raw mode 打开， 避免创建beam process. 

Rust:
```rust
use std::fs;
use std::fs::File;
use std::io::Read;
use std::os::fd::OwnedFd;
use std::path::Path;

fn main() {
    let make_and_read_file = |n: i32| -> Option<OwnedFd> {
        let fname = Path::new("/tmp/test/").join(n.to_string());
        match fs::write::<&Path, &str>(fname.as_ref(), fname.to_str().unwrap()) {
            Ok(()) => {}
            Err(e) => {
                println!("during write file: {e:?}");
                return None;
            }
        }

        let mut f = match File::open::<&Path>(fname.as_ref()) {
            Err(e) => {
                println!("during open file : {e:?}");
                return None;
            }
            Ok(f) => f,
        };

        let mut buffer = String::new();
        match f.read_to_string(&mut buffer) {
            Err(e) => {
                println!("read_to_string: {e:?}");
                return None;
            }
            Ok(_) => println!("{buffer}"),
        };
        Some(f.into())
    };

    let _fds = (1..50001).map(make_and_read_file).collect::<Vec<_>>();
    /*
        (1..50001)
            .map(make_and_read_file)
            .for_each(|fd| println!("{fd:?}"));
    */
}
```
这里故意避免了更为方便的fs::read, 而是通过open, read_to_string, 然后把File 转换成OwnedFd, 方便观察， 通常File 通过Drop 会自动管理句柄关闭.

Kotlin: 
```kotlin
package org.example
import java.io.File
import java.io.FileDescriptor
fun makeAndReadFile(fnum: Int) : FileDescriptor {
    val file = File("/tmp/test/$fnum")
    file.writeText("/tmp/test/$fnum")
    //println(file.readText())
    println(file.inputStream().reader().readText())
    return file.inputStream().fd
}
fun main() {
    val ret = (1.. 1048571).asSequence().map{ makeAndReadFile(it) }.toList()
    // val ret = (1.. 1048571).asSequence().map{ makeAndReadFile(it) }.forEach { }
}
```
不知道JDK 和 JVM 针对文件句柄管理都做了什么优化，kotlin 的demo 在成功创建读取了百万文件后， 才开始报错.

上面所有问题例子的模式是:

    (lazy things) . ( intermediate op 1) . ( intermediate op 2) . (.....) . (terminal op that accumulating things)

解决类似资源聚集引起的问题，需要把最后一步变成 (terminal op that consuming one thing)

模式本身不是问题，有时需要这种模式来尽量最大化某些资源的使用，[这篇blog中描述的情形](https://ntietz.com/blog/rusts-iterators-optimize-footgun/), 从one pass 到 multi pass . 

