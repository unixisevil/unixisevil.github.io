+++
title = "Rabin-Karp, Boyer-Moore, Knuth-Morris-Pratt  字符串匹配"
date = 2019-03-06

[taxonomies]
tags = ["Rabin-Karp", "Boyer-Moore", "Knuth-Morris-Pratt", "golang"]
+++


Rabin-Karp 使用计算滚动hash 的方式来搜索文本中的模式， 计算每一次模式串位移一个位置后相应的文本块对应的hash值， hash值匹配后，有很高的概率是匹配的， 可以不去挨个字符的在检查一遍，获得高概率的正确性(Monte Carlo版)， 也可以逐字符地核查一遍获得绝对正确性(Las Vegas版)。

Robert Sedgewick 的图示解释：
{{ image(src="https://oscimg.oschina.net/oscnet/53e34b15646f071b95caff1b3be14905b01.jpg", position="left") }}


把字符串看作是基数是R(上图是基数10)数字，然后使用这个数字计算hash， 向前推动模式，获取对应文本中数字的方式，基于公式：
prev = txt[i]*R^(m-1) + txt[i+1]*R^(m-2) + ....  + txt[i+m-1]*R^0  (m是模式的长度)

滑动一个位移后的新数字：
 next = (prev - txt[i] * R ^ (m-1))*R  + txt[i+m]

在字符串4 1 5 9 2 6 5 中滑动一次后，假设原来模式长度5， 开始是4 1 5 9 2， 滑动一次变成1 5 9 2 6， 应用上面的公式来获得这个数字：
（41592 - 4 * 10  ^ (5-1)) *10  + 6 

在计算取模哈希的过程中为了数字过大溢出，使用了霍纳方法，图示:
{{ image(src="https://oscimg.oschina.net/oscnet/f4f8651175f0b82329b5985a09ace5c0ba5.jpg", position="left") }}

因为取模运算的性质:

( a + b ) mod m = ( a mod m + b mod m ) mod m

( a − b ) mod m = ( a mod m − b mod m ) mod m

( a * b ) mod m = ( a mod m  *  b mod m ) mod m

```golang
const R = 256
type RK struct {
	pat     string //pat
	patHash int    //pat hash
	m       int    //pat len
	q       int    //prime
	rm      int    // r ^(m-1) % q
}

func hash(key string, m, q int) int {
	h := 0
	for i := 0; i < m; i++ {
		h = (h*R + int(key[i])) % q
	}
	return h
}

func NewRK(pat string) *RK {
	m := len(pat)
	q := 16777619
	// precompute R^(m-1) % q for use in removing leading digit
	rm := 1
	for i := 1; i <= m-1; i++ {
		rm = (R * rm) % q
	}
	patHash := hash(pat, m, q)
	return &RK{
		pat:     pat,
		patHash: patHash,
		m:       m,
		q:       q,
		rm:      rm,
	}
}

func (rk *RK) Search(txt string) int {
	n := len(txt)
	m := rk.m
	rm := rk.rm
	q := rk.q

	if n < m {
		return -1
	}
	txtHash := hash(txt, m, q)
	if txtHash == rk.patHash && txt[0:rk.m] == rk.pat {
		return 0
	}
	for i := m; i < n; i++ {
		txtHash = (txtHash + q - rm*int(txt[i-m])%q) % q
		txtHash = (R*txtHash + int(txt[i])) % q
		if txtHash == rk.patHash && txt[i-m+1:i+1] == rk.pat {
			return i - m + 1
		}
	}
	return -1
}
```
golang 标准库strings.Index函数的非快速路径使用了Rabin-Karp 方法，不过基数R不是256，使用了一个大的素数R=16777619,没有使用取模哈希，滚动hash时使用了公式：

prev = txt[i]*R^(m-1) + txt[i+1]*R^(m-2) + ....  + txt[i+m-1]*R^0  (m是模式的长度)

滑动一个位移后的新数字：

 next = (prev * R + txt[i+m]) -  txt[i]*R^m

```golang
const primeRK = 16777619

type RKStdLib struct {
	pat     string
	patHash uint32
	rm      uint32 //primeRK ^  len(pat)
}

func NewRKStdLib(pat string) *RKStdLib {
	hash := uint32(0)
	for i := 0; i < len(pat); i++ {
		hash = hash*primeRK + uint32(pat[i])
	}
	var pow, sq uint32 = 1, primeRK
	for i := len(pat); i > 0; i >>= 1 {
		if i&1 != 0 {
			pow *= sq
		}
		sq *= sq
	}
	return &RKStdLib{
		pat:     pat,
		patHash: hash,
		rm:      pow,
	}
}

func (rk *RKStdLib) Search(txt string) int {
	m := len(rk.pat)
	n := len(txt)
	if n < m {
		return -1
	}
	var hash uint32
	for i := 0; i < m; i++ {
		hash = hash*primeRK + uint32(txt[i])
	}
	if hash == rk.patHash && txt[:m] == rk.pat {
		return 0
	}
	for i := m; i < n; {
		hash *= primeRK
		hash += uint32(txt[i])
		hash -= rk.rm * uint32(txt[i-m]) //remove leading digit
		i++
		if hash == rk.patHash && txt[i-m:i] == rk.pat {
			return i - m
		}
	}
	return -1
}
```
Algorithms 4th中的Boyer-Moore 讲的好像是[Boyer–Moore–Horspool](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore%E2%80%93Horspool_algorithm "Boyer–Moore–Horspool")原始Boyer-Moore 的简化版，只用bad char 数组启发，没有good suffix 数组启发。

{{ image(src="https://oscimg.oschina.net/oscnet/ca1a745066362cdab3315f29e6da557912a.jpg", position="left") }}

Boyer-Moore 从右往左扫描模式，NEEDLE 中的E和文本中的N失配，找到N在模式中最右边的出现位置，把模式向右拉动，使得两个N对齐，然后重新从模式的右端开始比对，E 与文本中的S失配，查找模式中S在最右端的出现，没找到，把模式向右拉动滑过S, 然后重新从模式的右端开始比对, 在倒数第二的位置L与N失配，查找N在模式中的最右端出现位置，让两个N对齐后，从右端重启比对过程，这次没有发生失配，模式找到。

从上面的描述过程，可以看出，需要预处理模式，在R =256的字母表中，提前了解哪些字符出现在模式中，它们的最右边位置，哪些字符没有出现。

```golang
type BM struct {
	pat   string
	right []int //bad char skip table
}

func NewBM(pat string) *BM {
	right := make([]int, R)
	for c := 0; c < R; c++ {
		right[c] = -1
	}
	for j := 0; j < len(pat); j++ {
		right[pat[j]] = j
	}
	return &BM{
		pat:   pat,
		right: right,
	}
}

func (b *BM) Search(txt string) int {
	m := len(b.pat)
	n := len(txt)
	max := func(a, b int) int {
		if a < b {
			return b
		}
		return a
	}
	skip := 0
	for i := 0; i <= n-m; i += skip {
		skip = 0
		for j := m - 1; j >= 0; j-- {
			if b.pat[j] != txt[i+j] {
				skip = max(1, j-b.right[i+j])
				break
			}
		}
		if skip == 0 {
			return i
		}
	}
	return -1
}
```
发生失配时，有三种情况：

文本中导致失配的字符在模式中没有出现，需要向右滑动模式，模式左端对齐失配字符的下一个字符，b.right[i+j] 的值是-1 表示失配字符没有出现，j - (-1), 变成j+1,   模式需要向右滑动j+1 个位置；

文本中导致失配的字符在模式中出现，最右出现位置在当前失配位置的左边，当前失配位置跟最右位置的距离 j-b.right[i+j] 就是模式需要往右滑动的次数；

文本中导致失配的字符在模式中出现，最右出现位置在当前失配位置的右边，如果把文本中的失配字符与模式中对应的最右字符对齐，模式需要往左滑动，所以不使用bad char 启发信息，直接把模式往右滑动一个位置；

算法导论 32.3 中介绍的使用自动机做文本匹配其实就是Sedgewick 说的KMP DFA 版本，算法导论 接着在32.4 介绍了KMP NFA版本，Sedgewick 省略了NFA版本。kmp 也需要对模式做预处理，匹配时和蛮力法没有区别，都是逐个字符前进，kmp做到失配发生时，文本指针i不后退，只让模式指针j 回退到合适的位置。kmp 中文本指针i 一直前进，模式指针j 匹配时前进一步，失配时根据预处理信息，回退到合适的位置。

理解kmp dfa 的关键是如何构造这个dfa， Sedgewick使用二维数组来dfa[char][j] 表示匹配自动机，j是自动机的当前状态，表示已经成功匹配了几个字符，dfa['a'][1] 表达已经成功匹配了一个字符的情况下收到字符'a' 时，自动机应该把状态迁移到某个新的状态。dfa[pat[0]][0] = 1,  在自动机初始状态0 下，收到模式的第一个字符后，状态转变为1，表示成功匹配了一个，在状态0的情况下，收到任何不等于模式首字符的字符时，状态始终该保持在0。所以对于下图的二维状态表格，第一例的内容很容易理解：

{{ image(src="https://oscimg.oschina.net/oscnet/5e0da47f161168884435f705d0425998869.jpg", position="left") }}

对于其他的匹配转换也很容易理解，1的状态收到B，转移到状态2，收到A转移到3，....  一直到收到模式最后一个字符C后，状态变成6，表示成功匹配完了模式的所有字符；不容易理解的是失配迁移， Sedgewick的观察当txt[i] 和pat[j] 失配时，文本指针 i 需要重新扫描已经匹配过的字符块pat[1..j-1], 于是使用dfa来模拟这个重新扫描pat[1..j-1], 然后尾随一个失配字符txt[i]的迁移，就是状态j 时，失配迁移后到达的状态。

假设mc 表示失配字符，图中的dfa[mc][5]的值就等于 dfa 开机后，模拟pat[1..4]= "BABA" 到达状态3，  dfa[mc][5] = dfa[mc][3],     在3的状态下，收到A到达1，
在3的状态下，收到B到达4， 所以dfa[A][5] = 1,   dfa[B][5]=4

dfa[mc][4]的值就等于 dfa 开机后，模拟pat[1..3]="BAB"  到达状态2， dfa[mc][4] = dfa[mc][2]

dfa[mc][3]的值就等于 dfa 开机后，模拟pat[1..2]="BA" 到达状态1，   dfa[mc][3] = dfa[mc][1]

dfa[mc][2]的值就等于 dfa 开机后，模拟pat[1..1]="B" 到达状态0，   dfa[mc][2] = dfa[mc][0]

dfa[mc][1]的值就等于 dfa 开机后，模拟pat[1..0]="" 到达状态0，   dfa[mc][1] = dfa[mc][0]

dfa[mc][0] 总是等于0， 这样就能计算出所有状态下发生失配应该跳转的状态。代码实现上维护了dfa的当前状态x, 不用每次从开机后模拟:

```golang
const R = 256
type KmpDfa struct {
	dfa [R][]int
	pat string
}

func NewKmpDfa(pat string) *KmpDfa {
	m := len(pat)
	kmp := &KmpDfa{}
	kmp.pat = pat
	for r := 0; r < R; r++ {
		kmp.dfa[r] = make([]int, m)
	}
	kmp.dfa[pat[0]][0] = 1
	for x, j := 0, 1; j < m; j++ {
		for r := 0; r < R; r++ {
			kmp.dfa[r][j] = kmp.dfa[r][x]
		}
		kmp.dfa[pat[j]][j] = j + 1
		x = kmp.dfa[pat[j]][x]
	}
	return kmp
}

func (k *KmpDfa) Search(txt string) int {
	m := len(k.pat)
	n := len(txt)
	i, j := 0, 0
	for ; i < n && j < m; i++ {
		j = k.dfa[txt[i]][j]
	}
	if j == m {
		return i - m
	}
	return -1
}
```
kmp nfa 方法，消除了对字母表的依赖，通过预计算前缀函数pi 的值做匹配过程，算法导论上描述pi[q]是模式pat 的前缀 pat[q]的真后缀的最大长度，其实就是border string 的最大宽度。

有老外定义了border string 的概念，感觉更好理解。比如"a"的string border是 "",  "abcd"的string border是 "",   "aba" 是 "a",   "abab" 是"ab",  "abcabca" 是"abca" 。

pi[i]的值就是模式pat的前缀 pat[i]的border string的宽度，从左到右扫描模式字符串，产生模式的前缀子串， pat[0..i] ,  i 在[0..len(pat)-1] 范围递增，对于样例字符串"abababcaab"来说，计算它的prefix function pi 过程如下：
 
 i 的值|border string|border string 右边的字符 | 把border string 右边的字符与pat[i]比较|前缀函数pi 的取值|
 --------|----------------|------|----|-------------|
 |  0     |         ""       |   'a'   | no         |    0     |
 |  1      |        ""       |   'a'  | a 比较 b，不等           |        0   |
 |  2     |        ""           | 'a'  |  a 比较a ，相等，当前border string宽度加一   |    1    |
 |  3    |         "a"         | 'b'  |  b 比较b, 相等，当前border string宽度加一       |        2       |
 |  4    |        "ab"        | 'a'  |  a 比较a，相等，当前border string宽度加一     |   3            |
 |  5    |       "aba"       | 'b'  | b 比较b,相等,当前border string宽度加一     |    4            |
 |  6    |      "abab"      |  'a' |  a 比较c,不等,取当前border string最宽的border string,"ab" 作为当前border string,取右边字符a比较c,不等,继续取当前border string 的最宽border string,""作为当前border string,取右边字符a比较c,不等，当前border stirng为空,终止                                                                     |                0   |
 |  7    |     ""                  |   'a'   | a比较a,相等, 当前border string宽度加一                                                                     |        1            |
 |  8    | "a"                      |  'b'    | b比较a,不等,取当前border string最宽的border string,"", 作为当前border string,取右边字符a比较a,相等,当前border string宽度加一                                                                       |          1          |
 |  9    |     "a"                  |  'b'    | b比较b,相等,当前border string宽度加一                                                                          |    2                |
 ```golang
 type KmpNfa struct {
	pat string
	pi  []int
}

func NewKmpNfa(pat string) *KmpNfa {
	m := len(pat)
	pi := make([]int, m)
	pi[0] = 0
	border := 0
	for i := 1; i < m; i++ {
		for border > 0 && pat[i] != pat[border] {
			border = pi[border-1]
		}
		if pat[i] == pat[border] {
			border += 1
		} else {
			border = 0
		}
		pi[i] = border
	}
	fmt.Printf("pi table =%+v\n", pi)
	return &KmpNfa{
		pat: pat,
		pi:  pi,
	}
}

func (k *KmpNfa) Search(txt string) int {
	m := len(k.pat)
	n := len(txt)
	i, j := 0, 0
	for ; i < n && j < m; i++ {
		for j > 0 && txt[i] != k.pat[j] {
			j = k.pi[j]
		}
		if txt[i] == k.pat[j] {
			j++
		}
	}
	if j == m {
		return i - m
	}
	return -1
}
 ```
如果txt[i] 匹配pat[j],i ,j 一起前进，跟蛮力搜索一样，发生失配时,模式指针j后退到pat[j]的border string 的右边，重新尝试匹配，模式前缀pat[j] 的border string 右边位置正好是pat[j]。
