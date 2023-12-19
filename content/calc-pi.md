+++
title = "計算圓周率"
date = 2023-06-11

[taxonomies]
tags = ["rust", "pi", "chudnovsky"]
+++

计算pi的公式有很多，[chudnovsky](https://en.wikipedia.org/wiki/Chudnovsky_algorithm) 是其中比較高效的一個, 據wikipedia說世界記錄保持者基本都是使用chudnovsky
```python
#!/usr/bin/env python3

from decimal import Decimal, getcontext
import sys

if len(sys.argv) != 2:
    print(f"usage: {sys.argv[0]} 'number of digits'")
    raise SystemExit(1)

try:
    digits = int(sys.argv[1])
    if digits < 0:
        digits = 10
except ValueError:
    print(f"usage: {sys.argv[0]} 'number of digits'")
    raise SystemExit(2)

getcontext().prec = digits + 3
#print(f"prec = {digits + 3}")

q = 0
sum = Decimal(0)
kq = Decimal(-6)
lq = Decimal(13591409)
xq =  mq = Decimal(1)


while q < digits+2:
    q += 1
    sum = sum + (mq * lq / xq)
    kq = kq + 12
    mq = mq * ((kq ** 3 - 16*kq) / (q ** 3))
    lq = lq + 545140134
    xq = xq * -262537412640768000

#print(f'sum = {sum}')


C = 426880 * Decimal(10005).sqrt()
#print(f'C = {C}')

print(f'pi = {str(C / sum)[:-2]}')
```
上述python代碼實現了wikipedia 描述的迭代計算方法, q 初始值0, 每輪迭代向前推進計算kq, mq, lq, xq 等各個項和累計值 sum, 循環多迭代了兩個數字, 然後再扔掉失真的末尾2數字

相應的rust的實現:
```rust
use std::env;
use rug::{
    Float, Integer,
    ops::{Pow, CompleteRound},
};

fn main() {
    let r = env::args()
        .skip(1)
        .take(1)
        .next()
        .map_or(8u32, |a| a.parse().map_or(8u32, |v| v));

    println!("{}", pi(r));
}

fn pi(digits: u32) -> String {
    let bits_per_digit = 3.32192809488736234787; //log(10,2)
    let prec = ((digits + 3) as f64 * bits_per_digit) as u32;
    //let  digits_per_iteration = 14.1816474627254776555;
    //let limit = (digits /  digits_per_iteration) + 1;
    let limit = digits + 2;
    //dbg!(prec);
    let mut q = Integer::from(0);
    let mut sum = Float::with_val(prec, 0);
    let mut kq = Float::with_val(prec, -6);
    let mut lq = Float::with_val(prec, 13591409);
    let mut xq = Float::with_val(prec, 1);
    let mut mq = Float::with_val(prec, 1);

    while q < limit {
        q += 1;
        sum += (&mq * &lq).complete(prec) / &xq;
        kq += 12;
        mq *= (kq.clone().pow(3) - (&kq * 16u8).complete(prec)) / q.clone().pow(3);
        lq += 545140134;
        xq *= -262537412640768000i64;
    }

    let c = 426880 * Float::with_val(prec, 10005).sqrt();

    let r = Float::with_val(prec, c / sum);
    let mut s = r.to_string_radix(10, Some((1 + digits + 2) as usize));
    s.truncate(1 + digits as usize);
    s
}
```
與python的Decimal庫使用10進制不同的是，rust的rug庫使用而二進制表達精度, 平均每個十進制數字需要用log(10,2) 個bit; 由於rust 默認的move語義，需要在一些地方插入clone 

wikipedia給出的公式在計算更大的數字精度，比如10 ** 5 以上速度比較慢, 發現有一種使用[binary splitting加速chudnovsky的方法](https://www.craig-wood.com/nick/articles/pi-chudnovsky), 以下是相應的rust實現:
```rust
use std::env;
use rug::{
    ops::{Pow, NegAssign},
    Integer,
    Assign, Complete
};

const C: u64 = 640320;
const C3_OVER_24: u64 = C.pow(3) / 24;

fn pi_bs(digits: u32) -> Integer {
    let digits_per_term = (C3_OVER_24/6/2/6).ilog10();

    let n = digits / digits_per_term  + 1;
    let (_p,q,t) = bs(0.into(), n.into());

    let one_squared  = Integer::from(10).pow(2*digits);
    let sqrt_c: Integer = (one_squared * 10005u32).sqrt();

    (q * 426880 * sqrt_c) / t
}

fn bs(a: Integer, b: Integer) -> (Integer, Integer, Integer) {
    let mut pab = Integer::new();
    let mut qab = Integer::new();
    let mut tab = Integer::new();
    if (&b - &a).complete() == 1 {
        let tmp = (545140134i32 * &a).complete()  + 13591409;
        if a == 0 {
            pab.assign(1);
            qab.assign(1);
        } else {
            let a65 = (&a * 6i32).complete() - 5;
            let a61 = (&a * 6i32).complete() - 1;
            let a21 = (&a * 2i32).complete()  -1 ;
            let ap3 = (&a).pow(3u32).complete();
            pab.assign(a65 * a61 * a21);
            qab.assign(ap3 * C3_OVER_24);
        }
        tab.assign(tmp * &pab);
        if a.is_odd() {
            tab.neg_assign();
        }
    } else {
        let m = (&a + &b).complete() / 2i32 ;
        let mc = m.clone();
        let (pam, qam, tam) = bs(a, m);
        let (pmb, qmb, tmb) = bs(mc, b);
        pab.assign(&pam * &pmb);
        qab.assign(&qam * &qmb);
        tab.assign(qmb * tam  +  pam * tmb);
    }
    (pab, qab, tab)
}

fn main() {
    let r = env::args()
        .skip(1)
        .take(1)
        .next()
        .map_or(8u32, |a| a.parse().map_or(8u32, |v| v));

    println!("{}", pi_bs(r));
}
```
binary splitting 結構上是一種divide-and-conquer機制, craig wood說,應用在chudnovsky公式上效果就是:
>  "What it does is convert the sum of the individual fractions into one giant fraction. 
This means that you only do one divide at the end of the calculation which speeds things up greatly, because division is slow compared to multiplication."  

簡單測試計算10 ** 5個數字，瞬間打印出結果, 測試計算10 ** 9個數字使用時間:

<img src="/imgs/2023-06-11-14-14-33.png">

大概用時34分鐘左右,  divide-and-conquer結構的計算往往比較容易並行處理，遞歸函數bs 可以同時進行:
```rust
use std::env;
use rug::{
    ops::{Pow, NegAssign},
    Integer,
    Assign, Complete
};

const C: u64 = 640320;
const C3_OVER_24: u64 = C.pow(3) / 24;

fn pi_bs(digits: u32) -> Integer {
    let digits_per_term = (C3_OVER_24/6/2/6).ilog10();

    let n = digits / digits_per_term  + 1;
    let bz = Integer::from(0);
    let bn = Integer::from(n);
    let (_p,q,t) = bs(&bz,  &bn);

    let one_squared  = Integer::from(10).pow(2*digits);
    let sqrt_c: Integer = (one_squared * 10005u32).sqrt();

    (q * 426880 * sqrt_c) / t
}

fn bs(a: &Integer, b: &Integer) -> (Integer, Integer, Integer) {
    let mut pab = Integer::new();
    let mut qab = Integer::new();
    let mut tab = Integer::new();
    if (b - a).complete() == 1 {
        let tmp = (545140134i32 * a).complete()  + 13591409;
        if a.cmp0().is_eq() {
            pab.assign(1);
            qab.assign(1);
        } else {
            let a65 = (a * 6i32).complete() - 5;
            let a61 = (a * 6i32).complete() - 1;
            let a21 = (a * 2i32).complete()  -1 ;
            let ap3 = a.pow(3u32).complete();
            pab.assign(a65 * a61 * a21);
            qab.assign(ap3 * C3_OVER_24);
        }
        tab.assign(tmp * &pab);
        if a.is_odd() {
            tab.neg_assign();
        }
    } else {
        let m = (a + b).complete() / 2i32 ;
        let ((pam, qam, tam), (pmb, qmb, tmb))  =  rayon::join(|| bs(a, &m),  || bs(&m, b));
        pab.assign(&pam * &pmb);
        qab.assign(&qam * &qmb);
        tab.assign(qmb * tam  +  pam * tmb);
    }
    (pab, qab, tab)
}

fn main() {
    let r = env::args()
        .skip(1)
        .take(1)
        .next()
        .map_or(8u32, |a| a.parse().map_or(8u32, |v| v));

    println!("{}", pi_bs(r));
}
```
使用[rayon](https://github.com/rayon-rs/rayon)幾乎無縫銜接並行處理, rayon 內部使用work-stealing調度線程，默認構造了一個與cpu邏輯核心數目等同的線程池, 計算10 ** 9個數字使用時間, 大致用時18分種:

<img src="/imgs/2023-06-11-13-16-21.png"> 
