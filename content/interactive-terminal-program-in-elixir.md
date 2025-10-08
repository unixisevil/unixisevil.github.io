+++
title = "interactive terminal code in elixir"
date = 2025-10-08

[taxonomies]
tags = ["elixir"]
+++

之前用elixir写一些与命令行终端交互的脚本时, 发现读取终端按键总是以回车键结束, 没找到合适的方法获取即时的按键反馈, 现在[erlang 28 引入了raw terminal mode](https://elixirforum.com/t/raw-terminal-mode-coming-to-otp-28/67491)可以方便地解决这个问题. 

以下elixir code 可以立即获取用户按键, 不需要敲击回车键:
```elixir
    :shell.start_interactive({:noshell, :raw})
    input_key = IO.getn("", 1)
    :shell.start_interactive({:noshell, :cooked})
```

比如可以利用这个特性来编写[交互式终端命令行程序less](https://gist.github.com/unixisevil/e16d02b10d5df27318ecf8c97bf29c90) 

打开小文本文件:
<img src="/imgs/myless-small.gif">

打开大文本文件:
<img src="/imgs/myless-big.gif">

