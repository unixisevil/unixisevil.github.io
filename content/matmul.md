+++
title = "矩阵乘法"
date = 2023-06-17

[taxonomies]
tags = ["rust", "python"]
+++

普通的[矩阵乘法](https://en.wikipedia.org/wiki/Matrix_multiplication_algorithm)实现上比较简单，我想练习一下rust的const generics:
```rust
use std::fmt::Debug;
use std::ops::{AddAssign, Index, IndexMut, Mul};
use std::vec::Vec;

struct Array2d<T, const R: usize, const C: usize>(Vec<T>);

impl<T, const R: usize, const C: usize> Array2d<T, R, C> {
    const fn rows(&self) -> usize {
        R
    }
    const fn cols(&self) -> usize {
        C
    }

    fn new() -> Self {
        assert!(R > 0, "row must be greater than 0");
        assert!(C > 0, "column must be greater than 0");
        Array2d(Vec::with_capacity(R * C))
    }

    fn from_vec(data: Vec<T>) -> Self {
        assert!(R > 0, "row must be greater than 0");
        assert!(C > 0, "column must be greater than 0");
        assert_eq!(R * C, data.len());
        Array2d(data)
    }

    fn fill(&mut self, elem: T)
    where
        T: Clone,
    {
        for i in 0..R * C {
            self.0.insert(i, elem.clone())
        }
    }
}

impl<T, const R: usize, const C: usize> Index<(usize, usize)> for Array2d<T, R, C> {
    type Output = T;
    fn index(&self, pos: (usize, usize)) -> &T {
        &self.0[pos.0 * C + pos.1]
    }
}

impl<T, const R: usize, const C: usize> IndexMut<(usize, usize)> for Array2d<T, R, C> {
    fn index_mut(&mut self, pos: (usize, usize)) -> &mut T {
        &mut self.0[pos.0 * C + pos.1]
    }
}

impl<T: Debug, const R: usize, const C: usize> Debug for Array2d<T, R, C> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let mut ret = String::new();
        for r in 0..self.rows() {
            for c in 0..self.cols() {
                let mut elem_str = format!("{:?}", self.0[r * C + c]);
                if c != self.cols() - 1 {
                    elem_str.push_str(", ");
                }
                ret.push_str(&elem_str);
            }
            ret.push('\n');
        }
        write!(f, "{}", ret)
    }
}

#[cfg(test)]
mod tests {
    use crate::matmul_normal;
    use crate::Array2d;

    #[test]
    fn init_test() {
        let mut a = Array2d::<i32, 3, 2>::new();
        a.fill(10);
        assert_eq!(a.0, vec![10; 6]);
        a[(2, 1)] = 100;
        assert_eq!(a[(2, 1)], 100);
    }

    #[test]
    fn mul_test() {
        let a = Array2d::<u32, 2, 4>::from_vec(vec![11, 12, 13, 14, 21, 22, 23, 24]);
        let b = Array2d::<u32, 4, 3>::from_vec(vec![12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1]);
        let mut c = Array2d::<u32, 2, 3>::new();
        c.fill(0);
        matmul_normal(a, b, &mut c);
        assert_eq!(c.0, vec![360, 310, 260, 660, 570, 480]);
    }
}

fn matmul_normal<
    T: Clone + Mul<Output = T> + AddAssign,
    const N: usize,
    const M: usize,
    const P: usize,
>(
    a: Array2d<T, N, M>,
    b: Array2d<T, M, P>,
    c: &mut Array2d<T, N, P>,
) {
    //let mut c = Array2d::<T, N, P>::new();
    //c.fill(T::default());
    for i in 0..a.rows() {
        for k in 0..a.cols() {
            for j in 0..b.cols() {
                c[(i, j)] += a[(i, k)].clone() * b[(k, j)].clone();
            }
        }
    }
    //c
}

#[allow(dead_code)]
const fn const_max(a: usize, b: usize) -> usize {
    [a, b][(a < b) as usize]
}

fn main() {
    use tinyrand::{Rand, StdRand};
    let mut rand = StdRand::default();

    const dim: usize = 4096;
    let mut a = Array2d::<u32, dim, dim>::new();
    let mut b = Array2d::<u32, dim, dim>::new();
    let mut c = Array2d::<u32, dim, dim>::new();
    a.fill(0u32);
    b.fill(0u32);
    c.fill(0u32);

    for i in 0..a.rows() {
        for j in 0..a.cols() {
            a[(i, j)] = rand.next_lim_u32(255);
        }
    }
    //println!("{:?}", a);
    for i in 0..b.rows() {
        for j in 0..b.cols() {
            b[(i, j)] = rand.next_lim_u32(255);
        }
    }
    //println!("{:?}", b);
    use std::time;
    let now = time::Instant::now();
    matmul_normal(a, b, &mut c);
    println!("took {:?}", now.elapsed());
    //println!("{:?}", c);
}
```
上面的代码使用了ikj访问模式，而不是其它的访问模式; 交换loop order 实际上对执行结果没有影响，但是影响执行速度.
不同的访问模式，引起的cpu cache-misses 有差异, 使用linux perf 可以观察这些效果.

ikj 访问模式:
```bash
 perf stat -e cycles,instructions,cache-references,cache-misses,branches,branch-misses,task-clock,faults,minor-faults,cs,migrations  ./ikj2d 
took 25.791189735s

 Performance counter stats for './ikj2d':

   109,579,244,480      cycles:u                         #    4.232 GHz                       
   243,790,279,104      instructions:u                   #    2.22  insn per cycle            
     9,020,603,100      cache-references:u               #  348.383 M/sec                     
     5,262,055,310      cache-misses:u                   #   58.334 % of all cache refs       
    17,750,421,273      branches:u                       #  685.536 M/sec                     
        16,806,625      branch-misses:u                  #    0.09% of all branches           
         25,892.77 msec task-clock:u                     #    0.999 CPUs utilized             
            49,231      faults:u                         #    1.901 K/sec                     
            49,228      minor-faults:u                   #    1.901 K/sec                     
                 0      cs:u                             #    0.000 /sec                      
                 0      migrations:u                     #    0.000 /sec
```
jki 访问模式:
```bash
perf stat -e cycles,instructions,cache-references,cache-misses,branches,branch-misses,task-clock,faults,minor-faults,cs,migrations  ./jki2d 
took 1111.596667316s

 Performance counter stats for './jki2d':

 4,737,745,200,115      cycles:u                         #    4.264 GHz                       
   826,026,955,156      instructions:u                   #    0.17  insn per cycle            
   256,529,760,719      cache-references:u               #  230.858 M/sec                     
   161,332,092,569      cache-misses:u                   #   62.890 % of all cache refs       
   206,477,635,100      branches:u                       #  185.815 M/sec                     
        16,879,224      branch-misses:u                  #    0.01% of all branches           
      1,111,202.94 msec task-clock:u                     #    1.000 CPUs utilized             
            49,232      faults:u                         #   44.305 /sec                      
            49,229      minor-faults:u                   #   44.302 /sec                      
                 0      cs:u                             #    0.000 /sec                      
                 0      migrations:u                     #    0.000 /sec
```

正常模式下rust compiler 并没有发射针对特定机器的优化指令, 可以通过在项目配置文件.cargo/config.toml中添加编译器选项:
```bash
[build]
rustflags = ["-C", "target-cpu=native"]
```
之前4096x4096 大小的输入大概 25,26 s , 现在变成13,14s

我想实现wikipedia上说的[二分切割的方法](https://en.wikipedia.org/wiki/Matrix_multiplication_algorithm#Non-square_matrices),需要能访问matrix局部视图的方法:
```rust
use std::fmt::Debug;
use std::ops::{Index, IndexMut};
use std::vec::Vec;

struct Array2d<T> {
    rows: usize,
    cols: usize,
    data: Vec<T>,
}

#[derive(Clone)]
struct Array2dView<'a, T> {
    orig: &'a Array2d<T>,
    pos: (usize, usize),
    dim: (usize, usize),
}

impl<'a, T> Array2dView<'a, T> {
    fn view_at(&self, off: (usize, usize), dim: (usize, usize)) -> Array2dView<T> {
        Array2dView {
            orig: self.orig,
            pos: (self.pos.0 + off.0, self.pos.1 + off.1),
            dim,
        }
    }
}

impl<'a, T> Index<(usize, usize)> for Array2dView<'a, T> {
    type Output = T;
    fn index(&self, pos: (usize, usize)) -> &T {
        &self.orig.data[(self.pos.0 + pos.0) * self.orig.cols + (self.pos.1 + pos.1)]
    }
}

impl<'a, T> From<Array2dMutView<'a, T>> for Array2dView<'a, T> {
    fn from(amv: Array2dMutView<'a, T>) -> Self {
        Array2dView {
            orig: &*amv.orig,
            pos: amv.pos,
            dim: amv.dim,
        }
    }
}

impl<'a, T: Debug> Debug for Array2dView<'a, T> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let mut ret = String::new();
        for r in 0..self.dim.0 {
            for c in 0..self.dim.1 {
                let mut elem_str = format!(
                    "{:?}",
                    self.orig.data[(self.pos.0 + r) * self.orig.cols + (self.pos.1 + c)]
                );
                if c != self.dim.1 - 1 {
                    elem_str.push_str(", ");
                }
                ret.push_str(&elem_str);
            }
            ret.push('\n');
        }
        write!(f, "{}", ret)
    }
}

struct Array2dMutView<'a, T> {
    orig: &'a mut Array2d<T>,
    pos: (usize, usize),
    dim: (usize, usize),
}

impl<'a, T> Array2dMutView<'a, T> {
    fn mut_view_at(&mut self, off: (usize, usize), dim: (usize, usize)) -> Array2dMutView<T> {
        Array2dMutView {
            orig: self.orig,
            pos: (self.pos.0 + off.0, self.pos.1 + off.1),
            dim,
        }
    }
}

impl<'a, T> Index<(usize, usize)> for Array2dMutView<'a, T> {
    type Output = T;
    fn index(&self, pos: (usize, usize)) -> &T {
        &self.orig.data[(self.pos.0 + pos.0) * self.orig.cols + (self.pos.1 + pos.1)]
    }
}

impl<'a, T> IndexMut<(usize, usize)> for Array2dMutView<'a, T> {
    fn index_mut(&mut self, pos: (usize, usize)) -> &mut T {
        &mut self.orig.data[(self.pos.0 + pos.0) * self.orig.cols + (self.pos.1 + pos.1)]
    }
}

impl<T: Default + Clone> Array2d<T> {
    fn new(rows: usize, cols: usize) -> Self {
        assert!(rows > 0, "row must be greater than 0");
        assert!(cols > 0, "column must be greater than 0");
        Array2d {
            rows,
            cols,
            data: vec![T::default(); rows * cols],
        }
    }

    fn from_vec(dim: (usize, usize), data: Vec<T>) -> Self {
        assert!(dim.0 > 0, "row must be greater than 0");
        assert!(dim.1 > 0, "column must be greater than 0");
        assert_eq!(dim.0 * dim.1, data.len());
        Array2d {
            rows: dim.0,
            cols: dim.1,
            data,
        }
    }

    fn shape(&self) -> (usize, usize) {
        (self.rows, self.cols)
    }

    fn fill(&mut self, elem: T) {
        for i in 0..self.rows * self.cols {
            self.data[i] = elem.clone()
        }
    }

    fn view_at(&self, off: (usize, usize), dim: (usize, usize)) -> Array2dView<T> {
        Array2dView {
            orig: self,
            pos: (off.0, off.1),
            dim,
        }
    }

    fn mut_view_at(&mut self, off: (usize, usize), dim: (usize, usize)) -> Array2dMutView<T> {
        Array2dMutView {
            orig: self,
            pos: (off.0, off.1),
            dim,
        }
    }

    fn view(&self) -> Array2dView<T> {
        Array2dView {
            orig: self,
            pos: (0, 0),
            dim: (self.rows, self.cols),
        }
    }

    fn mut_view(&mut self) -> Array2dMutView<T> {
        Array2dMutView {
            dim: (self.rows, self.cols),
            orig: self,
            pos: (0, 0),
        }
    }
}

impl<T> Index<(usize, usize)> for Array2d<T> {
    type Output = T;
    fn index(&self, pos: (usize, usize)) -> &T {
        &self.data[pos.0 * self.cols + pos.1]
    }
}

impl<T> IndexMut<(usize, usize)> for Array2d<T> {
    fn index_mut(&mut self, pos: (usize, usize)) -> &mut T {
        &mut self.data[pos.0 * self.cols + pos.1]
    }
}

impl<T: Debug> Debug for Array2d<T> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let mut ret = String::new();
        for r in 0..self.rows {
            for c in 0..self.cols {
                let mut elem_str = format!("{:?}", self.data[r * self.cols + c]);
                if c != self.cols - 1 {
                    elem_str.push_str(", ");
                }
                ret.push_str(&elem_str);
            }
            ret.push('\n');
        }
        write!(f, "{}", ret)
    }
}

impl Array2d<u32> {
    fn randint(&mut self, lim: u32) {
        use tinyrand::{Rand, StdRand};
        let mut rand = StdRand::default();

        for i in 0..self.rows {
            for j in 0..self.cols {
                self[(i, j)] = rand.next_lim_u32(lim);
            }
        }
    }

    fn transpose(&self) -> Array2d<u32> {
        let mut ret = Array2d::<u32>::new(self.cols, self.rows);
        for i in 0..self.rows {
            for j in 0..self.cols {
                ret.data[j * self.rows + i] = self.data[i * self.cols + j];
            }
        }
        ret
    }
}

fn matmul(a: Array2dView<u32>, b: Array2dView<u32>, mut c: Array2dMutView<u32>) {
    for i in 0..a.dim.0 {
        for k in 0..a.dim.1 {
            for j in 0..b.dim.1 {
                c[(i, j)] += a[(i, k)] * b[(k, j)];
            }
        }
    }
    //println!("func:\n{:?}",  <Array2dMutView<u32> as Into<Array2dView<u32>>>::into(c));
}

fn matadd(a: Array2dView<u32>, b: Array2dView<u32>, mut c: Array2dMutView<u32>) {
    for i in 0..a.dim.0 {
        for j in 0..a.dim.1 {
            c[(i, j)] = a[(i, j)] + b[(i, j)];
        }
    }
}

fn matmul_dac(a: Array2dView<u32>, b: Array2dView<u32>, mut c: Array2dMutView<u32>) {
    let (n, m, p) = (a.dim.0, a.dim.1, b.dim.1);
    let _square = n == m && m == p;
    let &max = [n, m, p].iter().max().unwrap();
    if max <= 181 {
        //c[(0,0)] += a[(0,0)] * b[(0,0)];
        matmul(a, b, c);
        return;
    }
    if max == m {
        let m2 = m / 2;
        let a1 = a.view_at((0, 0), (n, m2));
        let a2 = a.view_at((0, m2), (n, m - m2));
        let b1 = b.view_at((0, 0), (m2, p));
        let b2 = b.view_at((m2, 0), (m - m2, p));

        let mut c1 = Array2d::<u32>::new(n, p);
        let mut c2 = Array2d::<u32>::new(n, p);
        matmul_dac(a1, b1, c1.mut_view());
        matmul_dac(a2, b2, c2.mut_view());
        matadd(c1.view(), c2.view(), c);
    } else if max == n {
        let n2 = n / 2;
        let a1 = a.view_at((0, 0), (n2, m));
        let a2 = a.view_at((n2, 0), (n - n2, m));
        matmul_dac(a1, b.clone(), c.mut_view_at((0, 0), (n2, p)));
        matmul_dac(a2, b, c.mut_view_at((n2, 0), (n - n2, p)));
    } else {
        let p2 = p / 2;
        let b1 = b.view_at((0, 0), (m, p2));
        let b2 = b.view_at((0, p2), (m, p - p2));
        matmul_dac(a.clone(), b1, c.mut_view_at((0, 0), (n, p2)));
        matmul_dac(a, b2, c.mut_view_at((0, p2), (n, p - p2)));
    }
}

fn main() {
    let dim = 4096usize;
    let mut a = Array2d::<u32>::new(dim, dim);
    let mut b = Array2d::<u32>::new(dim, dim);
    let mut c1 = Array2d::<u32>::new(dim, dim);
    let mut c2 = Array2d::<u32>::new(dim, dim);
    a.randint(10);
    b.randint(10);
    //println!("{:?}", a.view());
    //println!("{:?}", b.view());
    use std::time;

    let now = time::Instant::now();
    matmul(a.view(), b.view(), c1.mut_view());
    println!("took {:?}", now.elapsed());

    let now = time::Instant::now();
    matmul_dac(a.view(), b.view(), c2.mut_view());
    println!("took {:?}", now.elapsed());
    assert_eq!(c1.data, c2.data);
}

#[cfg(test)]
mod tests {
    use crate::Array2d;
    use crate::{matmul, matmul_dac};

    #[test]
    fn init_test() {
        let mut a = Array2d::<i32>::new(3, 2);
        a.fill(10);
        assert_eq!(a.data, vec![10; 6]);
        a[(2, 1)] = 100;
        assert_eq!(a[(2, 1)], 100);
    }

    #[test]
    fn mul_test() {
        let a = Array2d::<u32>::from_vec((2, 4), vec![11, 12, 13, 14, 21, 22, 23, 24]);
        let b = Array2d::<u32>::from_vec((4, 3), vec![12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1]);
        let mut c = Array2d::<u32>::new(2, 3);
        matmul(a.view(), b.view(), c.mut_view());
        assert_eq!(c.data, vec![360, 310, 260, 660, 570, 480]);
    }

    #[test]
    fn matmul_dac_test() {
        let a = Array2d::<u32>::from_vec((2, 4), vec![11, 12, 13, 14, 21, 22, 23, 24]);
        let b = Array2d::<u32>::from_vec((4, 3), vec![12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1]);
        let mut c = Array2d::<u32>::new(2, 3);
        matmul_dac(a.view(), b.view(), c.mut_view());
        assert_eq!(c.data, vec![360, 310, 260, 660, 570, 480]);
    }

    #[test]
    fn view_test() {
        let a = Array2d::<u32>::from_vec(
            (4, 5),
            vec![5, 9, 5, 5, 6, 1, 7, 6, 7, 1, 8, 1, 1, 8, 8, 0, 3, 4, 0, 8],
        );
        let av = a.view_at((0, 2), (4, 3));
        assert_eq!(format!("{:?}", av), "5, 5, 6\n6, 7, 1\n1, 8, 8\n4, 0, 8\n");
        let bv = av.view_at((2, 1), (2, 2));
        assert_eq!(format!("{:?}", bv), "8, 8\n0, 8\n");
        let cv = a.view_at((0, 0), (4, 2));
        assert_eq!(format!("{:?}", cv), "5, 9\n1, 7\n8, 1\n0, 3\n");
        let dv = a.view_at((0, 0), (3, 5));
        let ev = a.view_at((3, 0), (1, 5));
        assert_eq!(format!("{:?}", dv), "5, 9, 5, 5, 6\n1, 7, 6, 7, 1\n8, 1, 1, 8, 8\n");
        assert_eq!(format!("{:?}", ev), "0, 3, 4, 0, 8\n");
    }
}
```
Array2dView , Array2dMutView 类型分别代表Array2d的只读视图和可写视图, 它们的view_at() 方法接收一个2d 坐标 和大小维度，产生一个subview, matmul_dac 尽量均匀地切割matrix, 递归到某个阀值(当前使用math.sqrt(LEVEL1_DCACHE_SIZE),没啥特别的，可以测试调整,wikipedia另外提到的, math.sqrt(LEVEL1_DCACHE_SIZE/3))，使用普通迭代方法

简单测试性能不升反降; Array2dView , Array2dMutView的定义不能与[rayon](https://github.com/rayon-rs/rayon)join 中closures 可以安全地Send 跨线程要求相容,  逻辑上matmul_dac 中需要触碰的Array2dMutView应该是不重叠的, 需要修改view 类型定义:
```rust
use std::fmt::Debug;
use std::ops::{Index, IndexMut};
use std::vec::Vec;

struct Array2d<T> {
    rows: usize,
    cols: usize,
    data: Vec<T>,
}

#[derive(Clone)]
struct Array2dView<T> {
    addr: *const T,
    orig_cols: usize,
    dim: (usize, usize),
}

unsafe impl<T: Send> Send for Array2dView<T> {}

impl<T> Array2dView<T> {
    fn view_at(&self, off: (usize, usize), dim: (usize, usize)) -> Array2dView<T> {
        Array2dView {
            addr: unsafe { self.addr.add(off.0 * self.orig_cols + off.1) },
            dim,
            orig_cols: self.orig_cols,
        }
    }
}

impl<T: Debug> Index<(usize, usize)> for Array2dView<T> {
    type Output = T;
    fn index(&self, pos: (usize, usize)) -> &T {
        unsafe { &*self.addr.add(pos.0 * self.orig_cols + pos.1) }
    }
}

impl<T: Debug> Debug for Array2dView<T> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let mut ret = String::new();
        for r in 0..self.dim.0 {
            for c in 0..self.dim.1 {
                let mut elem_str = format!("{:?}", self[(r, c)]);
                if c != self.dim.1 - 1 {
                    elem_str.push_str(", ");
                }
                ret.push_str(&elem_str);
            }
            ret.push('\n');
        }
        write!(f, "{}", ret)
    }
}

struct Array2dMutView<T> {
    addr: *mut T,
    orig_cols: usize,
    dim: (usize, usize),
}

unsafe impl<T: Send> Send for Array2dMutView<T> {}

impl<T> Array2dMutView<T> {
    fn mut_view_at(&self, off: (usize, usize), dim: (usize, usize)) -> Array2dMutView<T> {
        Array2dMutView {
            addr: unsafe { self.addr.add(off.0 * self.orig_cols + off.1) },
            dim,
            orig_cols: self.orig_cols,
        }
    }
}

impl<T> Index<(usize, usize)> for Array2dMutView<T> {
    type Output = T;
    fn index(&self, pos: (usize, usize)) -> &T {
        unsafe { &*self.addr.add(pos.0 * self.orig_cols + pos.1) }
    }
}

impl<T> IndexMut<(usize, usize)> for Array2dMutView<T> {
    fn index_mut(&mut self, pos: (usize, usize)) -> &mut T {
        unsafe { &mut *self.addr.add(pos.0 * self.orig_cols + pos.1) }
    }
}

impl<T: Default + Clone> Array2d<T> {
    fn new(rows: usize, cols: usize) -> Self {
        assert!(rows > 0, "row must be greater than 0");
        assert!(cols > 0, "column must be greater than 0");
        Array2d {
            rows,
            cols,
            data: vec![T::default(); rows * cols],
        }
    }

    fn from_vec(dim: (usize, usize), data: Vec<T>) -> Self {
        assert!(dim.0 > 0, "row must be greater than 0");
        assert!(dim.1 > 0, "column must be greater than 0");
        assert_eq!(dim.0 * dim.1, data.len());
        Array2d {
            rows: dim.0,
            cols: dim.1,
            data,
        }
    }

    fn shape(&self) -> (usize, usize) {
        (self.rows, self.cols)
    }

    fn fill(&mut self, elem: T) {
        for i in 0..self.rows * self.cols {
            self.data[i] = elem.clone()
        }
    }

    fn view_at(&self, off: (usize, usize), dim: (usize, usize)) -> Array2dView<T> {
        Array2dView {
            addr: unsafe { self.data.as_ptr().add(off.0 * self.cols + off.1) },
            orig_cols: self.cols,
            dim,
        }
    }

    fn mut_view_at(&mut self, off: (usize, usize), dim: (usize, usize)) -> Array2dMutView<T> {
        Array2dMutView {
            addr: unsafe { self.data.as_mut_ptr().add(off.0 * self.cols + off.1) },
            orig_cols: self.cols,
            dim,
        }
    }

    fn view(&self) -> Array2dView<T> {
        Array2dView {
            addr: self.data.as_ptr(),
            orig_cols: self.cols,
            dim: (self.rows, self.cols),
        }
    }

    fn mut_view(&mut self) -> Array2dMutView<T> {
        Array2dMutView {
            addr: self.data.as_mut_ptr(),
            orig_cols: self.cols,
            dim: (self.rows, self.cols),
        }
    }
}

impl<T> Index<(usize, usize)> for Array2d<T> {
    type Output = T;
    fn index(&self, pos: (usize, usize)) -> &T {
        &self.data[pos.0 * self.cols + pos.1]
    }
}

impl<T> IndexMut<(usize, usize)> for Array2d<T> {
    fn index_mut(&mut self, pos: (usize, usize)) -> &mut T {
        &mut self.data[pos.0 * self.cols + pos.1]
    }
}

impl<T: Debug> Debug for Array2d<T> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let mut ret = String::new();
        for r in 0..self.rows {
            for c in 0..self.cols {
                let mut elem_str = format!("{:?}", self.data[r * self.cols + c]);
                if c != self.cols - 1 {
                    elem_str.push_str(", ");
                }
                ret.push_str(&elem_str);
            }
            ret.push('\n');
        }
        write!(f, "{}", ret)
    }
}

impl Array2d<u32> {
    fn randint(&mut self, lim: u32) {
        use tinyrand::{Rand, StdRand};
        let mut rand = StdRand::default();

        for i in 0..self.rows {
            for j in 0..self.cols {
                self[(i, j)] = rand.next_lim_u32(lim);
            }
        }
    }

    fn transpose(&self) -> Array2d<u32> {
        let mut ret = Array2d::<u32>::new(self.cols, self.rows);
        for i in 0..self.rows {
            for j in 0..self.cols {
                ret.data[j * self.rows + i] = self.data[i * self.cols + j];
            }
        }
        ret
    }
}

fn matmul(a: Array2dView<u32>, b: Array2dView<u32>, mut c: Array2dMutView<u32>) {
    for i in 0..a.dim.0 {
        for k in 0..a.dim.1 {
            for j in 0..b.dim.1 {
                c[(i, j)] += a[(i, k)] * b[(k, j)];
            }
        }
    }
}

fn matadd(a: Array2dView<u32>, b: Array2dView<u32>, mut c: Array2dMutView<u32>) {
    for i in 0..a.dim.0 {
        for j in 0..a.dim.1 {
            c[(i, j)] = a[(i, j)] + b[(i, j)];
        }
    }
}

fn matmul_dac(a: Array2dView<u32>, b: Array2dView<u32>, c: Array2dMutView<u32>) {
    let (n, m, p) = (a.dim.0, a.dim.1, b.dim.1);
    let _square = n == m && m == p;
    let &max = [n, m, p].iter().max().unwrap();
    if max <= 181 {
        //c[(0,0)] += a[(0,0)] * b[(0,0)];
        matmul(a, b, c);
        return;
    }
    if max == m {
        let m2 = m / 2;
        let a1 = a.view_at((0, 0), (n, m2));
        let a2 = a.view_at((0, m2), (n, m - m2));
        let b1 = b.view_at((0, 0), (m2, p));
        let b2 = b.view_at((m2, 0), (m - m2, p));

        let mut c1 = Array2d::<u32>::new(n, p);
        let mut c2 = Array2d::<u32>::new(n, p);
        rayon::join(
            || matmul_dac(a1, b1, c1.mut_view()),
            || matmul_dac(a2, b2, c2.mut_view()),
        );
        matadd(c1.view(), c2.view(), c);
    } else if max == n {
        let n2 = n / 2;
        let a1 = a.view_at((0, 0), (n2, m));
        let a2 = a.view_at((n2, 0), (n - n2, m));
        let bc = b.clone();
        let c1 = c.mut_view_at((0, 0), (n2, p));
        let c2 = c.mut_view_at((n2, 0), (n - n2, p));
        rayon::join(|| matmul_dac(a1, bc, c1), || matmul_dac(a2, b, c2));
    } else {
        let p2 = p / 2;
        let b1 = b.view_at((0, 0), (m, p2));
        let b2 = b.view_at((0, p2), (m, p - p2));
        let ac = a.clone();
        let c1 = c.mut_view_at((0, 0), (n, p2));
        let c2 = c.mut_view_at((0, p2), (n, p - p2));
        rayon::join(|| matmul_dac(ac, b1, c1), || matmul_dac(a, b2, c2));
    }
}

fn main() {
    let dim = 4096usize;

    let mut a = Array2d::<u32>::new(dim, dim);
    let mut b = Array2d::<u32>::new(dim, dim);
    let mut c1 = Array2d::<u32>::new(dim, dim);
    let mut c2 = Array2d::<u32>::new(dim, dim);
    a.randint(10);
    b.randint(10);

    use std::time;

    let now = time::Instant::now();
    matmul_dac(a.view(), b.view(), c2.mut_view());
    println!("took {:?}", now.elapsed());

    let now = time::Instant::now();
    matmul(a.view(), b.view(), c1.mut_view());
    println!("took {:?}", now.elapsed());

    assert_eq!(c1.data , c2.data);
}

#[cfg(test)]
mod tests {
    use crate::Array2d;
    use crate::{matmul, matmul_dac};

    #[test]
    fn init_test() {
        let mut a = Array2d::<i32>::new(3, 2);
        a.fill(10);
        assert_eq!(a.data, vec![10; 6]);
        a[(2, 1)] = 100;
        assert_eq!(a[(2, 1)], 100);
    }

    #[test]
    fn mul_test() {
        let a = Array2d::<u32>::from_vec((2, 4), vec![11, 12, 13, 14, 21, 22, 23, 24]);
        let b = Array2d::<u32>::from_vec((4, 3), vec![12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1]);
        let mut c = Array2d::<u32>::new(2, 3);
        matmul(a.view(), b.view(), c.mut_view());
        assert_eq!(c.data, vec![360, 310, 260, 660, 570, 480]);
    }

    #[test]
    fn matmul_dac_test() {
        let a = Array2d::<u32>::from_vec((2, 4), vec![11, 12, 13, 14, 21, 22, 23, 24]);
        let b = Array2d::<u32>::from_vec((4, 3), vec![12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1]);
        let mut c = Array2d::<u32>::new(2, 3);
        matmul_dac(a.view(), b.view(), c.mut_view());
        assert_eq!(c.data, vec![360, 310, 260, 660, 570, 480]);
    }

    #[test]
    fn view_test() {
        let a = Array2d::<u32>::from_vec(
            (4, 5),
            vec![5, 9, 5, 5, 6, 1, 7, 6, 7, 1, 8, 1, 1, 8, 8, 0, 3, 4, 0, 8],
        );
        let av = a.view_at((0, 2), (4, 3));
        assert_eq!(format!("{:?}", av), "5, 5, 6\n6, 7, 1\n1, 8, 8\n4, 0, 8\n");
        let bv = av.view_at((2, 1), (2, 2));
        assert_eq!(format!("{:?}", bv), "8, 8\n0, 8\n");
        let cv = a.view_at((0, 0), (4, 2));
        assert_eq!(format!("{:?}", cv), "5, 9\n1, 7\n8, 1\n0, 3\n");
        let dv = a.view_at((0, 0), (3, 5));
        let ev = a.view_at((3, 0), (1, 5));
        assert_eq!(format!("{:?}", dv), "5, 9, 5, 5, 6\n1, 7, 6, 7, 1\n8, 1, 1, 8, 8\n");
        assert_eq!(format!("{:?}", ev), "0, 3, 4, 0, 8\n");
    }
}
```

Array2dView, Array2dMutView 内部使用了raw pointer，分别实现了Send trait, 测试4096x4096 size 性能改善:
```bash
took 2.413877699s
took 13.859504138s
```
python 社区有numpy, cupy 等数学运算库可以使用, 使用pip install numpy 没啥问题， 安装cupy 就报错好像是要求gcc 版本不能高于10, cupy 依赖系统已经安装CUDA Toolkit 开发套件库, cuda 环境建立:
```bash
#!/bin/bash
sudo ln -s /usr/bin/gcc-9   /usr/lib/cuda/bin/gcc
echo 'export PATH=/usr/lib/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/lib/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source  ~/.bashrc
sudo    ldconfig
```
然后cupy 安装顺利, 写了一个python 脚本测试matrix 乘法:
```python
#!/usr/bin/env python3

import argparse
import numpy as np
import cupy as cp  
from time import monotonic

cmd = argparse.ArgumentParser()
cmd.add_argument(dest='size',metavar='size', type=int,  nargs=3)
cmd.add_argument('--dev', dest='dev', action='store',
            choices={'cpu','gpu'}, default='gpu', help='which device to run')

args = cmd.parse_args()

def time_it(func, arg1, arg2):  
    start_time = monotonic()    
    ret = func(arg1, arg2) 
    #cp.cuda.Device(0).synchronize()
    finish_time = monotonic()
    elapsed_time = finish_time - start_time    
    return ret, elapsed_time

if args.dev == 'cpu':
    cpu_a = np.random.randint(0, 255, size=args.size[:2], dtype='uint32')
    cpu_b = np.random.randint(0, 255, size=args.size[1:] , dtype='uint32')
    r, t = time_it(np.matmul, cpu_a, cpu_b)
    print(f"for cpu, spent = {t}")
else:
    gpu_a = cp.random.randint(0, 255, size=args.size[:2], dtype='uint32')
    gpu_b = cp.random.randint(0, 255, size=args.size[1:], dtype='uint32')
    r, t = time_it(cp.matmul, gpu_a, gpu_b)
    print(f"for gpu, spent = {t}")
```
```bash
./numpy-cupy-bench.py --dev cpu  4096 4096  4096
for cpu, spent = 454.7690795919989

./numpy-cupy-bench.py --dev gpu  4096 4096  4096
for gpu, spent = 1.61205981799867
```
不清楚numpy为什么执行相当慢， numpy 后端默认使用openblas这样高度优化的库

cupy 使用gpu 貌似对输入大小不敏感: 
```bash
./numpy-cupy-bench.py --dev gpu  20000  20000  20000
for gpu, spent = 1.600698182999622
```
[google后](https://forums.developer.nvidia.com/t/a-question-about-cuda-launch-blocking/203174)，cuda 默认是异步模式，需要设置环境变量调整:
```bash
CUDA_LAUNCH_BLOCKING=1 ./numpy-cupy-bench.py --dev gpu  20000  20000  20000
for gpu, spent = 7.337842835000629```
```


上面的rust 实现的matmul_dac,  matmul 计算20000 * 20000 的输入用时:
```bash
took 333.385482359s
took 1628.970295792s
```

