+++
title = "playing around with text processing in elixir"
date = 2024-10-30

[taxonomies]
tags = ["elixir", "string", "ets table"]
+++

最近读到了一篇关于[Elixir String Processing Optimization](https://blog.jola.dev/posts/elixir-string-processing-optimization) 的文章， 借着这个机会开始动手写elixir 代码，熟悉一下elixir标准库相关api的使用.


这是作者呈现的第一段代码:
```elixir
IO.stream(:stdio, :line)
|> Stream.flat_map(&String.split/1)
|> Enum.reduce(%{}, fn x, acc -> Map.update(acc, x, 1, &(&1 + 1)) end)
|> Enum.sort(fn {_, a}, {_, b} -> b < a end)
|> Enum.each(fn {word, count} ->
  IO.puts(String.pad_leading(Integer.to_string(count), 8) <> " " <> word)
end)
```
的确是短小精炼，流畅优雅, 不过速度很慢， 在我的机器上运行了1372.10s, 作者也解释了相关的原因，我也学习到了一些elixir 相关知识.

奇怪的是作者呈现的最后一段elixir code, 使用第三方库flow, 还是很慢， 在我机器上运行了1321.68s, 本来期待有很大改进的， 在阅读了一会儿[Flow的这部分文档](https://hexdocs.pm/flow/Flow.html#module-performance-discussions)后， 我测试了这段代码:
```elixir
parent = self()
pattern = :binary.compile_pattern([" ", "\n"])

System.argv()
|> Enum.at(0, "../words-llvm8.0.txt")
|> File.stream!(read_ahead: 1_024_000)
|> Flow.from_enumerable()
|> Flow.flat_map(&String.split(&1, pattern, trim: true))
|> Flow.partition(min_demand: 1_000_000, max_demand: 3_468_009)
|> Flow.reduce(fn -> :ets.new(:words, []) end, fn word, ets ->
  :ets.update_counter(ets, word, {2, 1}, {word, 0})
  ets
end)
|> Flow.on_trigger(fn ets ->
  :ets.give_away(ets, parent, [])
  {[ets], :new_reduce_state_which_wont_be_used}
end)
|> Enum.flat_map(&:ets.tab2list/1)
|> Enum.sort(fn {_, a}, {_, b} -> b < a end)
|> Enum.map(fn {word, count} ->
  [Integer.to_string(count), " ", word, "\n"]
end)
|> IO.binwrite()
```
使用hyperfine的cold cache benchmark 方法: 
```bash
hyperfine --prepare 'sync; echo 3 | sudo tee /proc/sys/vm/drop_caches'  'some command'
```
结果是:
```bash
Benchmark 1: mix run lib/words_flow.ex
  Time (mean ± σ):      5.749 s ±  0.105 s    [User: 40.729 s, System: 3.827 s]
  Range (min … max):    5.591 s …  5.897 s    10 runs
```
通过调整参数min_demand, max_demand 可以控制stage 之间的流动数据的数目, 这里max_demand 我选择了(the number of words in this doc) / (the number of virtual cores you have).

如果只使用标准库Task模块的async_stream， 做并行任务处理，效果会是如何？ 带着这个问题，开启了一轮试验.

wordfreq_oneline_task.exs: 
```elixir
defmodule WordFreq do
  def start_count(fname) do
    table = :ets.new(:words, [:public, {:write_concurrency, true}])
    pattern = :binary.compile_pattern([" ", "\n"])

    count_per_line = fn line ->
      line
      |> String.split(pattern, trim: true)
      |> Enum.each(&:ets.update_counter(table, &1, {2, 1}, {&1, 0}))
    end

    File.stream!(fname, read_ahead: 1_024_000)
    |> Task.async_stream(count_per_line, ordered: false, on_timeout: :exit, timeout: :infinity)
    |> Stream.run()

    :ets.tab2list(table)
    |> Enum.sort(fn {_, a}, {_, b} -> a > b end)
    |> Enum.map(fn {word, count} ->
      [Integer.to_string(count), " ", word, "\n"]
    end)
    |> IO.binwrite()
  end
end

System.argv()
|> Enum.at(0, "./words-llvm8.0.txt")
|> WordFreq.start_count()
```
每个beam process 以一行的粒度接收任务处理， 结果是:
```bash
Benchmark 1: elixir wordfreq_oneline_task.exs
  Time (mean ± σ):     16.991 s ±  0.235 s    [User: 35.676 s, System: 1.014 s]
  Range (min … max):   16.741 s … 17.393 s    10 runs
```
wordfreq_multilines_task.exs:
```elixir
defmodule WordFreq do
  def start_count(fname) do
    table = :ets.new(:words, [:public, {:write_concurrency, true}])
    pattern = :binary.compile_pattern([" ", "\n"])

    count_multi_lines = fn lines ->
      lines
      |> Stream.flat_map(&String.split(&1, pattern, trim: true))
      |> Enum.each(&:ets.update_counter(table, &1, {2, 1}, {&1, 0}))
    end

    File.stream!(fname, read_ahead: 1_024_000)
    |> Stream.chunk_every(151_589)
    |> Task.async_stream(count_multi_lines, ordered: false, on_timeout: :exit, timeout: :infinity)
    |> Stream.run()

    :ets.tab2list(table)
    |> Enum.sort(fn {_, a}, {_, b} -> a > b end)
    |> Enum.map(fn {word, count} ->
      [Integer.to_string(count), " ", word, "\n"]
    end)
    |> IO.binwrite()
  end
end

System.argv()
|> Enum.at(0, "./words-llvm8.0.txt")
|> WordFreq.start_count()
```
每个beam process 以多行粒度接收任务, 这里chunk_every的参数 = (lines of this doc) / (the number of virtual cores you have). bench 结果:
```bash
Benchmark 1: elixir wordfreq_multilines_task.exs
  Time (mean ± σ):      5.439 s ±  0.051 s    [User: 15.974 s, System: 1.053 s]
  Range (min … max):    5.367 s …  5.529 s    10 runs
```

wordfreq_chunkread.exs:
```elixir
defmodule WordFreq do
  defp line_stream_from_file(path, chunk_size) do
    path
    |> File.stream!(chunk_size, read_ahead: chunk_size)
    |> bin_stream_to_line_stream()
  end

  defp line_stream_from_stdin(chunk_size) do
    IO.binstream(:stdio, chunk_size) |> bin_stream_to_line_stream()
  end

  defp bin_stream_to_line_stream(bin_stream) do
    start_fn = fn -> "" end

    pattern = :binary.compile_pattern("\n")

    reducer_fn = fn chunk, acc ->
      # IO.inspect(chunk, label: "chunk", limit: :infinity)
      [r | lines] =
        (acc <> chunk)
        |> String.split(pattern)
        # |> IO.inspect(label: "list", limit: :infinity)
        |> Enum.reverse()

      {lines, r}
    end

    last_fn = fn acc ->
      {[acc], acc}
    end

    after_fn = fn acc ->
      acc
    end

    Stream.transform(bin_stream, start_fn, reducer_fn, last_fn, after_fn)
  end

  def start_count(fname) do
    table = :ets.new(:words, [:public, {:write_concurrency, true}])
    pattern = :binary.compile_pattern(" ")

    count_per_chunk = fn lines ->
      lines
      |> Stream.flat_map(&String.split(&1, pattern, trim: true))
      |> Enum.each(&:ets.update_counter(table, &1, {2, 1}, {&1, 0}))
    end

    line_stream_from_file(fname, 1_024_000)
    |> Stream.chunk_every(151_589)
    |> Task.async_stream(count_per_chunk, ordered: false, on_timeout: :exit, timeout: :infinity)
    |> Stream.run()

    :ets.tab2list(table)
    |> Enum.sort(fn {_, a}, {_, b} -> a > b end)
    |> Enum.map(fn {word, count} ->
      [Integer.to_string(count), " ", word, "\n"]
    end)
    |> IO.binwrite()
  end
end

System.argv()
|> Enum.at(0, "./words-llvm8.0.txt")
|> WordFreq.start_count()
```

之前一直使用File.stream! 函数递送文本行流, 这里开始切换使用binary chunk 粒度读取文件, 但是这种方式可能会出现单词撕裂的情况，所以bin_stream_to_line_stream 函数会把binary chunk stream转换成line stream.  bin_stream_to_line_stream 使用强大的Stream.transform/5 函数完成此目的，
主要是reducer_fn 函数来实现， 每递送一个binary chunk 过来， 尝试分割，分割结果的最后一行可能是一个不完整的行，需要留下来，与下一次递送过来的binary chunk 合并继续分割. bench 结果:
```bash
Benchmark 1: elixir wordfreq_chunkread.exs
  Time (mean ± σ):      3.683 s ±  0.087 s    [User: 17.650 s, System: 1.049 s]
  Range (min … max):    3.545 s …  3.851 s    10 runs
```

wordfreq_chunkread_multiets.exs:
```elixir
defmodule WordFreq do
  defp line_stream_from_file(path, chunk_size) do
    path
    |> File.stream!(chunk_size, read_ahead: chunk_size)
    |> bin_stream_to_line_stream()
  end

  defp bin_stream_to_line_stream(bin_stream) do
    start_fn = fn -> "" end

    pattern = :binary.compile_pattern("\n")

    reducer_fn = fn chunk, acc ->
      [r | lines] =
        (acc <> chunk)
        |> String.split(pattern)
        |> Enum.reverse()

      {lines, r}
    end

    last_fn = fn acc ->
      {[acc], acc}
    end

    after_fn = fn acc ->
      acc
    end

    Stream.transform(bin_stream, start_fn, reducer_fn, last_fn, after_fn)
  end

  def start_count(fname) do
    pattern = :binary.compile_pattern(" ")
    table = :ets.new(:words, [])
    parent = self()

    count_per_chunk = fn lines ->
      subtable = :ets.new(:words, [])

      lines
      |> Stream.flat_map(&String.split(&1, pattern, trim: true))
      |> Enum.each(&:ets.update_counter(subtable, &1, {2, 1}, {&1, 0}))

      :ets.give_away(subtable, parent, [])
      subtable
    end

    {_, acc} =
      line_stream_from_file(fname, 1_024_000)
      |> Stream.chunk_every(151_589)
      |> Task.async_stream(count_per_chunk, ordered: false, on_timeout: :exit, timeout: :infinity)
      # |> Enum.flat_map(fn {:ok, subtable} ->
      #  :ets.tab2list(subtable)
      # end)
      # |> Enum.reduce(table, fn {word, count}, acc ->
      #  :ets.update_counter(acc, word, {2, count}, {word, 0})
      #  acc
      # end)

      |> Enum.flat_map_reduce(table, fn {:ok, subtable}, acc ->
        :ets.tab2list(subtable)
        |> Enum.each(fn {word, count} -> :ets.update_counter(acc, word, {2, count}, {word, 0}) end)

        {[], acc}
      end)

    acc
    |> :ets.tab2list()
    |> Enum.sort(fn {_, a}, {_, b} -> a > b end)
    |> Enum.map(fn {word, count} ->
      [Integer.to_string(count), " ", word, "\n"]
    end)
    |> IO.binwrite()
  end
end

System.argv()
|> Enum.at(0, "./words-llvm8.0.txt")
|> WordFreq.start_count()
```
之前代码是每个beam process 共享一个ets table 来采集统计结果, 现在是每个process 单独一个table， 然后在最后阶段，合并成一个table, bench 结果:
```bash
Benchmark 1: elixir wordfreq_chunkread_multiets.exs
  Time (mean ± σ):      3.310 s ±  0.045 s    [User: 14.493 s, System: 1.047 s]
  Range (min … max):    3.245 s …  3.404 s    10 runs
```
在粗略读了部分[erlang ets table 的文档](https://www.erlang.org/doc/apps/stdlib/ets.html#)后， 发现有ordered_set 风格的ets table , 能不能使用ordered_set 排序,替换Enum.sort ? 

wordfreq_chunkread_multiets_sort.exs:

```elixir
defmodule WordFreq do
  defp line_stream_from_file(path, chunk_size) do
    path
    |> File.stream!(chunk_size, read_ahead: chunk_size)
    |> bin_stream_to_line_stream()
  end

  defp bin_stream_to_line_stream(bin_stream) do
    start_fn = fn -> "" end

    pattern = :binary.compile_pattern("\n")

    reducer_fn = fn chunk, acc ->
      [r | lines] =
        (acc <> chunk)
        |> String.split(pattern)
        |> Enum.reverse()

      {lines, r}
    end

    last_fn = fn acc ->
      {[acc], acc}
    end

    after_fn = fn acc ->
      acc
    end

    Stream.transform(bin_stream, start_fn, reducer_fn, last_fn, after_fn)
  end

  def start_count(fname) do
    pattern = :binary.compile_pattern(" ")
    table = :ets.new(:words, [])
    parent = self()

    count_per_chunk = fn lines ->
      subtable = :ets.new(:words, [])

      lines
      |> Stream.flat_map(&String.split(&1, pattern, trim: true))
      |> Enum.each(&:ets.update_counter(subtable, &1, {2, 1}, {&1, 0}))

      :ets.give_away(subtable, parent, [])
      subtable
    end

    {_, acc} =
      line_stream_from_file(fname, 1_024_000)
      |> Stream.chunk_every(151_589)
      |> Task.async_stream(count_per_chunk, ordered: false, on_timeout: :exit, timeout: :infinity)
      |> Enum.flat_map_reduce(table, fn {:ok, subtable}, acc ->
        :ets.tab2list(subtable)
        |> Enum.each(fn {word, count} -> :ets.update_counter(acc, word, {2, count}, {word, 0}) end)

        {[], acc}
      end)

    acc
    |> :ets.tab2list()
    |> Enum.group_by(&elem(&1, 1), &elem(&1, 0))
    |> Enum.reduce(:ets.new(:sort_arena, [:ordered_set]), fn {count, wordlist}, acc ->
      :ets.update_element(acc, count, {2, wordlist}, {0, []})
      acc
    end)
    |> :ets.select_reverse([{{:"$1", :"$2"}, [], [{{:"$1", :"$2"}}]}])
    |> Enum.map(fn {num, words} ->
      num_str = Integer.to_string(num)

      for word <- words do
        [num_str, " ", word, "\n"]
      end
    end)
    |> IO.binwrite()
  end
end

System.argv()
|> Enum.at(0, "./words-llvm8.0.txt")
|> WordFreq.start_count()
```
bench 结果:
```bash
Benchmark 1: elixir  wordfreq_chunkread_multiets_sort.exs
  Time (mean ± σ):      3.266 s ±  0.058 s    [User: 14.800 s, System: 1.027 s]
  Range (min … max):    3.187 s …  3.379 s    10 runs
```
稍微有点改进， 这段使用ordered_set 排序code，刚开始我是这样写的:
```elixir
acc
|> :ets.tab2list()
|> Enum.reduce(sort_arena, fn {word, count}, acc ->
  word_list =
    :ets.select(acc, [{{:"$1", :"$2"}, [{:==, :"$1", count}], [:"$2"]}])
    |> Enum.flat_map(fn e -> e end)

  :ets.update_element(acc, count, {2, [word | word_list]}, {0, []})
  acc
end)
|> :ets.select_reverse([{{:"$1", :"$2"}, [], [{{:"$1", :"$2"}}]}])
|> Enum.map(fn {num, words} ->
  num_str = Integer.to_string(num)

  for word <- words do
    [num_str, " ", word, "\n"]
  end
end)
|> IO.binwrite()
```
测试之后，发现慢的惊人, 大概二十多分钟, 不知道为何， 直觉猜测是频繁地查询，更新table ，产生了大量临时copy ? ets doc 这样说:

> In the current implementation, every object insert and look-up operation results in a copy of the object.

也许我该学习怎么使用cprof, eprof, fprof 或者 [eflambe](https://github.com/Stratus3D/eflambe). 

PS  ets table 查询语法比较怪异，有些类似mongodb 












