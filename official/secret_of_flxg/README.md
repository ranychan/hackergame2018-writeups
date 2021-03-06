# FLXG 的秘密
### 来自未来的漂流瓶
##### TL;DR
六十四卦那些卦象，你们看着不觉得就像二进制吗..。
##### 详解
下下来直接打开发现乱码。不知道为啥编辑器不觉得这是 UTF-8。切换到 UTF-8 的编码，发现果然一堆六十四卦的名词。

无论怎么编码的，第一步肯定是把不同卦分离开来。六十四卦的名称比较杂乱，是变长的 CISC 架构，代码里面的六十四卦卦名要老老实实写出来。

接下来就是脑洞时间了。六十四卦每一卦都有 6 个 bits 的信息，而一般的数据都是以 8 个 bits 为单位。因此，我们看一下长度，发现可以被 8 整除，这进一步验证了我们的猜想。接下来，就是考虑如何把一个卦象转化为 6 个 bits。

有三类显而易见的情况:
* 每个卦象自下而上，阴阳对应 0 和 1，这就是两种可能
* 每个卦象自上而下，阴阳对应 0 和 1，这又是两种可能
* 卦象以先天六十四卦顺序，也是 Unicode 字符集中的顺序编码

写出来脚本跑一跑，发现第二种情况能产生一个 gzip 的文件。解压时提示文件损坏，查看文件末尾即可得到 flxg。
### 难以参悟的秘密
##### TL;DR
Merkle Hellman Knapsack Cryptosystem
##### 详解
本题是一道逆向。

解压得到一个可执行文件和一堆动态链接库。拖进 IDA 里发现，程序会读取 passkey.txt 的内容，然后通过调用动态链接库的函数进行校验。最后经过一番处理，每 8 行变成一个大整数。然后根据一个 128bit 的数的某一位进行求和。最后判断是否结果等于最后一个大整数。这实际上是一个 Merkle Hellman Knapsack Cryptosystem。

二进制里重要函数都已经标注出来。一旦我们弄清楚程序的意图，就可以继续做下去了。思路非常清晰: 先要恢复 passkey.txt 的内容，进而得到每个大整数。然后求解 flxg。

第一步，需要选手批量处理动态链接库中的代码。动态链接库里面的验证基本可以总结为 kx+b=x，通过将 k 和 b 提取出来，可以求解 x。而 k 和 b 两个数字在 lock 函数中偏移固定，可以通过能处理 ELF 的 python 库或二进制分析框架提取出来。

第二步，我们得到了 passkey.txt 应有的内容。现在我们需要还原 128 个大整数。大致有两种思路，一种是动态运行程序，通过修改程序的代码或者 Hook 或者 DBI 或者调试器脚本的方法，可以得到这些大整数。另一种是通过逆向，自己重现相应的算法。这道题里面使用了一个不常见的 Hash 算法 -- JH，很难识别出来。并且代码中大量使用 SSE，很难手动实现。但是可以通过将可执行文件转为动态链接库的方法导出相关函数，从而直接运行程序的算法。

第三步，Low Density attack。这实际上是 1984 年的攻击了。需要找到论文简单复现一下即可。比如 http://www4.ncsu.edu/~smsulli2/MA437_Fall2017/knapsack.pdf。其核心思想是构造出一个 Lattice，这些向量加起来和为 0。然后就变成了格点规约问题了。通过 LLL 算法很容易求出解。

(此处省略丑陋的 Mathematica 代码)

本题代码见 flxg.c 与 lock.c。代码实际上不能直接编译，因为缺少 jh.h (只是一个 hash 算法的实现) 和 lock.h (由脚本生成)。不过大致流程比较清晰。可参看。