title: 比特币挖矿（一）：1 CPU 1 VOTE
date: 2015-01-02 11:22:17
updated: 2015-01-06 10:00:00
tags:
---

作者: 潘志彪

### 工作量证明

挖矿即完成工作量证明（Proof of Work），简称POW。POW机制最核心的是哈希(HASH)函数，hash函数最大的特点是：不同输入，随机输出。

1. 对于特定要求的输出，只能大量枚举尝试不同的输入来得到，没有捷径无法偷懒
1. 无论输入尝试了几次还是几亿次，验证者只需对结果执行一次该过程即可

举个例子，学校共1万人且都不同姓名，设计一个Hash函数，输出范围是：[1, 100]。

```
hash('张三') = 23
hash('李四') = 9
hash('王麻') = 82
```

欲找到输出等于42（1-100任意一个值都一样）的姓名，理论需要尝试100次。也意味着：1. 尝试100次不代表必然找到该姓名；2. 尝试100次也可能找出多个输出为42的姓名；3. 无论以什么遍历顺序，效率是一样的。

比特币的区块(Block)用于工作量证明，准确的说是区块头部(Block Head)的80字节。一个完整区块由两部分组成：Head + Body。Head存放固定80字节的信息，Body部分存放交易数据。

#### 块头部构成

字段名 | 含义  | 大小(字节)
:------|------|----------:
Version | 版本号 | 4
hashPrevBlock | 上一个block hash值 | 32
hashMerkleRoot | 上一个block产生之后至新block生成此时间内，<br/>交易数据打包形成的Hash | 32
Time | Unix时间戳 | 4
Bits | 目标值，即难度 | 4
Nonce | 随机数 | 4

块哈希算法：对这80字节进行`double sha256`计算。输出是32字节，通常见到的是以16进制显示的64位字符串。

```
DSHA256(block head) = Block Hash
```

### 挖矿过程

结合块头部构成，梳理一下挖矿的计算过程。

* 第一步、构造Merkle Root Hash，该值由块中的交易决定
   1. 收集全网未确认交易。
   1. 构造Coinbase交易。每个块有且只有一个coinbase交易，记录块奖励和交易费
   1. 逐层递归运算。左右节点合并为一个父节点，最终得到一个节点即Merkle Root
* 第二步、填入其他字段，这些字段无需复杂计算。`version`通常固定不变，升级时才会更新。`prev block hash`上一个块哈希，同一高度固定不变，拷贝过来即可。`time`当前时间戳。`bits`目标值，由前一批2016个块决定，难度调整之前固定不变。`nonce`填零即可。至此块头部构造完成。
* 第三步、计算块哈希
   1. nonce增一
   1. 计算块哈希
   1. 检测是否达到难度要求？
      1. 达到则广播块，进入下一个块的计算过程
      1. 未达到，则继续1的过程
* 第四步、一定时间未出块，则重新构造merkle root，保证块及时收纳新的交易；调整time，写入当前时间戳，保证时间戳不至于太旧。

### CPU时代

`nonce`大小为4字节(2^32)，其空间为42亿，或者说4G。这在CPU时代是完全够用的，单核计算Double SHA256的速度通常在[0.1M ~ 4M](https://en.bitcoin.it/wiki/Non-specialized_hardware_comparison#CPUs.2FAPUs)之间，核数越多越具优势。

bitcoind源码中CPU挖矿的代码片段, [bitcoin/src/miner.cpp](https://github.com/bitcoin/bitcoin/blob/v0.9.3/src/miner.cpp)（其中nonce遍历完2字节就会去重构块）：

```
//
// ScanHash scans nonces looking for a hash with at least some zero bits.
// It operates on big endian data.  Caller does the byte reversing.
// All input buffers are 16-byte aligned.  nNonce is usually preserved
// between calls, but periodically or if nNonce is 0xffff0000 or above,
// the block is rebuilt and nNonce starts over at zero.
//
unsigned int static ScanHash_CryptoPP(char* pmidstate, char* pdata, char* phash1, char* phash, unsigned int& nHashesDone)
{
    unsigned int& nNonce = *(unsigned int*)(pdata + 12);
    for (;;)
    {
        // Crypto++ SHA256
        // Hash pdata using pmidstate as the starting state into
        // pre-formatted buffer phash1, then hash phash1 into phash
        nNonce++;
        SHA256Transform(phash1, pdata, pmidstate);
        SHA256Transform(phash, phash1, pSHA256InitState);
 
        // Return the nonce if the hash has at least some zero bits,
        // caller will check if it has enough to reach the target
        if (((unsigned short*)phash)[14] == 0)
            return nNonce;
 
        // If nothing found after trying for a while, return -1
        if ((nNonce & 0xffff) == 0)
        {
            nHashesDone = 0xffff+1;
            return (unsigned int) -1;
        }
        if ((nNonce & 0xfff) == 0)
            boost::this_thread::interruption_point();
    }
}
```

过程非常简单，就是反复构造块，尝试nonce计算。CPU N个核就启动N个线程，所以挖矿时CPU通常100%。很快出现软件[cpuminer](https://github.com/jgarzik/cpuminer), 使得一个bitcond节点可以支持多台电脑挖矿，众多比特币爱好者纷纷购置高规格电脑开始挖矿，正如中本聪(Satoshi Nakamoto)在论文里所预测的理想数字民主：1 cpu 1 vote！


### 参考
* [http://en.wikipedia.org/wiki/Proof-of-work_system](http://en.wikipedia.org/wiki/Proof-of-work_system)
* [https://en.bitcoin.it/wiki/Block_hashing_algorithm](https://en.bitcoin.it/wiki/Block_hashing_algorithm)
* [https://en.bitcoin.it/wiki/Hashcash](https://en.bitcoin.it/wiki/Hashcash)