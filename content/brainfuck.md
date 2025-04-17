+++
title = "The unexpected slowness of the ETS table"
date = 2025-04-17

[taxonomies]
tags = ["elixir", "ets table", "brainfuck", "map", "bit string", "process dict"]
+++

最近发现ets table 并不是预期中那么快, 一些immutable data structure 像map, bit string 都有可能快过ets table, 比如在实现brainfuck interpreter 的场景.

运行其中一个例子, 打印sierpinski 三角形:

```bash
./bf_ets_storage.exs --example sierpinski
```
<img src="/imgs/slow_ets.gif">

```bash
./bf_bin_storage.exs --example sierpinski
```
<img src="/imgs/fast_bin.gif">

观察eprofile的结果, 过滤其中ets 相关:
```bash
****** Process <0.95.0>    -- 100.00 % of profiled time ***
FUNCTION                                              CALLS        %       TIME  [uS / CALLS]
--------                                              -----  -------       ----  [----------]
ets:new/2                                                 1     0.00          7  [      7.00]
ets:insert/2                                              1     0.00       4156  [   4156.00]
ets:update_counter/3                                 126577     0.04      39000  [      0.31]
ets:select/2                                          73265    99.67  102147914  [   1394.23]
```
ets:select/2  调用十分频繁而且每次调用开销不低, 使用ets table 模拟 brainfuck 机器的内存, 如果在get, set, incr, decr 等读写memory cell api 处加上dbg()观察, 会存在一些连续对同一个cell 的读取, 如果能cache 读取调用, 可能会有改善.

```bash
[bf_ets_storage.exs:29: BrainFuck.EtsCell.incr/2]
idx #=> 0

[bf_ets_storage.exs:29: BrainFuck.EtsCell.incr/2]
idx #=> 0

[bf_ets_storage.exs:29: BrainFuck.EtsCell.incr/2]
idx #=> 0
....
....
```
另外存在上述连续对同一个cell 自增操作, 可以做合并做一次写入, 可能优化的解释器,编译器会利用一些重复的code模式做优化, 像[bfc](https://github.com/Wilfred/bfc), 还没看过.

写代码过程中发现elixir不支持递归匿名函数，搜索后发现发现一个[trick](https://dev.to/lucassperez/recursive-anonymous-function-in-elixir-pn3) 可以像这样写, 引入一个额外的参数来表达匿名函数自身: 
```elixir
find = fn location, unmatched_brackets, f ->
      cond do
        location < 0 or location >= byte_size(source_code) ->
          IO.puts("Error: could not find match for #{start_bracket} at #{start}.")
          start

        true ->
          case :binary.at(source_code, location) do
            ^end_bracket ->
              if unmatched_brackets == 0 do
                location
              else
                f.(location + direction, unmatched_brackets - 1, f)
              end

            ^start_bracket ->
              f.(location + direction, unmatched_brackets + 1, f)

            _ ->
              f.(location + direction, unmatched_brackets, f)
          end
      end
    end

    find.(location, unmatched_brackets, find)
```
评论区有人说这是[Y Combinator](https://matt.might.net/articles/implementation-of-recursive-fixed-point-y-combinator-in-javascript-for-memoization/), elixir 中保存递归调用中的状态, 可以使用[process dictionary](https://hexdocs.pm/elixir/Process.html#put/2).

使用map, bit string, process dictionary 来模拟brainfuck 的memory cell 都比使用ets table 要快.
[full source code](https://gist.github.com/unixisevil/e1d872a2af73c282b244c23ef4309818)
