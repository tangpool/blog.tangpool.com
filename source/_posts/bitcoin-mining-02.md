title: 比特币挖矿（二）：GetWork协议
date: 2015-01-04 17:16:33
tags:
---
CPU挖矿时代并没有延续多久，马上就迎来了GPU时代。GPU由于具有天然的多核优势且功耗可控，很快开始普及，同时算力的快速上涨使得CPU无利可图遭到淘汰。

![GPU显卡矿机](https://cloud.githubusercontent.com/assets/514951/5595084/3f7982fc-92a4-11e4-9401-8926b351eac7.jpg)

由于币价上涨，挖矿变得很有利润，矿工们疯狂扫货高端显卡，导致市面上的高端显卡一卡难求。伴随显卡矿机，诞生了专业挖矿软件[cgmier](https://github.com/ckolivas/cgminer)，至今无数挖矿设备均出现其身影。

### GetWork协议

GetWork是早期挖矿协议，那时ASIC矿机尚未大规模部署，当ASIC时代来临后由于其效率底下目前已淘汰（bitcoind代码已经移除了）。这里翻出来是因为后来的矿机内部依然采用类似GetWork的数据协议，意义重大。

RPC请求:

```
{"method":"getwork","params":[],"id":1}
```

RPC响应:

```
{
  "id": "1", 
  "result": {
    "hash1": "00000000000000000000000000000000000000000000000000000000000000000000008000000000000000000000000000000000000000000000000000010000", 
    "data": "00000001c570c4764aadb3f09895619f549000b8b51a789e7f58ea750000709700000000103ca064f8c76c390683f8203043e91466a7fcc40e6ebc428fbcc2d89b574a864db8345b1b00b5ac00000000000000800000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000080020000",
    "midstate": "e772fc6964e7b06d8f855a6166353e48b2562de4ad037abc889294cea8ed1070", 
    "target": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffff00000000"
    }, 
 "error": null
}
```

首先看`data`段，`data`大小为128字节，块头(Block Head)大小是80字节，那为何是128字节呢？那就得回到SHA256算法。

### SHA256

先看一下SHA256伪代码：

```
Note 1: All variables are 32 bit unsigned integers and addition is calculated modulo 232
Note 2: For each round, there is one round constant k[i] and one entry in the message schedule array w[i], 0 ≤ i ≤ 63
Note 3: The compression function uses 8 working variables, a through h
Note 4: Big-endian convention is used when expressing the constants in this pseudocode,
    and when parsing message block data from bytes to words, for example,
    the first word of the input message "abc" after padding is 0x61626380

Initialize hash values:
(first 32 bits of the fractional parts of the square roots of the first 8 primes 2..19):
h0 := 0x6a09e667
h1 := 0xbb67ae85
h2 := 0x3c6ef372
h3 := 0xa54ff53a
h4 := 0x510e527f
h5 := 0x9b05688c
h6 := 0x1f83d9ab
h7 := 0x5be0cd19

Initialize array of round constants:
(first 32 bits of the fractional parts of the cube roots of the first 64 primes 2..311):
k[0..63] :=
   0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5, 0x3956c25b, 0x59f111f1, 0x923f82a4, 0xab1c5ed5,
   0xd807aa98, 0x12835b01, 0x243185be, 0x550c7dc3, 0x72be5d74, 0x80deb1fe, 0x9bdc06a7, 0xc19bf174,
   0xe49b69c1, 0xefbe4786, 0x0fc19dc6, 0x240ca1cc, 0x2de92c6f, 0x4a7484aa, 0x5cb0a9dc, 0x76f988da,
   0x983e5152, 0xa831c66d, 0xb00327c8, 0xbf597fc7, 0xc6e00bf3, 0xd5a79147, 0x06ca6351, 0x14292967,
   0x27b70a85, 0x2e1b2138, 0x4d2c6dfc, 0x53380d13, 0x650a7354, 0x766a0abb, 0x81c2c92e, 0x92722c85,
   0xa2bfe8a1, 0xa81a664b, 0xc24b8b70, 0xc76c51a3, 0xd192e819, 0xd6990624, 0xf40e3585, 0x106aa070,
   0x19a4c116, 0x1e376c08, 0x2748774c, 0x34b0bcb5, 0x391c0cb3, 0x4ed8aa4a, 0x5b9cca4f, 0x682e6ff3,
   0x748f82ee, 0x78a5636f, 0x84c87814, 0x8cc70208, 0x90befffa, 0xa4506ceb, 0xbef9a3f7, 0xc67178f2

Pre-processing:
append the bit '1' to the message
append k bits '0', where k is the minimum number >= 0 such that the resulting message
    length (modulo 512 in bits) is 448.
append length of message (without the '1' bit or padding), in bits, as 64-bit big-endian integer
    (this will make the entire post-processed length a multiple of 512 bits)

Process the message in successive 512-bit chunks:
break message into 512-bit chunks
for each chunk
    create a 64-entry message schedule array w[0..63] of 32-bit words
    (The initial values in w[0..63] don't matter, so many implementations zero them here)
    copy chunk into first 16 words w[0..15] of the message schedule array

    Extend the first 16 words into the remaining 48 words w[16..63] of the message schedule array:
    for i from 16 to 63
        s0 := (w[i-15] rightrotate 7) xor (w[i-15] rightrotate 18) xor (w[i-15] rightshift 3)
        s1 := (w[i-2] rightrotate 17) xor (w[i-2] rightrotate 19) xor (w[i-2] rightshift 10)
        w[i] := w[i-16] + s0 + w[i-7] + s1

    Initialize working variables to current hash value:
    a := h0
    b := h1
    c := h2
    d := h3
    e := h4
    f := h5
    g := h6
    h := h7

    Compression function main loop:
    for i from 0 to 63
        S1 := (e rightrotate 6) xor (e rightrotate 11) xor (e rightrotate 25)
        ch := (e and f) xor ((not e) and g)
        temp1 := h + S1 + ch + k[i] + w[i]
        S0 := (a rightrotate 2) xor (a rightrotate 13) xor (a rightrotate 22)
        maj := (a and b) xor (a and c) xor (b and c)
        temp2 := S0 + maj
 
        h := g
        g := f
        f := e
        e := d + temp1
        d := c
        c := b
        b := a
        a := temp1 + temp2

    Add the compressed chunk to the current hash value:
    h0 := h0 + a
    h1 := h1 + b
    h2 := h2 + c
    h3 := h3 + d
    h4 := h4 + e
    h5 := h5 + f
    h6 := h6 + g
    h7 := h7 + h

Produce the final hash value (big-endian):
digest := hash := h0 append h1 append h2 append h3 append h4 append h5 append h6 append h7
```

SHA256会将输入数据切分为512bits来处理，若输入长度不是512bits整数倍，则进行补位操作。h0~h7为每轮的输入和输出，k[0..63]为每次处理的常量值。第一轮处理时，h0~h7按照初始默认值进行；第二轮时用第一轮完成后的h0~h7作为输入再次处理；直至处理完所有数据区。

512bits是64字节，那么块头为80字节，需要补齐为1024bits(512的倍数)。SHA256补零规则：首先填1bit的1，再填入N个零，最后8字节为数据总长度。


#### `getwork`字段解读

##### `data`

根据SHA256，我们将`data`分解，那么每个数值的含义就很明确了：

```
00000001  // uint32, 块版本
c570c4764aadb3f09895619f549000b8b51a789e7f58ea750000709700000000  // uint256, 前向块Hash
103ca064f8c76c390683f8203043e91466a7fcc40e6ebc428fbcc2d89b574a86  // uint256, merkle root hash
4db8345b  // uint32, time
1b00b5ac  // uint32, bits
00000000  // uint32, nonce

00000080  // uint32, 补零的首个uint32，大小头翻转后二进制：10000000 00000000 00000000 00000000
000000000000000000000000000000000000000000000000000000000000000000  // 补零padding
000000000000000000000000000000000000000000000000000000000000000000  // 补零padding
0000000080020000  // uint64，翻转后即： 0x00000280 -> 640 表示数据区640bits(80Bytes)
```

##### `midstate`

计算block hash时，我们仅需要变换`nonce`（块头80字节的最后4字节），前面的76字节是不变的，于是提前处理好SHA256首个chunk，前64字节的输出(h0~h7)，32Bytes。通常，DSHA256的ASIC芯片的输入`44Bytes`就是这么得来的：

```
44 Bytes = 32 Bytes // 首轮SHA256计算块头前64字节的输出
         + 12 Bytes // 80(块头) - 64(前64字节) - 4(Nonce为4字节)
```

#### `hash1`

block hash采用的算法是double sha256，sha256输出是32字节，在进行第二次sha256计算时，必须对数据进行补齐：

```
填零的uint256:
0000000000000000 0000000000000000 0000000000000000 0000000000000000

根据sha256 padding规则，首个bit填1，然后填零:
0000000000000000 0000000000000000 0000000000000000 0000000000000000 8000000000000000 0000000000000000 0000000000000000 0000000000000000
                                                                    ^^
最后uint64填长度, 0x0100 = 256bits = 32Bytes
0000000000000000 0000000000000000 0000000000000000 0000000000000000 8000000000000000 0000000000000000 0000000000000000 0000000000000100
                                                                                                                               ^^^^^^^^
                                                                                                                               
最后我们翻转字节序，即得到hash1：
0000000000000000 0000000000000000 0000000000000000 0000000000000000 0000008000000000 0000000000000000 0000000000000000 0000000000010000
                                                                    ^^^^^^^^                                                   ^^^^^^^^
```

其实`hash1`是固定的，可以忽略掉。同理，`data`字段也无需返回那么128字节，76字节即可，后续字节都是固定的自行补充。

#### `target`

即出块的目标值，块Hash值必须小于该值。`target`类型是uint256，在存储时通常转化为`block.bits`(uint32)，即缩短存储空间，又不那么丢失精度。`bits`为4字节，第一个字节表示右移位数，后面三个字节表示值大小。

```
void BitsToTarget(uint32 bits, uint256 & target) {
  uint64 nbytes = (bits >> 24) & 0xff;
  target = bits & 0xffffffULL;
  target <<= (8 * ((uint8)nbytes - 3));
}
```

最高的`target`的`bits`定义为：`0x1d00ffff`，同时意味着最小难度：

```
0x00ffff * 2**(8*(0x1d - 3)) = 0x00000000FFFF0000000000000000000000000000000000000000000000000000
```

前面有32bits的零，矿机与矿池通信时的shares基本单位是`2^32`的原因就在此。比特币于[2009年01月03日](https://blockchain.info/block-height/0)诞生，直到2009年12月18日，全网难度均为该值：`0x1d00ffff`，约7MHash/s，这期间出现了目前为止最大的调整周期：41天。

根据Diff的后面补齐的值分为`pdiff`、`bdiff`。`pdiff`通常称为`pool diff`，用于矿池下发给矿机设置最小难度share。`bdiff`通常称为`bitcoin diff`。若`bits`为`0x1b0404cb`，两者计算如下：

```
// 0x1b0404cb: 
0x0404cb * 2**(8*(0x1b - 3)) = 0x00000000000404CB000000000000000000000000000000000000000000000000

// pdiff
0x00000000FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF /
0x00000000000404CB000000000000000000000000000000000000000000000000 
= 16307.669773817162

// bdiff
0x00000000FFFF0000000000000000000000000000000000000000000000000000 /
0x00000000000404CB000000000000000000000000000000000000000000000000 
= 16307.420938523983
```

### GetWork协议意义

其最大的意义是：数据与计算彻底分离。

在这之前（主要是CPU挖矿）必须运行bitcoind，`getwork`普及之后完全分离，后续进一步促使矿池诞生。`getwork`在ASIC时代早期依然非常流行，比如烤猫的早期矿机都是`getwork`协议工作，也有一些软件将`stratum`协议转为`getwork`来运行矿机。

矿机内部控制器与芯片部分数据流依然类似`getwork`协议，只是精简很多。

### 参考

* GPU显卡矿机图片来源：[https://en.bitcoin.it/wiki/Mining](https://en.bitcoin.it/wiki/Mining)
* 显卡矿场图片来源：[http://www.ltcgouwu.com/archives/102](http://www.ltcgouwu.com/archives/102)
* 由于GetWork已淘汰，其RPC的数据来源：[https://bitcointalk.org/index.php?topic=51281.msg611856#msg611856](https://bitcointalk.org/index.php?topic=51281.msg611856#msg611856)
* SHA256伪代码: [http://en.wikipedia.org/wiki/SHA-2](http://en.wikipedia.org/wiki/SHA-2)
* hash1: [http://bitcoin.stackexchange.com/questions/2906/in-getwork-how-does-one-calculate-the-value-of-hash1](http://bitcoin.stackexchange.com/questions/2906/in-getwork-how-does-one-calculate-the-value-of-hash1)
* Difficulty: [https://en.bitcoin.it/wiki/Difficulty](https://en.bitcoin.it/wiki/Difficulty)