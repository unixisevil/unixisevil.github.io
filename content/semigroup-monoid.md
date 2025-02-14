+++
title = "semigroup and monoid in rust"
date = 2025-02-14

[taxonomies]
tags = ["semigroup", "monoid", "rust"]
+++

前段时间看了看Haskell, 作为纯函数式编程语言, Haskell的根基几乎是用数学概念来构建, 比如[Semigroup](https://en.wikipedia.org/wiki/Semigroup), [Monoid](https://en.wikipedia.org/wiki/Monoid), 从这些数学定义看，这些概念有些抽象，不过从代码角度看，还是不难理解的.

Semigroup 和 Monoid 在Haskell 中被定义为type class(描述一类type 应该具备的行为, 类似rust的trait): 
```haskell
class Semigroup a where
  (<>) :: a -> a -> a

class Semigroup a => Monoid a where
  mempty :: a
  mappend :: a -> a -> a
  mconcat :: [a] -> a
```
Semigroup 的核心行为是combine 运算符 (<>) :: Semigroup a => a -> a -> a,  combine 是二元运算, 其接收两个同类型的东西, 输出同样的东西, 只要是符合结合律的二元操作都是合法的combine 实现.  比如Integer 的 + 运算:
```haskell
instance Semigroup Integer where
    (<>) x y = x + y
```
Monoid 是Semigroup 的拓展或者说是子类,  Monoid 具备combine 行为,  加上另一条行为要求: 实现Monoid 的type必须定义其identity 元素.
identity 元素的意思是 x <> id = x ,  id <> x  = x ,  任何x 与 identity 以任何order 的组合，结果都是返回x. 比如加法运算中的0，x + 0= 0 + x = 0,  乘法运算中的1, x * 1 = 1 * x =  x. 

据说是历史原因，Monoid 先于 Semigroup 在标准库中被实现， 本来可以更简洁地被定义成:
```haskell
class Semigroup a => Monoid a where
    identity :: a
```

rust库[frunk](https://crates.io/crates/frunk) 定义了Semigroup trait 和 Monoid trait. 以下是一些Semigroup 和 Monoid 的代码例子.

比如可以定义颜色是Semigroup 的实现: 
```rust
#[derive(Debug, PartialEq, Eq, Copy, Clone)]
enum Color {
    Red,
    Yellow,
    Blue,
    Green,
    Purple,
    Orange,
    Brown,
}

impl Semigroup for Color {
    fn combine(&self, other: &Self) -> Self {
        match (self, other) {
            (Self::Red, Self::Blue) => Self::Purple,
            (Self::Blue, Self::Red) => Self::Purple,
            (Self::Yellow, Self::Blue) => Self::Green,
            (Self::Blue, Self::Yellow) => Self::Green,
            (Self::Yellow, Self::Red) => Self::Orange,
            (Self::Red, Self::Yellow) => Self::Orange,
            (a, b) if a == b => *a,
            (a, b) => {
                let rbp = [Self::Red, Self::Blue, Self::Purple];
                let byg = [Self::Blue, Self::Yellow, Self::Green];
                let ryo = [Self::Red, Self::Yellow, Self::Orange];
                if rbp.contains(a) && rbp.contains(b) {
                    Self::Purple
                } else if byg.contains(a) && byg.contains(b) {
                    Self::Green
                } else if ryo.contains(a) && ryo.contains(b) {
                    Self::Orange
                } else {
                    Self::Brown
                }
            }
        }
    }
}
```
Color 类型的combine 实现中，定义不同的颜色如何混合形成另一种颜色.

```rust
type Events = Vec<String>;
type Probs = Vec<f64>;

#[derive(Clone)]
struct Ptable {
    events: Events,
    probs: Probs,
}

impl Semigroup for Ptable {
    fn combine(&self, other: &Self) -> Self {
        match (self, other) {
            (a, Ptable { events, probs }) if events.is_empty() && probs.is_empty() => a.clone(),
            (Ptable { events, probs }, b) if events.is_empty() && probs.is_empty() => b.clone(),
            (
                Ptable {
                    events: ea,
                    probs: pa,
                },
                Ptable {
                    events: eb,
                    probs: pb,
                },
            ) => {
                let em = combine_events(ea, eb);
                let pm = combine_probs(pa, pb);
                Ptable::new(em, pm)
            }
        }
    }
}

impl Monoid for Ptable {
    fn empty() -> Self {
        Ptable::new(vec![], vec![])
    }
}
```
事件发生的概率可以是Monoid，A 事件发生的概率是多少， B 事件的概率是多少， A 与 B 组合发生的概率是多少， 比如硬币连抛三次的事件的概率是多少.

```rust
#[derive(Clone)]
struct TimeSeries {
    ts: Vec<i32>,
    vs: Vec<Option<f64>>,
}

fn combine_time_series(a: &TimeSeries, b: &TimeSeries) -> TimeSeries {
    match (a, b) {
        (TimeSeries { ts, vs }, b) if ts.is_empty() && vs.is_empty() => b.clone(),
        (a, TimeSeries { ts, vs }) if ts.is_empty() && vs.is_empty() => a.clone(),
        (TimeSeries { ts: t1, vs: v1 }, TimeSeries { ts: t2, vs: v2 }) => {
            let mt = t1.iter().chain(t2);
            let min = mt.clone().min().copied().unwrap();
            let max = mt.max().unwrap();
            let complete_times = min..max + 1;
            let mut acc = BTreeMap::<i32, f64>::default();

            let insert_tv_pair = |map: &mut BTreeMap<_, _>, tv_pair: (&_, &_)| match tv_pair {
                (_, None) => {}
                (t, Some(v)) => {
                    map.insert(*t, *v);
                }
            };
            let mut acc = t1.iter().zip(v1.iter()).fold(&mut acc, |acc, e| {
                insert_tv_pair(acc, e);
                acc
            });
            let acc = t2.iter().zip(v2.iter()).fold(&mut acc, |acc, e| {
                insert_tv_pair(acc, e);
                acc
            });
            let final_vs: Vec<Option<f64>> = complete_times
                .clone()
                .map(|t| acc.get(&t).copied())
                .collect();

            TimeSeries {
                ts: complete_times.collect::<Vec<i32>>(),
                vs: final_vs,
            }
        }
    }
}

impl Semigroup for TimeSeries {
    fn combine(&self, other: &Self) -> Self {
        combine_time_series(self, other)
    }
}

impl Monoid for TimeSeries {
    fn empty() -> Self {
        Self {
            ts: vec![],
            vs: vec![],
        }
    }
}
```
时间序列可以是一种Monoid, 某个逻辑时间点上的值可能有缺失, 如何合并不同的time series， 时间序列类型上可以定义一些常见的操作.

[full source code](https://gist.github.com/unixisevil/badc8445d7a165d9abb8ec54147491e8)
