# 提纲
[toc]

# Compaction基础知识
## Compaction配置

这里将列出那些影响Compaction的配置项：

- Options::compaction_style - 当前Rocksdb支持两种类型的Compaction算法 - Universal style compaction和Level style compaction. 可以被设置为kCompactionStyleUniversal或者kCompactionStyleLevel。如果设置为kCompactionStyleUniversal，则可以通过设置Options::compaction_options_universal来配置Universal style compaction的某些参数。

- Options::disable_auto_compactions - 禁用自动Compaction，手动Compaction仍然是可以的。

- Options::compaction_filter - 允许应用在compaction过程中来修改/删除某个键值对。 如果用户要求每个compaction使用自己的compaction filter，则必须提供compaction_filter_factory。用户只能指定compaction_filter或者compaction_filter_factory中的一种。

- Options::compaction_filter_factory - 一个提供compaction filter对象的工厂，允许应用在compaction过程中来修改/删除某个键值对。

其它一些影响compaction性能的配置项:

- Options::access_hint_on_compaction_start - 指定compaction开始之后文件访问模式。默认值：NORMAL。

- Options::level0_file_num_compaction_trigger - 触发level-0 compaction的文件数目。 负数表示level-0 compaction将不以文件数目作为触发方式。

- Options::target_file_size_base and Options::target_file_size_multiplier - target_file_size_base表示level-1上每个文件的大小。level L上每个文件的大小可以通过(target_file_size_base * (target_file_size_multiplier ^ (L-1)))来计算。比如，如果target_file_size_base是2MB，target_file_size_multiplier是10，那么level-1上每一个文件为2MB，level-2上每一个文件为20MB，level-3上每一个文件为200MB。 默认的target_file_size_base是2MB， 默认的 target_file_size_multiplier是1。

- Options::max_compaction_bytes - compaction后的文件的最大大小。在compaction过程中，如果compaction后的大小超过该值，将不再扩充参与compaction的较低层级的文件集合。
- Options::max_background_compactions - 最大并发compaction任务数。

- Options::compaction_readahead_size - 如果不为0，则在compaction过程中执行较大的读。 如果在磁盘上运行Rocksdb，则至少应该被设置为2MB。如果没有采用Direct IO，则Rocksdb强制设置为2MB。

Compaction也可以手动触发，参考[Manual Compaction](https://github.com/facebook/rocksdb/wiki/Manual-Compaction)

可以在rocksdb/options.h中了解更多关于Compaction的配置选项。

## Leveled style compaction

### 文件组织

硬盘上文件以多个层组织，称为level-1，level-2，等，或者简称为L1，L2等。有一个特殊的level-0（或者简称L0）包含从memtable刷新下来的文件。每一层（除level-0之外）都存放排序的数据，因为每一个SST file内部keys是排序的。

![image](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/level_structure.png)

在每一层中（除level-0之外），数据按照范围划分成多个SST files：

![image](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/level_files.png)

为了确定某个key在level中的位置，首先二分查找所有SST files的start/end key，以找出哪个SST file包含该key，然后在该SST file内部二分查找以确定key的精确位置。

除level-0以外的所有层都有自己的目标大小，Compaction过程中必须确保每一层的大小不超过该目标大小。通常，每一层的目标大小是呈指数增长的：

![image](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/level_targets.png)

### 合并

当L0中文件数目达到level0_file_num_compaction_trigger，L0中的文件将被合并到L1中。通常而言，会挑出L0中的所有文件来进行合并，因为这些文件之间存在交叠：

![image](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/pre_l0_compaction.png)

合并之后，L1的大小可能会超过该层的目标大小：

![image](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/post_l0_compaction.png)

这种情况下，将从L1中挑选至少一个文件与L2中存在交叠的文件进行合并。合并后的文件将保存于L2：

![image](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/pre_l1_compaction.png)

如果合并导致下一层的大小超过目标大小，则跟上述过程一样，从该层中挑选至少一个文件与它的下一层进行合并：

![image](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/post_l1_compaction.png)

接着，

![image](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/pre_l2_compaction.png)

接着，

![image](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/post_l2_compaction.png)

如果需要，多个合并可以并发进行:

![image](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/multi_thread_compaction.png)

最大合并并发数由max_background_compactions控制.

L0向L1的compaction不可以与其他level compaction并行，在某些情况下，这可能会成为合并瓶颈，在这些情况下，用户可以设置max_subcompactions大于1，这样Rocksdb就会尝试进行分区，并采用多个线程来执行。

![image](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/subcompaction.png)

### Compaction优先级

目前Rocksdb提供了一个选项[options.compaction_pri](https://github.com/facebook/rocksdb/blob/d6c838f1e130d8860407bc771fa6d4ac238859ba/include/rocksdb/options.h#L83-L93)，支持设定3种不同算法来挑选合并的文件。

为什么需要不同的算法来挑选文件呢，因为影响文件挑选的因素很多，尚没有较好的办法在这些因素之间平衡，因此将这些算法暴露给用户，由用户进行选择。这些是考虑因素：

**写放大**

挑选了一个key分布范围不集中的文件去合并的代价是巨大的，因为在下一个level中与之存在交叠的文件个数可能更多，比如每个文件大小是100MB，L2中挑选的1个文件和L3中的8个文件存在交叠，则需要rewrite约800MB的数据到L3中，如果L2中挑选的这1个文件和L3中的12个文件存在交叠，则需要rewrite月1200MB的数据到L3中，由于key分布范围不集中，后一种情况带来50%的额外的写。

为了应对这种情况，Rocksdb提供了kOldestSmallestSeqFirst这一compaction策略，优先选择key分布范围集中的file去compact。在这种策略下，选择包含最老更新的文件，该文件通常包含分布范围最集中的key。在写操作均匀分布的情况下，如果希望减少写放大，就可以通过设置options.compaction_pri=kOldestSmallestSeqFirst来采用该策略。

**小数据集优化**

在考虑写放大这一因素的时候，假设这些更新是均匀分布的。但是在多数情况下并非如此，某些keys被频繁访问，而另外一些keys则很少被访问，在这种情况下，避免频繁访问的keys被合并到更高的层将会减少写放大，同样减少空间放大。

为了应对这种情况，Rocksdb提供了kOldestLargestSeqFirst这一compaction策略，优先选择冷数据文件去compact。在这种策略下，选择最近的更新最老的文件（pick a file whose latest update is the oldest），该文件在较长时间内都没有更新了，通常该文件就是最冷的文件。在小范围keys上频繁覆盖写的情况下，就可以通过设置options.compaction_pri=kOldestLargestSeqFirst来采用该策略。

**尽早丢弃带删除标记的记录**
越早合并被删除的keys到最后一层，硬盘空间就可以越早被回收，空间利用率越高。默认的compaction策略kByCompensatedSize考虑了该情况，优先选择被删除数据多的file去compact。如果某个文件中删除的数目超过插入的数目，该文件就很可能作为compaction候选，某个文件中删除数目超过插入数目越多，该文件就越可能被选择。

### 挑选参与合并的level

当多个levels都触发了合并条件，Rocksdb需要挑选出哪个level优先执行compaction。Rocksdb采用打分机制来挑选，每一个level都有一个score：

- 对于非0 level，score = 该level文件的总长度 / 阈值。已经正在做compaction的文件不计入总长度中。

- 对于L0，score = max{文件数量 / level0_file_num_compaction_trigger， L0文件总长度 / max_bytes_for_level_base} 并且 L0文件数量 > level0_file_num_compaction_trigger。 

计算出每一层的分数之后，首先挑选出分数最大的那一层进行合并。

### 从选定的level中挑选参与合并的文件集合

确定合并哪一层之后，就要确定挑选哪些文件来进行合并了，过程如下：
1. 假设当前要合并的input level是Lb，那么合并的output level是Lo = Lb + 1。

2. 根据不同的合并优先级，首先挑选出具有最高优先级的第一个文件进行合并，如果该文件或者它在Lo层的父文件（即在Lo层与该文件具有交叠的key区间的文件）正在另一个合并任务中，则跳过该文件，查找下一个具有次高优先级的文件， 直到找到一个候选文件，将这个文件添加到合并的inputs集合中。

3. 继续扩充inputs集合，直到确信inputs集合中的文件和它周围的文件在边界处具有清晰的界限（即两者交界处不存在交叠）。这将确保合并过程中不遗漏任何一个key。以5个具有不同key区间的文件为例：

    f1[a1 a2] f2[a3 a4] f3[a4 a6] f4[a6 a7] f5[a8 a9]

    如果在第2步挑选了f3，那么在第3步，将inputs集合从{f3}扩充到{f2, f3, f4}，因为f2和f3的边界是连续的，同样f3和f4的边界也是连续的。那么为什么会存在两个文件具有相同的user key边界呢？是因为Rocksdb在文件中存放的是由user key，sequence number和记录类型组成的InternalKey，文件中可能存在多个具有相同user key的InternalKey。合并过程中，这些具有相同user key的InternalKey将会被合并。

4. 检查inputs集合中的文件是否和当前正在进行合并的文件之间存在交叠，如果存在交叠则只能终止本次挑选。

5. 在Lo上找出所有与input集合中的文件存在交叠的文件，并且和第3步一样不断扩充它们直到确信这些文件和它周围的文件在边界处具有清晰的界限（即两者交界处不存在交叠）。如果这些文件中的任何一个正在compaction过程中，则终止本次挑选。否则，将这些文件放入output_level_inputs集合中。

6. 一种可选的优化是：如果可以继续扩充inputs集合中的文件，而不会增加output_level_inputs集合中文件个数，则继续扩充inputs集合中的文件。如果这个扩充过程导致Lb包含了具有某些user key的InternalKey，同时排除了一些其它的具有相同user key的InternalKey，则不扩充。举个例子，考虑如下情形：

    Lb: f1[B E] f2[F G] f3[H I] f4[J M]
    Lo: f5[A C] f6[D K] f7[L O]
     
    如果在第2步挑选了f2，那么将合并f2(inputs)和f6(output_level_inputs)，但是实际上可以安全的合并f2，f3和f6，与此同时并没有扩充Lo层。

7. inputs和output_level_inputs中的文件作为level compaction的候选。


### 每一个level的文件大小阈值

**level_compaction_dynamic_level_bytes is false**

Target_Size(L1) = max_bytes_for_level_base

Target_Size(Ln+1) = (Target_Size(Ln) * max_bytes_for_level_multiplier * max_bytes_for_level_multiplier_additional[n].max_bytes_for_level_multiplier_additional)，默认的，所有的max_bytes_for_level_multiplier_additional都是1。

例如： 
max_bytes_for_level_base = 16384 
max_bytes_for_level_multiplier = 10 
max_bytes_for_level_multiplier_additional = 1 
那么每个level的触发阈值为 L1, L2, L3 and L4 分别为 16384, 163840, 1638400, and 16384000。

**level_compaction_dynamic_level_bytes is true**
最后一个level（num_levels-1）的文件长度总是固定的。

Target_Size(Ln-1) = Target_Size(Ln) / max_bytes_for_level_multiplier 
如果计算得到的值小于max_bytes_for_level_base / max_bytes_for_level_multiplier， 那么该level将不存放任何文件，L0做compaction时将保持该level为空，并跳过该level直接merge到较高level中第一个应当存放文件的level上。 

例如： 
max_bytes_for_level_base = 1G 
num_levels = 6 
level 6 size = 276G 
那么从L1到L6的触发阈值分别为：0， 0， 0.276G， 2.76G， 27.6G，276G。 

![image](https://github.com/facebook/rocksdb/raw/gh-pages-old/pictures/dynamic_level.png)

这样分配，保证了稳定的LSM-tree结构。并且有90%的数据存储在最后一层，9%的数据保存在倒数第二层。

## Universal style compaction
### 概览

Universal Compaction主要用于减少写放大（lower write amplification），在读放大和空间放大之间折中（trading off read amplification and space amplification）。

采用Universal Compaction的时候，所有的SST文件都以"sorted runs"的形式组织。关于这些sorted runs有以下特点：
- 一个sorted run中包含某个时间段内产生的所有数据；
- sorted runs按照更新时间来分布；
- 不同的sorted runs对应的时间段绝不会重叠；
- 合并只能在两个或者多个具有相邻时间段的sorted runs上进行；
- 合并后产生的新的sorted run对应的时间段是参与合并的两个或者多个sorted run的时间段的合并；
- 任何一次合并之后，所有的sorted runs对应的时间段也绝不会重叠；
- 一个sorte run可能对应L0中的一个文件，或者其它level中的所有文件（也就是说，L0中的每一个文件都对应一个sorted run，其它level中的所有文件对应一个sorted run）；

### 数据存放和布局
#### Sorted Runs
根据前面的描述，sorted runs按照更新时间来分布，并且要么以L0中的文件的形式存在，要么以其它level中的所有文件的形式存在。下面给出一个典型的sorted runs的布局：

    Level 0: File0_0, File0_1, File0_2
    Level 1: (empty)
    Level 2: (empty)
    Level 3: (empty)
    Level 4: File4_0, File4_1, File4_2, File4_3
    Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7

具有较大编号的Level包含较老的sorted run，本示例中总共包含5个sorted run：L0中3个文件分别对应3个sorted run，L4和L5则分别对应1个sorte run。L5是最老的sorted run，L4则是较新的sorted run，而L0中的文件是最新的sorted run。

#### Placement of Compaction Outputs
Compaction is always scheduled for sorted runs with consecutive time ranges and the outputs are always another sorted run. We always place compaction outputs to the highest possible level, following the rule of older data on levels with larger numbers.
具有相邻时间段的sorted runs将被合并，产生新的sorted run。产生的新的sorted run将按照“具有较大编号的Level包含较老的sorted run”的规则被存放在尽可能高的level中（注意，这里“较老的sorted run”是根据sorted run中数据对应的时间段来划分的，而不是sorted run产生的时间）。

举例说明，假设初始状态下sorted runs为：
    Level 0: File0_0, File0_1, File0_2
    Level 1: (empty)
    Level 2: (empty)
    Level 3: (empty)
    Level 4: File4_0, File4_1, File4_2, File4_3
    Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7

如果将所有的sorted runs进行合并，产生的新的sorted runs将会被存放于L5中：

    Level 5: File5_0', File5_1', File5_2', File5_3', File5_4', File5_5', File5_6', File5_7'
    
在初始状态下，如果将File0_1，File0_2合并到L4中，产生的新的sorted run将被存放于L4中：

    Level 0: File0_0
    Level 1: (empty)
    Level 2: (empty)
    Level 3: (empty)
    Level 4: File4_0', File4_1', File4_2', File4_3'
    Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7
    
在初始状态下，如果将File0_0，File0_1和File0_2合并，产生的新的sorted run将被存放于L3中：

    Level 0: (empty)
    Level 1: (empty)
    Level 2: (empty)
    Level 3: File3_0, File3_1, File3_2
    Level 4: File4_0, File4_1, File4_2, File4_3
    Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7
    
在初始状态下，如果将File0_0和File0_1合并，产生的新的sorted run将被存放于L0中：

    Level 0: File0_0', File0_2
    Level 1: (empty)
    Level 2: (empty)
    Level 3: (empty)
    Level 4: File4_0, File4_1, File4_2, File4_3
    Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7

### 挑选参与合并的sorted runs
假设存在如下的sorted runs：

    R1, R2, R3, ..., Rn
    
R1包含最近的更新，而Rn包含最老的更新（注意：R1到Rn的sorted runs是以各sorted run内部数据的年龄来划分的，而不是sorted run自身产生的时间）。

按照下述步骤挑选参与合并的sorted runs:

**前置条件:**
n >= options.level0_file_num_compaction_trigger

如果不满足该前置条件，则不会触发合并。在满足前置条件的情况下，按优先级顺序触发以下合并。

**1. 由空间放大触发的compaction（Compaction Triggered by Space Amplification）**

如果预估的空间放大超比例超过(options.compaction_options_universal.max_size_amplification_percent / 100)，则所有文件进行一次compaction，即所谓的full compaction。

空间放大比例是通过以下方式进行预估：
    size amplification ratio = (size(R1) + size(R2) + ... size(Rn-1)) / size(Rn)
    
采用这种方式来预估空间放大比例是基于以下原因：在一个稳定的数据库系统中，删除速率和插入速率应该接近，意味着除了Rn以外的所有sorted runs中应当包含近似相同数目的插入和删除，在将R1，R2 ... ，Rn-1，Rn合并之后，删除和插入相互抵消，新的sorted run的大小也应该等于Rn，也就是真实有效数据的大小（不存在应当被删除但尚未删除的数据，也不存在关于同一个key的重复的数据），所以以Rn作为空间放大预估的基准。

为了方便理解，下面举个例子，并作如下假设：
- options.compaction_options_universal.max_size_amplification_percent = 25，即数据库中文件占用的空间不超过真实有效数据所需空间的125%倍；
- 只有空间放大会触发合并；
- options.level0_file_num_compaction_trigger = 1；
- 每次从memtable中flush下来的数据占恰好占用1个大小为1的文件；
- 合并后的文件大小总是等于参与合并的文件的总大小；

经过两次flush之后，将有两个sorte runs（每个sorted run对应1个大小为1的文件），预估的空间放大比例为1 / 1 = 1，而设定的最大空间放大比例为(options.compaction_options_universal.max_size_amplification_percent / 100) = 25%，所以需要在L0执行一次full compaction，L0中2个sorted runs合并之后产生一个新的sorted run（包含2个大小分别为1的文件，总大小为2）：

    1 1  =>  2
    
接着，又一个新的memtable flush在L0中产生了一个新的sorted run（包含1个大小为1的文件）：

    1 2   =>  3

因为1 / 2 > 25%，所以这2个sorted runs将合并产生一个新的sorted run（包含3个大小为1的文件，总大小为3）：

    1 3  =>  4

接着，又一个新的memtable flush在L0中产生了一个新的sorted run（包含1个大小为1的文件），但是这2个sorted run将不会被合并：

    1 4

因为1 / 4 <= 25%。

接着，又一个新的memtable flush在L0中产生了一个新的sorted run（包含1个大小为1的文件），又会触发合并：

    1 1 4  =>  6
    
因为(1 + 1) / 4 > 25%.

后续则按照以下方式继续运行：

    1
    1 1  =>  2
    1 2  =>  3
    1 3  =>  4
    1 4
    1 1 4  =>  6
    1 6
    1 1 6  =>  8
    1 8
    1 1 8
    1 1 1 8  =>  11
    1 11
    1 1 11
    1 1 1 11  =>  14
    1 14
    1 1 14
    1 1 1 14
    1 1 1 1 14  =>  18

**2. 基于大小比例的触发（Compaction Triggered by Individual Size Ratio）**

通过以下方式计算基于大小比例的触发：

     size_ratio_trigger = 1 + options.compaction_options_universal.size_ratio / 100

从R1开始，如果size(R2) / size(R1) <= size_ratio_trigger，那么(R1, R2)将被合并。继续判断R3是否能加入，如果size(R3) / size(R1 + R2) <= size_ratio_trigger，那么(R1, R2, R3)将被合并。对于R4，进行相同的判断，直到遇到size(Rk) / size(R1 + R2 + ... + Rk-1) > size_ratio_trigger的情况发生为止。

为了方便理解，下面举个例子，并作如下假设：
- options.compaction_options_universal.size_ratio = 0，那么size_ratio_trigger = 1；
- 只有空间放大会触发合并；
- options.level0_file_num_compaction_trigger = 1；
- 每次从memtable中flush下来的数据占恰好占用1个大小为1的文件；
- 合并后的文件大小总是等于参与合并的文件的总大小；

Now we start with only one file with size 1. After another mem table flush, we have two files size of 1, which triggers a compaction:
经过两次flush之后，将有两个sorte runs（每个sorted run对应1个大小为1的文件），因为(1 / 1) <= 1，触发一次合并：

    1 1  =>  2
    
接着，又一次新的memtable flush，在L0中产生了一个新的sorted run（包含1个大小为1的文件），不会触发合并：

    1 2  (no compaction triggered)
    
因为2 / 1 > 1。

接着，又一次新的memtable flush，在L0中产生了一个新的sorted run（包含1个大小为1的文件），触发所有文件的合并：

    1 1 2  =>  4
    
因为(1 / 1) >=1 and 2 / (1 + 1) >= 1.

后续则按照以下方式继续运行：

    1 1  =>  2
    1 2  (no compaction triggered)
    1 1 2  =>  4
    1 4  (no compaction triggered)
    1 1 4  =>  2 4
    1 2 4  (no compaction triggered)
    1 1 2 4 => 8
    1 8  (no compaction triggered)
    1 1 8  =>  2 8
    1 2 8  (no compaction triggered)
    1 1 2 8  =>  4 8
    1 4 8  (no compaction triggered)
    1 1 4 8  =>  2 4 8
    1 2 4 8  (no compaction triggered)
    1 1 2 4 8  =>  16
    1 16  (no compaction triggered)
    ......

在合并过程中，每一次合并过程中参与合并的的sorted runs的数目必须介于[options.compaction_options_universal.min_merge_width, options.compaction_options_universal.max_merge_width]之间。

3. 基于sorted runs数目的触发（Compaction Triggered by number of sorted runs）

如果第1和第2种情况都没有compaction，则强制选择前N个文件进行合并，以确保合并之后sorted runs的数目不超过options.level0_file_num_compaction_trigger + 1。


### 举例
[Universal Style Compaction Example](https://github.com/facebook/rocksdb/wiki/Universal-Style-Compaction-Example)

### 并发compaction（Parallel Compaction）

如果设置了options.max_background_compactions > 1，则可以并行执行compaction。但是在同一个sorted run内部不能并行执行compaction。

### 分区compaction（Subcompaction）

Universal Compaction也支持Subcompaction，如果compaction的输出不在L0这一层，将会对合并前的文件进行分区，并采用多个线程来并发执行compaction。并发线程数目由options.max_subcompaction选项控制。

### 合并选项设置

有一个CompactionOptionsUniversal对象专门用于配置Universal Compaction，关于它的定义请参考rocksdb/universal_compaction.h。可以通过Options::compaction_options_universal来设置它。下面列举出CompactionOptionsUniversal中的选项：

- CompactionOptionsUniversal::size_ratio - Percentage flexibility while comparing file size. If the candidate file(s) size is 1% smaller than the next file's size, then include next file into this candidate set. Default: 1

- CompactionOptionsUniversal::min_merge_width - The minimum number of files in a single compaction run. Default: 2

- CompactionOptionsUniversal::max_merge_width - The maximum number of files in a single compaction run. Default: UINT_MAX

- CompactionOptionsUniversal::max_size_amplification_percent - The size amplification is defined as the amount (in percentage) of additional storage needed to store a single byte of data in the database. For example, a size amplification of 2% means that a database that contains 100 bytes of user-data may occupy upto 102 bytes of physical storage. By this definition, a fully compacted database has a size amplification of 0%. Rocksdb uses the following heuristic to calculate size amplification: it assumes that all files excluding the earliest file contribute to the size amplification. Default: 200, which means that a 100 byte database could require upto 300 bytes of storage.

- CompactionOptionsUniversal::compression_size_percent - If this option is set to be -1 (the default value), all the output files will follow compression type specified. If this option is not negative, we will try to make sure compressed size is just above this value. In normal cases, at least this percentage of data will be compressed. When we are compacting to a new file, here is the criteria whether it needs to be compressed: assuming here are the list of files sorted by generation time: [ A1...An B1...Bm C1...Ct ], where A1 is the newest and Ct is the oldest, and we are going to compact B1...Bm, we calculate the total size of all the files as total_size, as well as the total size of C1...Ct as total_C, the compaction output file will be compressed iff total_C / total_size < this percentage

- CompactionOptionsUniversal::stop_style - The algorithm used to stop picking files into a single compaction run. Can be kCompactionStopStyleSimilarSize (pick files of similar size) or kCompactionStopStyleTotalSize (total size of picked files > next file). Default: kCompactionStopStyleTotalSize

### 合并调优

下面列举出所有可能影响Universal Compaction的选项：

- options.compaction_options_universal: various options mentioned above

- options.level0_file_num_compaction_trigger the triggering condition of any compaction. It also means after all compactions' finishing, number of sorted runs will be under options.level0_file_num_compaction_trigger+1

- options.level0_slowdown_writes_trigger: if number of sorted runs exceed this value, writes will be artificially slowed down.

- options.level0_stop_writes_trigger: if number of sorted runs exceed this value, writes will stop until compaction finishes and number of sorted runs turn under this threshold.

- options.num_levels: if this value is 1, all sorted runs will be stored as level 0 files. Otherwise, we will try to fill non-zero levels as much as possible. The larger num_levels is, the less likely we will have large files on level 0.

- options.target_file_size_base: effective if options.num_levels > 1. Files of levels other than level 0 will be cut to file size not larger than this threshold.

- options.target_file_size_multiplier: it is effective, but we don't know a way to use this option in universal compaction that makes sense. So we don't recommend you to tune it.

## FIFO Compaction Style

参考[FIFO compaction style](https://github.com/facebook/rocksdb/wiki/FIFO-compaction-style)。

## 多线程compaction

如果配置正确，Compactions可以在多线程中运行。测试发现，如果在SSD上运行数据库，相较于单线程compaction，多线程compaction的性能可能提升多达10倍。

## Compaction过滤器（Compaction Filter）

Compaction Filter是Rocksdb为用户提供的一种在后台根据用户自定义逻辑删除或者修改键值对的方法。这对于实现用于自定义的垃圾回收（比如根据TTL移除过期的keys，或者丢弃某个范围内的keys等）非常有用。

为了使用compaction fileter，应用需要实现rocksdb/compaction_filter.h中的CompactionFilter接口，并且将它设置给ColumnFamilyOptions。或者，应用也可以实现CompactionFilterFactory接口，用于每个compaction实例都是用自己的compaction filter。CompactionFilterFactory可以根据compaction的上下文选择不同的compaction filter。

```
    options.compaction_filter = new CustomCompactionFilter();
    // or
    options.compaction_filter_factory.reset(new CustomCompactionFilterFactory());
```
    
两种不同的提供compaction filter的方式带来不同的线程安全要求。如果提供一个单一的compaction filter给Rocksdb，那么这个compaction filter应当是线程安全的，因为可能有多个compaction实例在并发的运行，这些compaction实例共享该compaction filter实例。如果提供一个compaction filter factory给Rocksdb，那么每一个compaction实例将创建自己的compaction filter实例，因此对于compaction filter的线程安全性不作要求。

尽管memtable flush也是一种特殊类型的compaction，在memtable flush过程中，compaction filter将不会起作用。

有两种实现compaction filter的API。一种是Filter/FilterMergeOperand API，提供了回调供compaction来判断是否过滤掉某些键值对。另一种是FilterV2 API，扩展了前一种API，允许改变某个key的value，或者丢弃从某个key开始的一段范围的key。

运用compaction filter之后可能产生如下结果：

- 保留某个key，什么也不做；

- 过滤掉某个key，value部分被一个删除标记（deletion marker）替代。如果合并的output level是最大level，则无需用删除标记替代。[注解：除最大level以外的level中用删除标记替代，是为了在后续较大level中删除具有相同key的记录，而最大level将不再参与合并，删除标记不再起到通知较大level删除与该删除标记对应的记录具有相同key的记录，在最大level直接过滤掉该key，无需写入删除标记，也无需写入该key相关的记录]；

- 改变某个key的value；

- 在返回KRemoveAndSkipUntil的情况下，跳过一段keys范围，这里不会插入删除标记，在较大level中依然可能出现这些keys，如果应用确信在较大level中没有关于这段keys的较老版本的数据，或者有关于这段keys的较老版本的数据但是出现这些较老版本的数据没关系，直接跳过一段keys范围是非常高效的；

- 如果在compaction的输入文件中有关于同一个key的多个版本，compaction filter只会运用在在最新的版本上，如果该最新版本是一个删除标记，将不会运用该compaction filter（但是如果关于某个key已经插入了删除标记，但是该删除标记不在compaction的输入中，则依然会在该key上运行compaction filter，虽然该key已经被标记为删除）；

如果开启了merge操作符（merge operator），则会在merge操作数（merge operand）上分别运行compaction filter，然后再执行merge操作；

如果在某个键值对上做了快照，则不能在该键值对上运行compaction filter。

# Compaction实现细节
## 待续......
## 待续......
## 待续......

# 参考
[RocksDB Level Compact 分析](https://developer.aliyun.com/article/655902)