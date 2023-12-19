+++
title = "carbon 如何表达解析树"
date = 2023-12-18

[taxonomies]
tags = ["rust", "tree", "carbon"]
+++


之前看到[Chandler Carruth介绍关于现代化carbon编译器工具链](https://www.youtube.com/watch?v=ZI198eFghJk&t=3684s&ab_channel=CppNow)的视频, carbon 的parse tree 的实现不再基于指针来构造tree, 使用cache 友好的动态数组表达.

[想起Adrian Sampson的blog也再讲Flattening AST](https://www.cs.cornell.edu/~asampson/blog/flattening.html)也是使用vector存储表达式节点，然后使用整数索引替换指针.

无独有偶rust的编译器内部也是大量使用整数id的数据结构, 像是[graph的实现](https://doc.rust-lang.org/stable/nightly-rustc/rustc_data_structures/graph/implementation/struct.Graph.html), [AST node的表达](https://doc.rust-lang.org/stable/nightly-rustc/rustc_ast/node_id/struct.NodeId.html)等等.

carbon 的parse tree 实现相对比较容易理解, 以下我用rust代码大致解释其实现:

```rust
use index_vec::{index_vec, IndexVec};

fn main() {
    let mut tree = Tree::<char>::new();

    let b = tree.new_node('b');
    let c = tree.new_node('c');
    let h = tree.new_node('h');
    let d = tree.new_node('d');
    tree.add_child(d, h);

    let i = tree.new_node('i');
    let p = tree.new_node('p');
    let q = tree.new_node('q');
    let j = tree.new_node('j');
    tree.add_child(j, p);
    tree.add_child(j, q);
    let e = tree.new_node('e');
    tree.add_child(e, i);
    tree.add_child(e, j);

    let k = tree.new_node('k');
    let l = tree.new_node('l');
    let m = tree.new_node('m');
    let f = tree.new_node('f');
    tree.add_child(f, k);
    tree.add_child(f, l);
    tree.add_child(f, m);

    let n = tree.new_node('n');
    let g = tree.new_node('g');
    tree.add_child(g, n);

    let a = tree.new_node('a');
    tree.add_child(a, b);
    tree.add_child(a, c);
    tree.add_child(a, d);
    tree.add_child(a, e);
    tree.add_child(a, f);
    tree.add_child(a, g);

    tree.postorder_subtree(e);
    println!();
    for c in tree.postorder_children(e) {
        println!("{c:?}, {:?}", tree.nodes[c]);
    }
    println!();
    for c in tree.children(e) {
        println!("{c:?}, {:?}", tree.nodes[c]);
    }
    println!();
    let mut buf = String::new();
    tree.postorder_yaml(&mut buf);
    println!("{buf}");

    buf.clear();
    tree.preorder_yaml(&mut buf);
    println!("{buf}");
}


index_vec::define_index_type! {
    pub struct NodeRef = u32;
    MAX_INDEX = (u32::max_value() - 1) as usize;
    DEFAULT = NodeRef::from_raw_unchecked(u32::max_value());
    IMPL_RAW_CONVERSIONS = true;
    DISABLE_MAX_INDEX_CHECK = cfg!(not(debug_assertions));
}

#[derive(Debug)]
struct Node<T: std::fmt::Debug> {
    value: T,
    subtree_size: isize,
}

struct ChildrenIter<'a, T: std::fmt::Debug> {
    tree: &'a Tree<T>,
    curr: isize,
    end: isize,
}

impl<'a, T: std::fmt::Debug> Iterator for ChildrenIter<'a, T> {
    type Item = NodeRef;
    fn next(&mut self) -> Option<Self::Item> {
        if self.curr <= self.end {
            return None;
        }
        let curr_tree_size = self.tree.nodes[self.curr as usize].subtree_size;
        let ret = self.curr;
        self.curr -= curr_tree_size;
        Some(NodeRef::from(ret as usize))
    }
}

impl<T: std::fmt::Debug> Node<T> {
    fn new(val: T) -> Self {
        Self {
            value: val,
            subtree_size: 1,
        }
    }
}

struct Tree<T: std::fmt::Debug> {
    nodes: IndexVec<NodeRef, Node<T>>,
}

impl<T: std::fmt::Debug> Tree<T> {
    fn new() -> Self {
        Self {
            nodes: index_vec![],
        }
    }

    fn with_capacity(cap: usize) -> Self {
        Self {
            nodes: IndexVec::with_capacity(cap),
        }
    }

    fn new_node(&mut self, val: T) -> NodeRef {
        self.nodes.push(Node::new(val))
    }

    fn add_child(&mut self, p: NodeRef, c: NodeRef) {
        self.nodes[p].subtree_size += self.nodes[c].subtree_size;
    }

    fn children(&self, n: NodeRef) -> ChildrenIter<'_, T> {
        ChildrenIter {
            tree: self,
            curr: usize::from(n) as isize - 1,
            end: usize::from(n) as isize - self.nodes[n].subtree_size,
        }
    }

    fn postorder_children(&self, n: NodeRef) -> Vec<NodeRef> {
        let mut pos: Vec<NodeRef> = self.children(n).collect();
        pos.reverse();
        pos
    }

    fn size(&self, n: NodeRef) -> isize {
        self.nodes[n].subtree_size
    }

    fn postorder(&self) {
        for e in self.nodes.iter() {
            println!("{:?}", e);
        }
    }

    fn postorder_subtree(&self, n: NodeRef) {
        let end_pos = n + 1;
        let start_pos = end_pos - self.size(n) as usize;
        for e in &self.nodes[start_pos..end_pos] {
            println!("{e:?}");
        }
    }

    fn yaml_node(&self, buf: &mut String, n: NodeRef, depth: u32, preorder: bool) -> bool {
        let indent = " ".repeat(2 * (depth as usize + 1));
        buf.push_str(&indent);
        buf.push('{');
        if preorder {
            buf.push_str(&format!("node_index: {n:?}, "));
        }
        buf.push_str(&format!("value: {:?}", self.nodes[n].value));
        if self.size(n) > 1 {
            buf.push_str(&format!(", subtree_size: {}", self.size(n)));
            if preorder {
                buf.push_str(", children: [\n");
                return true;
            }
        }
        buf.push('}');
        false
    }

    fn postorder_yaml(&self, buf: &mut String) {
        let mut indents = vec![0u32; self.nodes.len()];
        let mut stack = Vec::<(NodeRef, u32)>::with_capacity(16);

        stack.push((NodeRef::from(self.nodes.len() - 1), 0));
        while let Some((n, depth)) = stack.pop() {
            for c in self.children(n) {
                indents[usize::from(c)] = depth + 1;
                stack.push((c, depth + 1));
            }
        }
        buf.push_str("[\n");
        for n in 0..self.nodes.len() {
            self.yaml_node(buf, NodeRef::from(n), indents[n], false);
            buf.push_str(",\n");
        }
        buf.push_str("]\n");
    }

    fn preorder_yaml(&self, buf: &mut String) {
        let mut stack = Vec::<(NodeRef, u32)>::with_capacity(16);
        stack.push((NodeRef::from(self.nodes.len() - 1), 0));
        buf.push_str("[\n");
        while let Some((n, depth)) = stack.pop() {
            if self.yaml_node(buf, n, depth, true) {
                for c in self.children(n) {
                    stack.push((c, depth + 1));
                }
                continue;
            }
            let next_depth = if stack.is_empty() {
                0
            } else {
                stack.last().unwrap().1
            };
            assert!(next_depth <= depth);
            for _ in 0..(depth - next_depth) {
                buf.push_str("]}");
            }
            buf.push_str("  ,\n");
        }
        buf.push_str("]\n");
    }
}
```

main 函数中构造了一颗这样的树:

<img src="/imgs/tree.svg">


tree节点在vector中的存储顺序是post order, 因为parse过程会采用递归下降，自底向上地构造parse tree 或者AST.

<img src="/imgs/vec.svg">

顺便值得一提，[kroki](https://kroki.io/)很好用的在线画图工具，支持GraphViz code.

postorder_yaml方法，打印post order的yaml:
```yaml
[
    {value: 'b'},
    {value: 'c'},
      {value: 'h'},
    {value: 'd', subtree_size: 2},
      {value: 'i'},
        {value: 'p'},
        {value: 'q'},
      {value: 'j', subtree_size: 3},
    {value: 'e', subtree_size: 5},
      {value: 'k'},
      {value: 'l'},
      {value: 'm'},
    {value: 'f', subtree_size: 4},
      {value: 'n'},
    {value: 'g', subtree_size: 2},
  {value: 'a', subtree_size: 16},
]
```

preorder_yaml方法，打印pre order的yaml:
```yaml
[
  {node_index: 15, value: 'a', subtree_size: 16, children: [
    {node_index: 0, value: 'b'}  ,
    {node_index: 1, value: 'c'}  ,
    {node_index: 3, value: 'd', subtree_size: 2, children: [
      {node_index: 2, value: 'h'}]}  ,
    {node_index: 8, value: 'e', subtree_size: 5, children: [
      {node_index: 4, value: 'i'}  ,
      {node_index: 7, value: 'j', subtree_size: 3, children: [
        {node_index: 5, value: 'p'}  ,
        {node_index: 6, value: 'q'}]}]}  ,
    {node_index: 12, value: 'f', subtree_size: 4, children: [
      {node_index: 9, value: 'k'}  ,
      {node_index: 10, value: 'l'}  ,
      {node_index: 11, value: 'm'}]}  ,
    {node_index: 14, value: 'g', subtree_size: 2, children: [
      {node_index: 13, value: 'n'}]}]}  ,
]
```
