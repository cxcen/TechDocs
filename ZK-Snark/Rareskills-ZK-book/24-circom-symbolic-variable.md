# Circom 中的符号变量

Circom 中的符号变量，是指被赋予了信号值的变量。

当信号被赋给变量，使其成为符号变量后，该变量就成为信号以及对该信号所施加算术运算的容器。符号变量与普通变量一样，使用 `var` 关键字声明。

例如，下面两个电路等价，也就是会生成相同的底层 R1CS：

```jsx
template ExampleA() {
	signal input a;
	signal input b;
	signal input c;

	a * b === c;
}

template ExampleB() {
	signal input a;
	signal input b;
	signal input c;

	// symbolic variable v "contains" a * b
	var v = a * b;

	// a * b === c under the hood
	v === c;
}
```

在 `ExampleB` 中，符号变量 `v` 只是表达式 `a * b` 的占位符。`ExampleA` 与 `ExampleB` 编译得到的 R1CS 完全相同，功能上没有任何区别。

## 符号变量的使用场景

### 检查 $\sum\texttt{in}[i]=\texttt{sum}$

如果要在循环中对信号数组求和，符号变量非常方便。事实上，这也是它们最常见的用途：

```jsx
// assert sum of in === sum
template Sum(n) {
	signal input in[n];
	signal input sum;

	var accumulator;
	for (var i = 0; i < n; i++) {
		accumulator += in[i];
	}

	// in[0] + in[1] + in[2] + ... + in[n - 1] === sum
	accumulator === sum;
}
```

### 检查 `in` 是否为 `k` 的有效二进制表示

另一个更有意思的例子，是证明 `in[n]` 是 `k` 的二进制表示，其中 `n` 是模板参数。在下面的电路中，我们检查：

$$
\texttt{in[0]}+2\cdot\texttt{in[1]}+4\cdot\texttt{in[2]} +\dots2^{n-1}\cdot\texttt{in[n-1]} == k
$$

如果还把 `in` 中的所有信号约束为只能取 $\set{0,1}$，就意味着 `in[]` 是 `k` 的二进制表示：

```jsx
template IsBinaryRepresentation(n) {

	signal input in[n];
	signal input k;

	// in is binary only
	for (var i = 0; i < n; i++) {
		in[i] * (in[i] - 1) === 0;
	}

	// in is the binary representation of k
	var acc; // symbolic variable
	var powersOf2 = 1; // regular variable
	for (var i = 0; i < n; i++) {
		acc += powersOf2 * in[i];
		powersOf2 *= 2;
	}

	acc === k;
}
```

### 符号变量为何有用

回到证明 $\sum\texttt{in}[i]=\texttt{sum}$ 的例子。如果没有符号变量，表达下式会很笨拙：

```jsx
sum === in[0] + in[1] + in[2] + ... + in[n-1];
```

因为我们并不预先知道 `n` 的值。即使 `n` 固定为 32，手工写出 32 个变量也很麻烦。因此，符号变量允许我们逐步构造 `in[0] + in[1] + in[2] + ...`，不必显式写出所有信号。

## 符号变量导致的非二次约束

符号变量可以“包含”两个信号之间的乘法；如果不小心，就可能把两次乘法嵌入同一条约束。考虑下面这个无法编译的示例：

```jsx
template QViolation() {
	signal input a;
	signal input b;
	signal input c;
	signal input d;

	// v "contains" a * b
	var v = a * b;

	// error: there are two
	// multiplications
	// in this constraint
	v === c * d;
}
```

在上述代码中，符号变量 `v` 包含一次乘法，并且我们声明了 `v == a*b`。因此，约束 `v === c * d;` 等价于 `a * b = c * d;`，所以上述代码无法编译。

## 非符号变量可以使用任意运算符

普通变量可以执行取模、位移等运算。不过，这意味着该变量之后不能成为约束的一部分：

```jsx
// this has no constraints
// but it will compile
template Ok() {
	signal input a;
	signal input b;

	var v = a % b;
}
```

上面的示例可以编译，因为 `v` 没有用于约束。但如果在约束中使用 `v`，代码将无法编译，如下所示：

```jsx
template NotOk() {
	signal input a;
	signal input b;
	signal input c;

	var v = a % b;

	// non-quadratic constraint
	c === v;
}
```

### 符号变量不能决定循环边界或条件

同样，只有普通变量可以决定循环边界或 if 语句的条件。使用符号变量会导致代码无法编译：

```jsx
template NotOk() {
	signal input a;
	signal input b;
	signal input c;

	var v = a * b;

	// v is a symbolic variable
	// used in an if statement
	if (v == 0) {
		c === 0;
	} else {
		c === 1;
	}
}
```

## 总结

符号变量是从信号获得赋值的变量。它们最常用于把参数化数量的信号相加，因为可以在 for 循环中累加求和。符号变量实际上是一个“容器”或“桶”，其中保存单个信号，或由多个信号相加、相乘组成的表达式。如果变量从未被赋予来自信号的值，它就不是符号变量。

由于符号变量包含信号，使用时必须注意避免违反二次约束规则。
