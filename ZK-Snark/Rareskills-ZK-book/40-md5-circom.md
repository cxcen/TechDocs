# Circom 中的 MD5 哈希

本教程将在 Circom 中实现 MD5 哈希：既计算哈希，也通过 Circom 约束证明计算正确。

虽然 MD5 哈希函数在密码学上已不再安全（因为已经发现碰撞），但它的工作机制与密码学安全的哈希函数相似。

更重要的是，MD5 哈希函数很容易快速掌握。下面这段 14 分钟的视频解释了 MD5 哈希的工作原理，建议先观看：

[https://www.youtube.com/watch?v=5MiMK45gkTY](https://www.youtube.com/watch?v=5MiMK45gkTY)

要在不公开原像的情况下证明我们知道某个 MD5 哈希的原像，就需要证明哈希的每一步都执行正确，并产生了特定结果。本教程将展示如何为每次状态转移设计约束。

具体来说，MD5 哈希包含以下子程序：

- 按位 AND、OR、NOT 和 XOR
- LeftRotate（循环左移）
- 将 32 位数相加，并在 $2^{32}$ 处溢出
- 函数 `Func`，它使用按位运算符组合寄存器 `B`、`C` 和 `D`
- 开始时的填充步骤：在输入后添加一个 1 比特，并写入输入的长度（以比特为单位）

此外，MD5 输出通常以大端序写成一个 128 位数。假设有一个 128 位值 `0x1234567890ABCDEF1122334455667788`

它的大端序写法为：

```solidity
0x12   0x34   0x56   0x78   0x90   0xAB   0xCD   0xEF   0x11   0x22   0x33   0x44   0x55   0x66   0x77   0x88
```

小端序写法则为：

```solidity
0x88   0x77   0x66   0x55   0x44   0x33   0x22   0x11   0xEF   0xCD   0xAB   0x90   0x78   0x56   0x34   0x12
```

我们需要一个单独的例程，将字节顺序从小端序反转为大端序。大多数哈希实现都输出大端序，因此为了方便将结果与成熟库进行比较，我们希望自己的实现也采用大端格式。稍后会为此创建一个 `ToBytes` 组件。

虽然代码中有大量数组索引操作，但使用哪个索引完全由哈希的迭代轮次决定，因此哈希约束中不需要 Quin Selector——可以把数组索引写死。

## 用 Python 构建 MD5 原型

构建哈希函数这样复杂的程序时，最好先用 Python 等更熟悉、更易调试的语言编写参考实现，再把 Python 代码翻译成 Circom。

下面是 MD5 的 Python 实现（为简单起见，只支持 448 位输入）。它在很大程度上参考了 [Utkarsh87 的另一份实现](https://github.com/Utkarsh87/md5-hashing)。我们尽量让这些函数表现得像“组件”，从而更直接地翻译成 Circom。

实现说明：

- 模 $2^{32}$ 加法通过先把数字相加，再调用函数 `Overflow32()` 完成。
- 输入采用字节数组，而不是比特数组。
- 字节 `0x80` 的二进制表示是 `10000000`，这让我们可以在输入末尾填充一个比特。
- 输出采用大端格式。

```python
s = [7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22,
     5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20,
     4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23,
     6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21]

K = [0xd76aa478, 0xe8c7b756, 0x242070db, 0xc1bdceee,
     0xf57c0faf, 0x4787c62a, 0xa8304613, 0xfd469501,
     0x698098d8, 0x8b44f7af, 0xffff5bb1, 0x895cd7be,
     0x6b901122, 0xfd987193, 0xa679438e, 0x49b40821,
     0xf61e2562, 0xc040b340, 0x265e5a51, 0xe9b6c7aa,
     0xd62f105d, 0x02441453, 0xd8a1e681, 0xe7d3fbc8,
     0x21e1cde6, 0xc33707d6, 0xf4d50d87, 0x455a14ed,
     0xa9e3e905, 0xfcefa3f8, 0x676f02d9, 0x8d2a4c8a,
     0xfffa3942, 0x8771f681, 0x6d9d6122, 0xfde5380c,
     0xa4beea44, 0x4bdecfa9, 0xf6bb4b60, 0xbebfbc70,
     0x289b7ec6, 0xeaa127fa, 0xd4ef3085, 0x04881d05,
     0xd9d4d039, 0xe6db99e5, 0x1fa27cf8, 0xc4ac5665,
     0xf4292244, 0x432aff97, 0xab9423a7, 0xfc93a039,
     0x655b59c3, 0x8f0ccc92, 0xffeff47d, 0x85845dd1,
     0x6fa87e4f, 0xfe2ce6e0, 0xa3014314, 0x4e0811a1,
     0xf7537e82, 0xbd3af235, 0x2ad7d2bb, 0xeb86d391]

iter_to_index = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 1, 6, 11, 0, 5, 10, 15, 4, 9, 14, 3, 8, 13, 2, 7, 12, 5, 8, 11, 14, 1, 4, 7, 10, 13, 0, 3, 6, 9, 12, 15, 2, 0, 7, 14, 5, 12, 3, 10, 1, 8, 15, 6, 13, 4, 11, 2, 9]

def Overflow32(x):
    return x & 0xFFFFFFFF

def leftRotate(x, amount):
    #x &= 0xFFFFFFFF
    xo = Overflow32(x)
    return Overflow32((xo << amount | xo >> (32 -amount)))

def func(B, C, D, i):
    out = None

    # note that i will be 1..64 inclusive
    if i <= 16:
        out = (B & C) | (~B & D)

    elif i > 16 and i <= 32:
        out = (D & B) | (~D & C)

    elif i > 32 and i <= 48:
        out = B ^ C ^ D

    elif i > 48 and i <= 64:
        out = C ^ (B | ~D)

    else:
        assert False, "1) What"

    return out

# concatenates four bytes to become 32 bits
def To32BitWord(byte1, byte2, byte3, byte4):
    return byte1 + byte2 * 2**8 + byte3 * 2**16 + byte4 * 2**24

# length is the byte where the data stops
# so if we have zero bytes, we write 0x80
# to byte 0
def md5(bytes, length):
    data = bytearray(64)
    msg = bytearray(bytes, 'ascii')

    # 56 bytes, 64 is the max
    assert length < 56, "too long"

    if length < 56:
        data[length] = 0x80
        data[56] = (length * 8).to_bytes(1, byteorder='little')[0]
        for i in range(57,64):
            data[i] = 0x00

    for i in range(0, length):
        data[i] = msg[i]

    # data is a len 64 array of bytes. However, it will be much easier to work
    # on if we turn it into a len 16 array of 32 bit words
    data_32 = [0] * 16
    for i in range(0, 16):
        data_32[i] = To32BitWord(data[4*i], data[4*i + 1], data[4*i + 2], data[4*i + 3])

    # algo runs for 64 iterations with 4 registers, each using 32 bits
    # we allocate 65, because the 0th will be the default starting value
    buffer = [[0]*4 for _ in range(65)]

    buffer[0][0] = 0x67452301
    buffer[0][1] = 0xefcdab89
    buffer[0][2] = 0x98badcfe
    buffer[0][3] = 0x10325476

    A = 0
    B = 1
    C = 2
    D = 3
    for i in range(1, 65):

        F = func(buffer[i - 1][B], buffer[i - 1][C], buffer[i - 1][D], i)
        G = iter_to_index[i - 1]
        to_rotate = buffer[i-1][A] + F + K[i - 1] + data_32[iter_to_index[i - 1]]
        rotated = leftRotate(to_rotate, s[i - 1])
        new_B = Overflow32(buffer[i-1][B] + rotated)

        buffer[i][A] = buffer[i - 1][D]
        buffer[i][B] = new_B
        buffer[i][C] = buffer[i - 1][B]
        buffer[i][D] = buffer[i - 1][C]

    final = [0,0,0,0]
    for i, b in enumerate(buffer[64]):
        final[i] = Overflow32((b + buffer[0][i]))

    digest = final[0] + final[1] * 2**32 + final[2] * 2**64 + final[3] * 2**96

    raw = digest.to_bytes(16, byteorder='little')
    return int.from_bytes(raw, byteorder='big')


print(hex(md5("RareSkills", 10)))

```

## 前置组件

### Overflow32

Overflow32 模拟 VM 中发生在 $2^{32}$ 处的 32 位溢出：

```jsx
template Overflow32() {
    signal input in;
    signal output out;

    component n2b = Num2Bits(252);
    component b2n = Bits2Num(32);

    n2b.in <== in;
    for (var i = 0; i < 32; i++) {
        n2b.out[i] ==> b2n.in[i];
    }

    b2n.out ==> out;
}
```

## LeftRotate（循环左移）

LeftRotate 将这些比特视为放在环形缓冲区中，并对其进行旋转：

```jsx
template LeftRotate(s) {
    signal input in;
    signal output out;

    component n2b = Num2Bits(32);
    component b2n = Bits2Num(32);

    n2b.in <== in;

    for (var i = 0; i < 32; i++) {
        b2n.in[(i + s) % 32] <== n2b.out[i];
    }

    out <== b2n.out;
}
```

## 按位 AND、OR、XOR 和 NOT

下面这些模板来自前面的 32 位模拟教程：

```jsx
template BitwiseAnd32() {
    signal input in[2];
    signal output out;

    // range check
    component n2ba = Num2Bits(32);
    component n2bb = Num2Bits(32);
    n2ba.in <== in[0];
    n2bb.in <== in[1];

    component b2n = Bits2Num(32);
    component Ands[32];
    for (var i = 0; i < 32; i++) {
        Ands[i] = AND();
        Ands[i].a <== n2ba.out[i];
        Ands[i].b <== n2bb.out[i];
        Ands[i].out ==> b2n.in[i];
    }

    b2n.out ==> out;
}

template BitwiseOr32() {
    signal input in[2];
    signal output out;

    // range check
    component n2ba = Num2Bits(32);
    component n2bb = Num2Bits(32);
    n2ba.in <== in[0];
    n2bb.in <== in[1];

    component b2n = Bits2Num(32);
    component Ors[32];
    for (var i = 0; i < 32; i++) {
        Ors[i] = OR();
        Ors[i].a <== n2ba.out[i];
        Ors[i].b <== n2bb.out[i];
        Ors[i].out ==> b2n.in[i];
    }

    b2n.out ==> out;
}

template BitwiseXor32() {
    signal input in[2];
    signal output out;

    // range check
    component n2ba = Num2Bits(32);
    component n2bb = Num2Bits(32);
    n2ba.in <== in[0];
    n2bb.in <== in[1];

    component b2n = Bits2Num(32);
    component Xors[32];
    for (var i = 0; i < 32; i++) {
        Xors[i] = XOR();
        Xors[i].a <== n2ba.out[i];
        Xors[i].b <== n2bb.out[i];
        Xors[i].out ==> b2n.in[i];
    }

    b2n.out ==> out;
}

template BitwiseNot32() {
    signal input in;
    signal output out;

    // range check
    component n2ba = Num2Bits(32);
    n2ba.in <== in;

    component b2n = Bits2Num(32);
    component Nots[32];
    for (var i = 0; i < 32; i++) {
        Nots[i] = NOT();
        Nots[i].in <== n2ba.out[i];
        Nots[i].out ==> b2n.in[i];
    }

    b2n.out ==> out;
}
```

### Func

Func 接收寄存器 `B`、`C` 和 `D`，并把它们组合成单个输出——具体组合方式取决于当前迭代轮次：

```jsx
template Func(i) {
    assert(i <= 64);
    signal input b;
    signal input c;
    signal input d;

    signal output out;

    if (i <= 16) {
        signal bAndc;
        component a1 = BitwiseAnd32();
        a1.in[0] <== b;
        a1.in[1] <== c;

        component a2 = BitwiseAnd32();
        component n1 = BitwiseNot32();
        n1.in <== b;
        a2.in[0] <== n1.out;
        a2.in[1] <== d;

        component o1 = BitwiseOr32();
        o1.in[0] <== a1.out;
        o1.in[1] <== a2.out;

        out <== o1.out;
    }
    else if (i > 16 && i <= 32) {
        component a1 = BitwiseAnd32();
        a1.in[0] <== d;
        a1.in[1] <== b;

        component n1 = BitwiseNot32();
        n1.in <== d;
        component a2 = BitwiseAnd32();
        a2.in[0] <== n1.out;
        a2.in[1] <== c;

        component o1 = BitwiseOr32();
        o1.in[0] <== a1.out;
        o1.in[1] <== a2.out;

        out <== o1.out;
    }
    else if (i > 32 && i <= 48) {
        component x1 = BitwiseXor32();
        component x2 = BitwiseXor32();

        x1.in[0] <== b;
        x1.in[1] <== c;
        x2.in[0] <== x1.out;
        x2.in[1] <== d;

        out <== x2.out;
    }
    // i must be <= 64 by the assert
    // statement above
    else {
        component o1 = BitwiseOr32();
        component n1 = BitwiseNot32();
        n1.in <== d;
        o1.in[0] <== n1.out;
        o1.in[1] <== b;

        component x1 = BitwiseXor32();
        x1.in[0] <== o1.out;
        x1.in[1] <==c;

        out <== x1.out;
    }

```

### 输入填充

为简单起见，这个哈希函数接收字节数组，而不是比特数组。此外，我们把输入长度限制为 56 字节，这样就能把“在哈希所用的 64 字节（512 位）输入的第 56 个字节处插入长度”写死。

由于输入最多为 56 字节，用来表示长度的数不会超过 448 位，因此最多需要 2 个字节保存。

```jsx
// n is the number of bytes
template Padding(n) {
    // 56 bytes = 448 bits
    assert(n < 56);

    signal input in[n];

    // 64 bytes = 512 bits
    signal output out[64];

    for (var i = 0; i < n; i++) {
        out[i] <== in[i];
    }

    // add 128 = 0x80 to pad the 1 bit (0x80 = 10000000b)
    out[n] <== 128;

    // pad the rest with zeros
    for (var i = n + 1; i < 56; i++) {
        out[i] <== 0;
    }

    var lenBits = n * 8;
    if (lenBits < 256) {
        out[56] <== lenBits;
    }
    else {
        var lowOrderBytes = lenBits % 256;
        var highOrderBytes = lenBits \ 256;
        out[56] <== lowOrderBytes;
        out[57] <== highOrderBytes;
    }
}
```

## Num2Bytes

要改变端序，需要把信号 `in` 转换为字节数组 `out`：

```jsx
// n is the number of bytes
template ToBytes(n) {
    signal input in;
    signal output out[n];

    component n2b = Num2Bits(n * 8);
    n2b.in <== in;

    component b2ns[n];
    for (var i = 0; i < n; i++) {
        b2ns[i] = Bits2Num(8);
        for (var j = 0; j < 8; j++) {
            b2ns[i].in[j] <== n2b.out[8*i + j];
        }
        out[i] <== b2ns[i].out;
    }
}
```

## 最终方案

下面的代码组合了所有组件，用于执行 MD5 哈希，同时把结果转换为大端序。读者可以在 [zkrepl](https://zkrepl.dev) 中测试这段代码。

```jsx
include "circomlib/bitify.circom";
include "circomlib/gates.circom";

template BitwiseAnd32() {
    signal input in[2];
    signal output out;

    // range check
    component n2ba = Num2Bits(32);
    component n2bb = Num2Bits(32);
    n2ba.in <== in[0];
    n2bb.in <== in[1];

    component b2n = Bits2Num(32);
    component Ands[32];
    for (var i = 0; i < 32; i++) {
        Ands[i] = AND();
        Ands[i].a <== n2ba.out[i];
        Ands[i].b <== n2bb.out[i];
        Ands[i].out ==> b2n.in[i];
    }

    b2n.out ==> out;
}

template BitwiseOr32() {
    signal input in[2];
    signal output out;

    // range check
    component n2ba = Num2Bits(32);
    component n2bb = Num2Bits(32);
    n2ba.in <== in[0];
    n2bb.in <== in[1];

    component b2n = Bits2Num(32);
    component Ors[32];
    for (var i = 0; i < 32; i++) {
        Ors[i] = OR();
        Ors[i].a <== n2ba.out[i];
        Ors[i].b <== n2bb.out[i];
        Ors[i].out ==> b2n.in[i];
    }

    b2n.out ==> out;
}

template BitwiseXor32() {
    signal input in[2];
    signal output out;

    // range check
    component n2ba = Num2Bits(32);
    component n2bb = Num2Bits(32);
    n2ba.in <== in[0];
    n2bb.in <== in[1];

    component b2n = Bits2Num(32);
    component Xors[32];
    for (var i = 0; i < 32; i++) {
        Xors[i] = XOR();
        Xors[i].a <== n2ba.out[i];
        Xors[i].b <== n2bb.out[i];
        Xors[i].out ==> b2n.in[i];
    }

    b2n.out ==> out;
}

template BitwiseNot32() {
    signal input in;
    signal output out;

    // range check
    component n2ba = Num2Bits(32);
    n2ba.in <== in;

    component b2n = Bits2Num(32);
    component Nots[32];
    for (var i = 0; i < 32; i++) {
        Nots[i] = NOT();
        Nots[i].in <== n2ba.out[i];
        Nots[i].out ==> b2n.in[i];
    }

    b2n.out ==> out;
}

// n is the number of bytes
template ToBytes(n) {
    signal input in;
    signal output out[n];

    component n2b = Num2Bits(n * 8);
    n2b.in <== in;

    component b2ns[n];
    for (var i = 0; i < n; i++) {
        b2ns[i] = Bits2Num(8);
        for (var j = 0; j < 8; j++) {
            b2ns[i].in[j] <== n2b.out[8*i + j];
        }
        out[i] <== b2ns[i].out;
    }
}

// n is the number of bytes
template Padding(n) {
    // 56 bytes = 448 bits
    assert(n < 56);

    signal input in[n];

    // 64 bytes = 512 bits
    signal output out[64];

    for (var i = 0; i < n; i++) {
        out[i] <== in[i];
    }

    // add 128 = 0x80 to pad the 1 bit (0x80 = 10000000b)
    out[n] <== 128;

    // pad the rest with zeros
    for (var i = n + 1; i < 56; i++) {
        out[i] <== 0;
    }

    var lenBits = n * 8;
    if (lenBits < 256) {
        out[56] <== lenBits;
    }
    else {
        var lowOrderBytes = lenBits % 256;
        var highOrderBytes = lenBits \ 256;
        out[56] <== lowOrderBytes;
        out[57] <== highOrderBytes;
    }
}
template Overflow32() {
    signal input in;
    signal output out;

    component n2b = Num2Bits(252);
    component b2n = Bits2Num(32);

    n2b.in <== in;
    for (var i = 0; i < 32; i++) {
        n2b.out[i] ==> b2n.in[i];
    }

    b2n.out ==> out;
}

template LeftRotate(s) {
    signal input in;
    signal output out;

    component n2b = Num2Bits(32);
    component b2n = Bits2Num(32);

    n2b.in <== in;

    for (var i = 0; i < 32; i++) {
        b2n.in[(i + s) % 32] <== n2b.out[i];
    }

    out <== b2n.out;
}

template Func(i) {
    assert(i <= 64);
    signal input b;
    signal input c;
    signal input d;

    signal output out;

    if (i < 16) {
        component a1 = BitwiseAnd32();
        a1.in[0] <== b;
        a1.in[1] <== c;

        component a2 = BitwiseAnd32();
        component n1 = BitwiseNot32();
        n1.in <== b;
        a2.in[0] <== n1.out;
        a2.in[1] <== d;

        component o1 = BitwiseOr32();
        o1.in[0] <== a1.out;
        o1.in[1] <== a2.out;

        out <== o1.out;
    }
    else if (i >= 16 && i < 32) {
        // (D & B) | (~D & C)
        component a1 = BitwiseAnd32();
        a1.in[0] <== d;
        a1.in[1] <== b;

        component n1 = BitwiseNot32();
        n1.in <== d;
        component a2 = BitwiseAnd32();
        a2.in[0] <== n1.out;
        a2.in[1] <== c;

        component o1 = BitwiseOr32();
        o1.in[0] <== a1.out;
        o1.in[1] <== a2.out;

        out <== o1.out;
    }
    else if (i >= 32 && i < 48) {
        component x1 = BitwiseXor32();
        component x2 = BitwiseXor32();

        x1.in[0] <== b;
        x1.in[1] <== c;
        x2.in[0] <== x1.out;
        x2.in[1] <== d;

        out <== x2.out;
    }
    // i must be < 64 by the assert statement above
    else {
        component o1 = BitwiseOr32();
        component n1 = BitwiseNot32();
        n1.in <== d;
        o1.in[0] <== n1.out;
        o1.in[1] <== b;

        component x1 = BitwiseXor32();
        x1.in[0] <== o1.out;
        x1.in[1] <==c;

        out <== x1.out;
    }
}

// n is the number of bytes
template MD5(n) {

    var s[64] = [7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22,
     5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20,
     4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23,
    6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21];

    var K[64] = [0xd76aa478, 0xe8c7b756, 0x242070db, 0xc1bdceee,
     0xf57c0faf, 0x4787c62a, 0xa8304613, 0xfd469501,
     0x698098d8, 0x8b44f7af, 0xffff5bb1, 0x895cd7be,
     0x6b901122, 0xfd987193, 0xa679438e, 0x49b40821,
     0xf61e2562, 0xc040b340, 0x265e5a51, 0xe9b6c7aa,
     0xd62f105d, 0x02441453, 0xd8a1e681, 0xe7d3fbc8,
     0x21e1cde6, 0xc33707d6, 0xf4d50d87, 0x455a14ed,
     0xa9e3e905, 0xfcefa3f8, 0x676f02d9, 0x8d2a4c8a,
     0xfffa3942, 0x8771f681, 0x6d9d6122, 0xfde5380c,
     0xa4beea44, 0x4bdecfa9, 0xf6bb4b60, 0xbebfbc70,
     0x289b7ec6, 0xeaa127fa, 0xd4ef3085, 0x04881d05,
     0xd9d4d039, 0xe6db99e5, 0x1fa27cf8, 0xc4ac5665,
     0xf4292244, 0x432aff97, 0xab9423a7, 0xfc93a039,
     0x655b59c3, 0x8f0ccc92, 0xffeff47d, 0x85845dd1,
     0x6fa87e4f, 0xfe2ce6e0, 0xa3014314, 0x4e0811a1,
     0xf7537e82, 0xbd3af235, 0x2ad7d2bb, 0xeb86d391];

    var iter_to_index[64] = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15,
     1, 6, 11, 0, 5, 10, 15, 4, 9, 14, 3, 8, 13, 2, 7, 12,
     5, 8, 11, 14, 1, 4, 7, 10, 13, 0, 3, 6, 9, 12, 15, 2,
    0, 7, 14, 5, 12, 3, 10, 1, 8, 15, 6, 13, 4, 11, 2, 9];

    signal input in[n];

    signal inp[64];
    component Pad = Padding(n);

    for (var i = 0; i < n; i++) {
        Pad.in[i] <== in[i];
    }
    for (var i = 0; i < 64; i++) {
        Pad.out[i] ==> inp[i];
    }

    signal data32[16];
    for (var i = 0; i < 16; i++) {
        data32[i] <== inp[4 * i] + inp[4 * i + 1] * 2**8 + inp[4 * i + 2] * 2**16 + inp[4 * i + 3] * 2**24;
    }

    var A = 0;
    var B = 1;
    var C = 2;
    var D = 3;
    signal buffer[65][4];
    buffer[0][A] <== 1732584193;
    buffer[0][B] <== 4023233417;
    buffer[0][C] <== 2562383102;
    buffer[0][D] <== 271733878;

    component Funcs[64];
    signal toRotates[64];
    component SelectInputWords[64];
    component LeftRotates[64];
    component Overflow32s[64];
    component Overflow32s2[64];
    for (var i = 0; i < 64; i++) {
        Funcs[i] = Func(i);
        Funcs[i].b <== buffer[i][B];
        Funcs[i].c <== buffer[i][C];
        Funcs[i].d <== buffer[i][D];

        Overflow32s[i] = Overflow32();
        Overflow32s[i].in <== buffer[i][A] + Funcs[i].out + K[i] + data32[iter_to_index[i]];

        // rotated = rotate(to_rotate, s[i])
        toRotates[i] <== Overflow32s[i].out;
        LeftRotates[i] = LeftRotate(s[i]);
        LeftRotates[i].in <== toRotates[i];

        // new_B = rotated + B
        Overflow32s2[i] = Overflow32();
        Overflow32s2[i].in <== LeftRotates[i].out + buffer[i][B];

        // store into the next state
        buffer[i + 1][A] <== buffer[i][D];
        buffer[i + 1][B] <== Overflow32s2[i].out;
        buffer[i + 1][C] <== buffer[i][B];
        buffer[i + 1][D] <== buffer[i][C];
    }

    component addA = Overflow32();
    component addB = Overflow32();
    component addC = Overflow32();
    component addD = Overflow32();

    // we hardcode initial state because we only
    // process one 512 bit block
    addA.in <== 1732584193 + buffer[64][A];
    addB.in <== 4023233417 + buffer[64][B];
    addC.in <== 2562383102 + buffer[64][C];
    addD.in <== 271733878 + buffer[64][D];

    signal littleEndianMd5;
    littleEndianMd5 <== addA.out + addB.out * 2**32 + addC.out * 2**64 + addD.out * 2**96;

    // convert the answer to bytes and reverse
    // the bytes order to make it big endian
    component Tb = ToBytes(16);
    Tb.in <== littleEndianMd5;

    // sum the bytes in reverse
    var acc;
    for (var i = 0; i < 16; i++) {
        acc += Tb.out[15 - i] * 2**(i * 8);
    }
    signal output out;
    out <== acc;
}

component main = MD5(10);

// The result out =
// "RareSkills" in ascii to decimal
/* INPUT = {"in": [82, 97, 114, 101, 83, 107, 105, 108, 108, 115]} */

// The result is 246193259845151292174181299259247598493

// The MD5 hash of "RareSkills" is 0xb93718dd21d2f5081239d7a16cf69b9d when converted to decimal is 246193259845151292174181299259247598493
```

## 为什么需要 ZK 友好型哈希函数

上面代码生成的 [R1CS](https://www.rareskills.io/post/rank-1-constraint-system) 超过五万二千行，如下图所示。电路规模还有许多优化空间，尤其是可以避免每次使用域元素时都把它转换成 32 位数组。

![展示生成约束数量的截图](https://r2media.rareskills.io/Md5-circom/md5-constraints.png)

然而，MD5（以及其他现代哈希）中的每个机器字都是 32 位，因此与普通代码相比，需要 32 倍数量的信号来表示。

下一章将介绍直接在原生有限域而非 32 位机器字上运算的哈希函数。这些函数会避开 XOR 等昂贵操作，因为此类操作需要把信号分解成比特。
