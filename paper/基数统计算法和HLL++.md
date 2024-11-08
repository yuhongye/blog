这是对google的《HyperLogLog in Practice: Algorithm ENgineering of a State of The Art Candinality Estimation Algorithm》的阅读笔记. 首先会介绍LinearCounting和HyperLogLog算法，然后是对论文中各章节的总结.

# 0 LC, LLC and HLL

### 0.1 LinearCounting, 空间复杂度O(N)

设一哈希函数H，其哈希空间为m(最小值0，最大值m-1), 并且服从均匀分布。使用一个长度为m的bitmap, 初始值都为0，对于待预估的集合，使用H得到一个值k, bitmap(k) = 1；遍历完整个集合后，得到bitmap中值为0的个数为u, 则n=-mlog(${u} \over {m}$)​为基数的一个估计值，且为最大似然估计。

其空间复杂度为O(Nmax)，大约是使用bitmap的十分之一，该算法的空间复杂度较高，但在元素数据较少的时候表现优异，用来弥补HyperLogLog在元素数较少时预估偏大的缺陷。

### 0.2 LogLogCounting, 空间复杂度O(log(log(N))))

LogLogCounting空间复杂度为O(log2(log2(N)))), 使得通过KB级内存预估数亿级别的基数成为可能，目前在处理大数据时基数计算问题时，所采用的算法基本是LLC或其变种。

#### 均匀随机化

首先需要选取一个哈希函数H应用于所有元素，然后对哈希值进行基数估计，H必须满足如下条件:

1. H的结果具有很好的随机性，也就是说无论原始集合元素的值分布如何，其哈希结果的值几乎服从均匀分布（D.Knuth已经证明不可能通过一个哈希函数将一组不服从均匀分布的数据映射为绝对服从均匀分布，但是很多哈希函数可以生成近乎服从均匀分布的结果）
2. H的碰撞几乎可以忽略不计
3. H的结果是固定长度的, 设为L. 这里有一个疑问: 目前主流的hash函数都会把所有位数占满吗?

#### 思想来源

设w是待估集合中某个元素经过hash后得到的值，w可以看做一个固定长度为L的bit串，每个bit的编码分别为1, 2, ...L, 设p(w)为w的bit串中第一次出现1的位置，显然`1<=p(w)<=L`。如果我们遍历集合的全部元素后，取$p_{max}$为所有p(w)中最大的那一个，可以把n=${2^{p_{max}}}$作为基数的一个粗糙估计，根据伯努利过程，可以得到下面两条解释:

1. 如果n 远远小于 ${2^{p_{max}}}$, 那么我们得到$p_{max}$为当前值的概率几乎为0
2. 如果n 远远大于 ${2^{p_{max}}}$, 那么我们得到$p_{max}$为当前值的概率也几乎为0

因此可以把${2^{p_{max}}}$作为基数的一个粗糙估计

#### 分桶平均

如果直接使用上面的单一估计量进行基数估计，会由于偶然性导致较大的误差，LLC采用了分桶平均的思想来消除误差。具体来说就是:
1. 将哈希空间分成m份，每份称之为一个桶(bucket)，哈希值w的前p位用来标识桶，则m=$2^p$
2. 剩下的L-p位作为基数估计的比特串，得到每个桶的$p_{max}$
3. $p_{max}$ = 全部桶的$p_{max}$的均值, n = ${2^{p_{max}}}$
4. 偏差修正, ${2^{p_{max}}}$是一个粗糙估计，需要修正，修正后的结果：n = $a_m{2^{p_{max}}}$, 其中${a_m}$=xxx
5. 误差分析，LLC的错误率为: $ 1.3 \over \sqrt{m}$
6. 内存使用分析，m个桶，则每个桶存储$p_{max}$时最多需要：registers = log(L-p) bit,  内存使用量为: mlog(L-p)

LLC算法的优势是内存使用较少，不过当基数较少时，估计误差过大，因此实际使用时都是使用的LLC的改进算法。

### 0.3 HyperLogLog

#### 对LLC的改进

##### 使用调和平均数

HLLC的第一个改进是使用调和平均数替代几何平均数。注意LLC是对各个桶取算数平均数，而算数平均数最终被应用到2的指数上，所以总体来看LLC取得是几何平均数。由于几何平均数对于离群值（例如这里的0）特别敏感，因此当存在离群值时，LLC的偏差就会很大，这也从另一个角度解释了为什么n不太大时LLC的效果不太好。这是因为n较小时，可能存在较多空桶，而这些特殊的离群值强烈干扰了几何平均数的稳定性。

改进后的预估公式：$\hat{n}=\frac{\alpha_m m^2}{\sum{2^{-M}}}$, $\alpha_m=(m\int _0^\infty (log_2(\frac{2+u}{1+u}))^m du)^{-1}$

##### 分段偏差纠正

在HLLC的论文中，作者在实现建议部分还给出了在n相对于m较小或较大时的偏差修正方案。具体来说，设E为估计值：

1. 当${E<={5 \over 2} m}$时，使用LC进行估计。

2. 当${{5 \over 2} m}<E≤{1 \over 30} {2^{32}}$，使用上面给出的HLLC公式进行估计。

3. 当$E>{1 \over 30} {2^{32}}$，估计公式如为n̂ = $-2^{32}log(1-E/2^{32})$

#### 误差分析和内存分析

Error = ${1.04 \over \sqrt{m}}$, 比LLC的$ 1.3 \over \sqrt{m}$要低，内存占用量同LLC。当p=14时，错误率约等于0.81%，内存占用12KB不到。

前面介绍了LC, LLC, HLL 3种基数统计算法，在大数据基数预估方面HLL是最好的算法，在13年的时候google放出了一遍对HLL改进的论文，改进点主要在工程上面，值得一读。下面的部分是对论文的阅读笔记。

# 2 HyperLogLog++

论文的前面部分就说需要使用概率数据结构来进行大数据预估，第3部分给出了基数预估的在实际工程中使用的前提条件：

1. 准确度，在给定的内存要求下面，预估算法应该给出尽可能高的准确度，尤其是对于小集合，预估的值应该要接近真实值
2. 内存高效，预估算法必须高效使用内存，对于小集合内存使用应该要少于用户指定的最大占用内存
3. 可以预估大集合，现在几十亿的集合很常见，必须要对这个量级有准确的预估
4. 预估算法在工程上要易于实现和维护

接近这论文对原始HyperLogLog进行了回顾，接下来就是介绍每一项的改进措施了。在论文中假定了p=14来讨论。



#### 1. 使用64位的hash function

原始的HLL使用32位哈希函数，当预估的集合较大时哈希碰撞率增高，准确的预估变得不可能，使用64位哈希函数可以解决这个问题。HLL有一个很好的性质：内存增长跟哈希函数位数增加是非线性的，哈希函数从32变成64位，每个桶的只增加1bit的存储开销。

在原始论文中，当超过1/30最大值的时候会进行修正，使用64位hash function目前来看没有必要，在现实世界中，64位的哈希函数已经远远够用了。如果真到了那个时候，更好的方式是增加hash function位数，比如变成hash128，这样每个桶的存储从6bit增加到1bit，仅增加15%。

在下一节中，作者通过实验来验证结果的过程中尝试了若干hash function，包括md5, sha1, sha256, murmur3等，它们在效果上几乎一样。

#### 2. 修正较小集合的预估

HLL在预估较小的集合时结果会偏大，所以原始的HLL会对这部分进行修正: 当n < 5/2m时，使用LC算法进行预估，但是实际上这个值给出的并不准确，所以作者指出在实际试验的时候当n>5/2m=40960的时候，原始的HLL的错误率会有一个很高的尖峰。作者指出当n > 5m后，HLL的预估是比较准确的，对于<5m的情况，需要修正。论文通过观察偏差和真实值的情况，选择了200个插入点，使用k-nearst neighbor来预估偏差，作者把这个方法叫做: Bias Corrected Raw Estimate of ${HLL_{64bit}}$，作者在附录中给出了p在不同值的情况下的插入点。

LC算法对于较小的集合预估小姑较好，因此在这个区间段内使用LC，最终的算法根据n的大小分成了3段:

1. 小于LC Threshold，使用LC预估。作者给出了p在不同长度下的阈值，12：3100， 13：6500， 14：11500，15：20000， 16：50000。
2. 小于5m时，使用Bias Corrected Raw Estimate 方法
3. 使用原始的HLL，这里没有对较大值的修正，因为在64位hash function下不会触碰到这个天花板。

#### 3. 稀疏表示，错误率显著低

在HLL中每个桶需要6bit，当n << m时，大部分桶都是空的，因此可以不用存储它们。为了节省内存，针对这种情况论文作者提出了稀疏表示法，使用一个sort list存储每个存在的桶和它最高位: (idx, p(w))。其中idx占用14位，p(w)占用6位，因此完全可以把这个pair使用一个int32来存储，其中高位存储idx, 低位存储p(w)。当使用稀疏表示超过了最大限制内存时就回退到dense representation。

为了加快插入速度，google的实现中使用了两种数据结构，sorted list用来保存已有的数据，temp set用来执行插入，并定期的将temp set合并到sorted list中。因为是有序的list，merge操作可以一次遍历完成。

##### 3.1 更高的准确度

HLL的error = $1.04 \over \sqrt{m}$, m= $2^p$,增大p可以降低错误路。

在稀疏表示中，p(w)占用6位，对于int32而言，还剩余26位可以用来表示idx，因此可以增加p的位数到$p^{'}$，并且可以从$p^{'}$回退到p的dense representation。

$p^{'}$的选择是精准度和内存的trade off，$p^{'}$越大越精准，但是会越快达到6m的阈值，然后回退到normal mode。论文中选择的是25，这是因为25+6对于int32还富裕1bit，这1bit在下文用来干别的事情了。

##### 3.2 压缩稀疏表示

这部分是关于如何高效的压缩sorted int list，可以使用varint算法，同时可以使用difference encoding，就是只存差值，这一部分没什么好说的，之前做列存时很熟了，对于差值还可以使用packed方法进一步压缩。一切都得益于sorted list的结构，可以让这一部分也实现的高效，

##### 3.3 Encoding Hash Values

稀疏表示的元素个数小于LC的Threshold，因此在稀疏表示中可以退化为如何高效实现LC算法。在LC算法中，我们只关心m和去重出现的桶数，高$p^{'}$表示的就是桶idx，因此在LC算法中没必须存$p^{'}(w)$，只有回退到dense representation， 并且只有当${<x_{63-p},…,x_{64-p^{'}}}>$ 都是0时才需要存储$p^{'}(w)$，因为如果${<x_{63-p},…,x_{64-p^{'}}}>$中不全为0，则最高位1一定出现在这几位中。并且这几位全部是0的概率: ${2^{p-p^{'}}}$，因为hash function服从均匀分布，因此这几位任一个是0的概率为${1 \over 2}$, 都为0的概率就是${2^{p-p^{'}}}$。

因此可以继续压缩存储，将最低位作为区分：

1. 如果${<x_{63-p},…,x_{64-p^{'}}}>$ 全是0，需要存储$p^{'}(w)$: ${<x_{63},…,x_{64-p^{'}}}>$|$<p^{'}(w)>$|1
2. 否则，不需要存储$p^{'}(w)$, ${<x_{63},…,x_{64-p^{'}}}>$|0

这样int32的全部bit都派上用场了。至此全部的优化都完成了，这里得到的就是HLL++

##### 3.4 为什么要拼命压缩存储

压缩越高效，则sparse representation可以存储的元素个数越多，在这个区间内的准确率也就越高。

使用sparse representation每个element占用32bit，使用varint后每个占用19.49bit，加上difference encoding后每个占用11.75，压缩了hash value后每个占用8.12bits。作者给出了一个表来表示在不同压缩情况下，sparse representation可以表示的元素数，p=14, m=16384：

* 原始的sparse representation： 3072
* 使用var int                                :    5043.91
* 使用diffrence encoding          :    8366.45
* HLL++                                        :    12107



论文最后作者讲了一个HLL++是怎么用在Dictionary encoding的，没看懂。