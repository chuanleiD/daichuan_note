[TOC]

## 一、实验简述

#### 1.实验目的

​		本文以**OpenEuler-1.0-LTS**为研究对象，研究`/kernel-openEuler-1.0-LTS/fs/ext2`文件夹下Ext2第二代扩展文件系统，主要研究**索引结构的文件读取方式**、**写入数据时创建新块的逻辑**以及其他拓展内容。

#### 2.报告编写逻辑

​		本篇报告，以`//kernel-openEuler-1.0-LTS/fs/ext2/ext2.h`为主要获取数据结构定义，以及一些常量定义的文件，以`//kernel-openEuler-1.0-LTS/fs/ext2/inode.c`为核心代码文件进行研究。

​		在报告中，首先会展现并解释函数执行相关的**数据结构**、**常量定义**。接着，会对索引结构的**文件读取**、**创建新块**的逻辑过程中分别进行介绍：`ext2_get_block()、ext2_get_blocks()`中的调用方式，重要函数的执行逻辑



## 二、数据结构/定义

#### 1.Ext2有关定义

`EXT2_NDIR_BLOCKS`值为12，定义了直接块的数目。

`EXT2_IND_BLOCK`、`EXT2_DIND_BLOCK`、`EXT2_TIND_BLOCK`分别为一级、二级、三级间接块在`i_block[]`里面的位置。

`EXT2_N_BLOCKS`值为15，`i_block[]`存储15个地址（直接加间接）

```c
//kernel-openEuler-1.0-LTS/fs/ext2/ext2.h
#define	EXT2_NDIR_BLOCKS		12
#define	EXT2_IND_BLOCK			EXT2_NDIR_BLOCKS
#define	EXT2_DIND_BLOCK			(EXT2_IND_BLOCK + 1)
#define	EXT2_TIND_BLOCK			(EXT2_DIND_BLOCK + 1)
#define	EXT2_N_BLOCKS			(EXT2_TIND_BLOCK + 1)
```

根据`inode->i_sb`的值，获得数据块的大小，并且计算一个数据块可以存储多少个地址。

```c
//kernel-openEuler-1.0-LTS/fs/ext2/ext2.h
#define EXT2_BLOCK_SIZE(s)		((s)->s_blocksize)
#define	EXT2_ADDR_PER_BLOCK(s)		(EXT2_BLOCK_SIZE(s) / sizeof (__u32))
#define EXT2_BLOCK_SIZE_BITS(s)   ((s)->s_blocksize_bits)
#define	EXT2_ADDR_PER_BLOCK_BITS(s)	(EXT2_SB(s)->s_addr_per_block_bits)
```

其他变量定义

```c
typedef unsigned long ext2_fsblk_t;
```





#### 2.Ext2索引节点

Ext2 的索引节点的数据结构为：ext2_inode。其中`__le32	i_block[EXT2_N_BLOCKS];`表示地址索引块中的地址信息为**4Byte**，Ext2的地址索引表中保存了**15**个地址。`__le32	i_size;`表示采用32bit数据来保存一个文件的大小信息，也就是说单个文件的大小小于 $2^{32}$ Byte即4GB。

```c
//kernel-openEuler-1.0-LTS/fs/ext2/ext2.h
struct ext2_inode {
	__le16	i_mode;		/* 文件类型 */
	__le16	i_uid;		/* 拥有者标识号 */
	__le32	i_size;		/* 以bytes计，文件大小 */
	......
	__le32	i_blocks;	/* 文件所占块数 */
	__le32	i_flags;	/* 文件打开方式 */
    __le32	i_block[EXT2_N_BLOCKS];/* 指向数据块的指针数组 */
	......
};
```



#### 3.Ext2超级块

Ext2 超级块采用了`ext2_super_block`数据结构，用来描述 Ext2 文件系统整体信息。

其中，`s_log_block_size`是一个32bit的整数，表示了一个磁盘块的大小。具体计算逻辑如下：
$$
BlockSize = 1 <  < \left( {s\_log\_block\_size + 10} \right)Byte
$$
实际代码运行过程中，通过`int blocksize = inode->i_sb->s_blocksize;`获得块大小。

```c
//kernel-openEuler-1.0-LTS/fs/ext2/ext2.h
struct ext2_super_block {
	__le32	s_inodes_count;		/* 索引总数 */
	__le32	s_blocks_count;		/* 总块数 */
	__le32	s_r_blocks_count;	/* 超级用户保留块数 */
	__le32	s_free_blocks_count;	/* 空闲块数 */
	__le32	s_free_inodes_count;	/* 空闲索引节点数 */
	__le32	s_first_data_block;	/* 数个数据块 */
	__le32	s_log_block_size;	/* 数据块大小 */
	......
};
```



#### 4.间接指向的结构体

该数据结构记录了映射链中**每一级的索引信息**，通过多个Indirect结构体，可以链接起一串索引信息。

`*p`为指针，指向下一级索引/数据块的物理地址。

`key`记录了*p指针指向的内容，如果为0表示链的断裂。

<img src="https://i0.hdslb.com/bfs/album/d4f244b9222259ea5a17c091520922e4f87d1c09.png" alt="image-20220524204702727" style="zoom:70%;" /> 

`*bh`指向该当前间接块被从磁盘读出保存在内存中的数据结构（该条信息为阅读文献后得知的，因为内存操作有关，所以暂不研究）

```c
//kernel-openEuler-1.0-LTS/fs/ext2/ext2.h
typedef struct {
  __le32  *p;
  __le32  key;
  struct buffer_head *bh;
} Indirect;
```



#### 5.封装函数static int  ext2_get_block()

由于是封装函数，所以摆在前面先讲。

通过文件索引结构，找寻要找到文件数据所在的数据块主要实现函数。该函数是对`static int  ext2_get_blocks()`函数的封装

##### （1）代码执行逻辑

本函数作为`ext2_get_blocks()`函数的==封装函数==，隐藏了`ext2_get_blocks()`中的主要执行逻辑以及边界等情况的处理。

##### （2）输入介绍

`Input struct inode *inode`：目标文件的索引数据结构

`Input sector_t iblock`：块在文件中的逻辑号

`Input struct buffer_head *bh_result`：内存中的位置

`Input int create`：表示是否需要创建

```c
int ext2_get_block(struct inode *inode, sector_t iblock,
		struct buffer_head *bh_result, int create)
```

##### （3）输出介绍

`int ret`：返回int值代表函数的执行情况。

##### （4）代码介绍

```c
//kernel-openEuler-1.0-LTS/fs/ext2/inode.c
int ext2_get_block(struct inode *inode, sector_t iblock,
		struct buffer_head *bh_result, int create)
{
	//获取块数目的最大值
    unsigned max_blocks = bh_result->b_size >> inode->i_blkbits;
    //变量初始化
	bool new = false, boundary = false;
	u32 bno;
	int ret;
	//重点：执行ext2_get_blocks()函数
	ret = ext2_get_blocks(inode, iblock, max_blocks, &bno, &new, &boundary,
			create);
	if (ret <= 0)
		return ret;
    //达到边界、需要新创建块情况处理。
	map_bh(bh_result, inode->i_sb, bno);
	bh_result->b_size = (ret << inode->i_blkbits);
	if (new)
		set_buffer_new(bh_result);
	if (boundary)
		set_buffer_boundary(bh_result);
	return 0;
}
```



## 三、索引结构的文件数据块找寻

### 相关知识准备

​		首先，参考老师课堂所讲内容以及《深入分析Linux内核源代码》，我对于Ext2文件索引读取方式的执行逻辑有了初步的了解。以下是索引结构的介绍，核心执行逻辑请见代码介绍：static int  ext2_get_blocks()。

​		存储在系统中的文件使用一个数据结构维护其文件的索引信息。主要有文件控制模式、用户标识、地址索引表等等。其中，地址索引表是一个数组，存储了12个地址，其中前十个直接指向磁盘块，后面三个依次为一级间接块、二级间接块、三级间接块。以一级间接块为例，地址指向了一个存储地址的直接寻址块，读到一级间接块时，依次读取其指向的直接寻址块所指向的每个磁盘块。以此类推，采用一个小的地址索引表、采用索引结构即可找到该文件所有的地址块。

<img src="https://i0.hdslb.com/bfs/album/5106bffa84e9cacd0ab6a6771398ad2029a4e5c5.png" alt="image-20220524102418954" style="zoom:50%;" /> 



### 相关函数实现

#### 1.核心函数static int  ext2_get_blocks()

​		通过文件索引结构，找寻要找到文件数据所在的数据块主要实现函数。如果成功找到，则参与读取文件的文件物理地址找寻功能成功。如果no_block，则调用函数为文件数据分配一个新块。这里主要介绍其第一个功能。

##### （1）代码执行逻辑

​		ext2_get_blocks()函数实际执行过程中，主要调用ext2_block_to_path()、ext2_get_branch()。通过ext2_block_to_path()先获得逻辑块号对应索引结构的各级偏移地址、再调用ext2_get_branch()根据各级相对偏移地址获得真实各级物理地址链。再在主函数的if(!partial)的判断语句中，根据物理地址链读取数据块。对于获得的直接寻址块的地址，可以直接读取该地址到边界的所有地址对应的数据块以节省操作，这也就是boundary参数的意义。

<img src="https://i0.hdslb.com/bfs/album/a7d8ce6cd07c8641ff76cd4820e1c12b4e359b67.png" alt="image-20220526104209406" style="zoom: 67%;" />    

##### （2）输入介绍

最有价值的参数

`Input struct inode *inode`：目标文件的索引数据结构。

`Input sector_t iblock`：块在文件中的逻辑号。

判断条件

`Input unsigned long maxblocks`：读取块数目的最大限制，作为判断条件之一。

`Output u32 *bno`：参与错误处理，用于ext2_get_branch的参数。

`Output bool *new`：与创建新块有关，暂不考虑。

`Output bool *boundary`：间接块边界判断。

`Input int create`：告知程序是否需要创建新块，与创建新块有关。

##### （3）输出介绍

 return > 0，映射或分配的块数。

 return = 0，不同查找失败返回0。

 return < 0，返回各种错误的编号。

##### （4）代码介绍

```c
//kernel-openEuler-1.0-LTS/fs/ext2/inode.c
//主要介绍找寻数据块、文件读取相关功能，新块创建功能在之后会解释
static int ext2_get_blocks(struct inode *inode,
			   sector_t iblock, unsigned long maxblocks,
			   u32 *bno, bool *new, bool *boundary,
			   int create)
{
	//初始化变量：offset[4]、chain[4]、*partial、blocks_to_boundary、depth等
    int err;
    int offsets[4];
	Indirect chain[4];
    Indirect *partial;
	ext2_fsblk_t goal;
	int indirect_blks;
    int blocks_to_boundary = 0;
    int depth;
	struct ext2_inode_info *ei = EXT2_I(inode);
	int count = 0;
	ext2_fsblk_t first_block = 0;

	BUG_ON(maxblocks == 0);//若maxblocks==0，debug是否开启

    //执行ext2_block_to_path()函数，获得索引结构下的偏移数组，
    //blocks_to_boundary：到边界的距离
    //返回depth，偏移数组的深度
	depth = ext2_block_to_path(inode,iblock,offsets,&blocks_to_boundary);

	if (depth == 0)//depth为0，说明错误（正确取值：1、2、3、4）
		return -EIO;

    //执行ext2_get_branch()，根据得到的offset[4]数组，获得物理地址链chain[4]
    //返回值*partial，如果正常查找返回NULL，否则返回key=0的空白块对应的指针
	partial = ext2_get_branch(inode, depth, offsets, chain, &err);
    //当前功能正常会进入该判断语句。
	if (!partial) {
        //读取数据块物理地址并转换地址形式（大小端转换）
		first_block = le32_to_cpu(chain[depth - 1].key);
		count++;
        //如果读取数目不到输入的最大限制、并且没到边界，则可以继续
		while (count < maxblocks && count <= blocks_to_boundary) {
			ext2_fsblk_t blk;
			//检查chain链信息的准确性
			if (!verify_chain(chain, chain + depth - 1)) {
				err = -EAGAIN;
				count = 0;
				partial = chain + depth - 1;
				break;
			}
            //读取直接块物理地址的count偏移指向的数据块
			blk = le32_to_cpu(*(chain[depth-1].p + count));
			if (blk == first_block + count)
				count++;
			else
				break;
		}
		if (err != -EAGAIN)//链中数据没有问题
			goto got_it;
	}
	//如果不是创建新块、也没发生错误，可以进入cleanup清缓存结束了
	if (!create || err == -EIO)
		goto cleanup;    
//文件读取功能用不到的代码省略
......
//文件读取功能用不到的代码省略
got_it: ...err = count;...//进入got_it生成err返回值，判断是否到边界，之后“Clean up and exit”
cleanup: ...return err;...//收尾工作，清缓存，返回错误等  
}		
```



#### 2.static int ext2_block_to_path()

##### （1）代码执行逻辑

根据输入的块号`i_block`，在输入的文件索引中进行索引结构的查找，返回该块在存储中对应的各级偏移位置。`ext2_block_to_path()`通过多级if/else嵌套，实现了将文件索引中的逻辑地址转换为offset[4]数组中存储的，各级间接块存储的相对偏移地址。具体的转换逻辑请看代码详情里的详细注释。

<img src="https://i0.hdslb.com/bfs/album/681861c25c19f9bae7e851cade78241a5e5932c6.png" alt="image-20220526104309764" style="zoom:67%;" /> 

##### （2）输入介绍

`Input struct inode *inode`：inode型指针，指向想要查找bolck的那个文件的索引结构。

`Input long i_block`：文件内的逻辑块号，作为输入。

`Output int offsets[4]`：32bit的长度为4的数组，因为Ext2最多有三级简介块，因此采用长度为4的数组保存计算得到的各个组内的偏移量。

`Output int *boundary`：ext2_get_block()函数中定义`int blocks_to_boundary = 0;`，之后输入，用于边界判定。

##### （3）输出介绍

`int n`：输出值，表示逻辑块号所对应的深度

值为0:错误、值为1:位于直接块内、值为2:位于第一间接块内、值为3:位于第二间借块内、值为4:位于第三间接块内。

##### （4）代码详情

```c
//kernel-openEuler-1.0-LTS/fs/ext2/inode.c
static int ext2_block_to_path(struct inode *inode,
			long i_block, int offsets[4], int *boundary)
{
    //有“Ext2有关定义”可知，ptrs为一个数据块可以存放的32bit指针数目
	int ptrs = EXT2_ADDR_PER_BLOCK(inode->i_sb);
    //ptrs所对应的值的二进制表示的位数
	int ptrs_bits = EXT2_ADDR_PER_BLOCK_BITS(inode->i_sb);
    
    //direct_blocks为直接块的数量：12
    //indirect_blocks为第一间接块所映射的数据块的数量,=ptrs
    //double_blocks为第二间接块所映射的数据块的数量,=ptrs^2（等价于）
	const long direct_blocks = EXT2_NDIR_BLOCKS,//EXT2_NDIR_BLOCKS=12
		indirect_blocks = ptrs,
		double_blocks = (1 << (ptrs_bits * 2));
    
	int n = 0;//n为返回值，表示深度
	int final = 0;//用于boundary的边界判断
	
    //如果i_block逻辑块号小于0，那么调用ext2_msg()报错
	if (i_block < 0) {
		ext2_msg(inode->i_sb, KERN_WARNING,
			"warning: %s: block < 0", __func__);
        
    //如果0<=i_block逻辑块号<12，位于直接块内
	} else if (i_block < direct_blocks) {
		offsets[n++] = i_block;//offsets[0]赋值为直接块内的偏移
		final = direct_blocks;//n=1,final=12
        
    //首先，i_block-=12
    //此时，如果i_block<indirect_blocks第一间接块大小，说明对应与第一间接块上的偏移
	} else if ( (i_block -= direct_blocks) < indirect_blocks) {
		//offsets[0]=12，对应i_block上的偏移
        offsets[n++] = EXT2_IND_BLOCK;
        //offsets[1]=i_block，对应其在直接寻址块上的偏移
		offsets[n++] = i_block;
		final = ptrs;//n=2,final=ptrs
        
    //首先，i_block-=indirect_blocks
    //此时，如果i_block<double_blocks第二间接块大小，说明对应与第二间接块上的偏移    
	} else if ((i_block -= indirect_blocks) < double_blocks) {
		//offsets[0]=13，对应i_block上的偏移
        offsets[n++] = EXT2_DIND_BLOCK;
        //offsets[1]=i_block//ptrs，对应第一间接块上的偏移
		offsets[n++] = i_block >> ptrs_bits;
        //offsets[2]=i_block%ptrs，对应第一间接块->直接寻址块上的偏移
		offsets[n++] = i_block & (ptrs - 1);
		final = ptrs;//n=3,final=ptrs
        
    //首先，i_block-=double_blocks
    //此时，如果i_block<第三间接块大小ptrs^3，说明对应与第二间接块上的偏移     
	} else if (((i_block -= double_blocks) >> (ptrs_bits * 2)) < ptrs) {
        //offsets[0]=14，对应i_block上的偏移
		offsets[n++] = EXT2_TIND_BLOCK;
        //offsets[1]=i_block//(ptrs^2)，对应第二间接块上的偏移
		offsets[n++] = i_block >> (ptrs_bits * 2);
        //offsets[2]=(i_block//ptrs)%ptrs，对应第二间接块->第一间接块上的偏移
		offsets[n++] = (i_block >> ptrs_bits) & (ptrs - 1);
        //offsets[3]=i_block%ptrs，对应第二间接块->第一间接块->直接寻址快上的偏移
		offsets[n++] = i_block & (ptrs - 1);
		final = ptrs;//n=4,final=ptrs
        
    //如果运行到这里，说明文件过大，这时应当调用ext2_msg()报错    
	} else {
		ext2_msg(inode->i_sb, KERN_WARNING,
			"warning: %s: block is too big", __func__);
	}
    
    //通过以上获得的final值，以及i_block、ptrs来获得boundary
    //如果要寻找的块是在块内的最后一个指针指向的，*boundary的值就是0,否则返回与边界的距离??
	if (boundary)//为什么不是*boundary？？
		*boundary = final - 1 - (i_block & (ptrs - 1));

	return n;//返回该逻辑块的深度
}
```



### 3.static Indirect *ext2_get_branch()

##### （1）代码执行逻辑

​		根据调用`ext2_block_to_path()`获得的`offsets[4]`数组，我们知道了找到文件的逻辑位置对应的物理地址所需要知道的各级间接块内的相对位置偏置，也知道了该逻辑地址对应于哪一级间接块，即深度`depth`。通过调用`*ext2_get_branch()`函数，采用`while(--depth){}`循环的方式，找出`offset[0]`、`offset[1]`、`offset[2]`、`offset[3]`各级对应的真实物理地址，采用`Indirect chain[4]`的结构存储。chain[i]中的p指针指向真实的该偏移的物理地址，key值对应该物理地址所指向的内容的值。通过4个chain[i]，即可对于offset[4]各级相对偏移找到其真实的物理地址，通过真实物理地址的各级跳转，即可找到逻辑位置对应的真实的数据块的物理地址。

<img src="https://i0.hdslb.com/bfs/album/f4f9a4baaebf3d264042ffa638ddd5b24e085680.png" alt="image-20220526104327860" style="zoom:67%;" /> 

##### （2）输入介绍

`Input: struct inode *inode`：需要读的文件的索引节点。

`Input: int depth`：由`ext2_block_to_path`函数得到的深度，代表由几层间接关系。

`Input: int *offsets`：由`ext2_block_to_path`函数得到，各级块内的相对偏移值。

`Output: Indirect chain[4]`：采用4个`Indirect`结构体链接起实际节点。

`Output: int *err`：记录错误情况并返回。

##### （3）输出介绍

返回一个`Indirect`结构体的指针，即`chain`的首地址`*p`。

##### （4）代码详情

```c
//kernel-openEuler-1.0-LTS/fs/ext2/inode.c
static Indirect *ext2_get_branch(struct inode *inode,
				 int depth,
				 int *offsets,
				 Indirect chain[4],
				 int *err)
{
	struct super_block *sb = inode->i_sb;//该文件对于的文件系统超级块
	Indirect *p = chain;//chain数组首地址
    
	struct buffer_head *bh;
	*err = 0;
    
	//通过*offsets将offsets[0]的物理地址以及指向的位置，索引信息添加到chain[0]中。
	add_chain (chain, NULL, EXT2_I(inode)->i_data + *offsets);
    //如果p->key为0，说明没有指向一个块，即逻辑块的指针还没有和数据块联系起来，跳转到no_block
	if (!p->key)
		goto no_block;
    
    //由depth的值进行逐层分解，depth先自减后判定
    //这里直接块的话，好像进不去循环，
	while (--depth) {
        //驱动层的读写函数读取上一层找到的间接块的数据
		bh = sb_bread(sb, le32_to_cpu(p->key));
		if (!bh)//如果读不到，失败转到failure
			goto failure;
		read_lock(&EXT2_I(inode)->i_meta_lock);
        
        //检验建立的这个间接结构体有没有问题
		if (!verify_chain(chain, p))
			goto changed;//如果有问题，转到changed
        
        //检验没有问题之后，采用*++offsets，为++p指向的内容赋值。
        //即处理offsets[1]中的相对偏移，获得物理地址，指向的物理地址
        //(__le32*)bh->b_data + *++offsets的意思是
        //上边得到bh缓冲区是间接块的缓冲区，加上偏移就是下一级的入口
		add_chain(++p, bh, (__le32*)bh->b_data + *++offsets);
        
		read_unlock(&EXT2_I(inode)->i_meta_lock);
		if (!p->key)//如果p->key为0，即下一级没有块了，则转到no_block
			goto no_block;
	}
	return NULL;

changed:
	read_unlock(&EXT2_I(inode)->i_meta_lock);
	brelse(bh);
	*err = -EAGAIN;
	goto no_block;
failure:
	*err = -EIO;
no_block:
	return p;//正常的返回
}
```



#### 4.add_chain()与verify_chain()

作为`ext2_get_branch()`中的两个较为重要的函数，这里做稍微详细的介绍。

<img src="https://i0.hdslb.com/bfs/album/d4f244b9222259ea5a17c091520922e4f87d1c09.png" alt="image-20220524204702727" style="zoom:70%;" /> 

`add_chain()`增加链接，把间接指针指向这个内存缓冲区，v就是offset[0]地址的指针的值，是文件索引地址数组的15个值之一。通过该函数，建立一层一层的chain结构体。

```c
static inline void add_chain(Indirect *p, struct buffer_head *bh, __le32 *v)
{
	p->key = *(p->p = v);
	p->bh = bh;//初始的时候，bh=NULL，因为此时不是通过间接块来存储索引，而是i_data[];
}
```

`verify_chain()`检验建立的这个间接结构体有没有问题。检测chain、p：

从from开始，只要from->key == *from->p，及上图中的关系对于每个from都正确，就是对的，正确则from++检查下一个直至检查结束，则<=to指针的都检查成功，返回1，否则返回0。

```c
static inline int verify_chain(Indirect *from, Indirect *to)
{
	while (from <= to && from->key == *from->p)
		from++;
	return (from > to);
}
```



## 四、新块创建流程

### 相关知识准备

​		新块创建时，首先同样进行索引结构的文件数据块找寻，通过执行`ext2_block_to_path()`找到文件需要创建存储数据块在索引中的各级相对偏移、然后执行`*ext2_get_branch()`，通过返回的`partial`指针得知缺少数据块的级数、物理地址。接着执行`ext2_find_goal()`、`ext2_blks_to_allocate()`、`ext2_alloc_branch()`进行具体的创建。

### 相关函数实现

#### 1.核心函数static int  ext2_get_blocks()

​		通过文件索引结构，找寻要找到文件数据所在的数据块主要实现函数。如果成功找到，则参与读取文件的文件物理地址找寻功能成功。如果no_block，则调用函数为文件数据分配一个新块。这里主要介绍其第二个功能。

##### （1）代码执行逻辑

​		`*ext2_get_branch()`前的操作与上面描述的相同，在执行完该函数后，

<img src="https://i0.hdslb.com/bfs/album/cab6da9d6f2cb333d8aa9af75c2ff204eb8892be.png" alt="image-20220526110624889" style="zoom:67%;" />     

##### （2）输入介绍

最有价值的参数

`Input struct inode *inode`：目标文件的索引数据结构。

`Input sector_t iblock`：块在文件中的逻辑号。

判断条件

`Input unsigned long maxblocks`：读取块数目的最大限制，作为判断条件之一。

`Output u32 *bno`：参与错误处理，用于ext2_get_branch的参数。

`Output bool *new`：与创建新块有关。

`Output bool *boundary`：间接块边界判断。

`Input int create`：告知程序是否需要创建新块，与创建新块有关。

##### （3）输出介绍

 return > 0，映射或分配的块数。

 return = 0，不同查找失败返回0。

 return < 0，返回各种错误的编号。

##### （4）代码介绍

```c
//kernel-openEuler-1.0-LTS/fs/ext2/inode.c
//主要介绍找寻数据块、文件读取相关功能，新块创建功能在之后会解释
static int ext2_get_blocks(struct inode *inode,
			   sector_t iblock, unsigned long maxblocks,
			   u32 *bno, bool *new, bool *boundary,
			   int create)
{
	//初始化变量定义
    int err;
    int offsets[4];
	Indirect chain[4];
    Indirect *partial;
	ext2_fsblk_t goal;
	int indirect_blks;
    int blocks_to_boundary = 0;
    int depth;
	struct ext2_inode_info *ei = EXT2_I(inode);
	int count = 0;
	ext2_fsblk_t first_block = 0;

	BUG_ON(maxblocks == 0);//若maxblocks==0，debug是否开启
    
	//----------------------------------------------------------------------------
    //根据文件号，最终获取物理地址链的过程，前方有细讲因此省略
	depth = ext2_block_to_path(inode,iblock,offsets,&blocks_to_boundary);
	if (depth == 0)//depth为0，说明错误（正确取值：1、2、3、4）
		return -EIO;
	partial = ext2_get_branch(inode, depth, offsets, chain, &err);
    //----------------------------------------------------------------------------
    
    //----------------------------------------------------------------------------
	if (!partial) {
        //文件读取功能，相关代码已在前面展示，故此省略
        ......
	}
	//如果不是创建新块、也没发生错误，可以进入cleanup清缓存结束了
	if (!create || err == -EIO)
		goto cleanup;    
    //----------------------------------------------------------------------------
    
    mutex_lock(&ei->truncate_mutex);//忙等同步机制
    
	//----------------------------------------------------------------------------
	//错误排查段
    //如果chain已有块的链接有错误，partial位置错误。partial指向chain[i]最小的key=0的位置
    if (err == -EAGAIN || !verify_chain(chain, partial)) {
		while (partial > chain) {
			brelse(partial->bh);//partial位置错误：回退并释放缓存
			partial--;
		}
        //重新获得partial
		partial = ext2_get_branch(inode, depth, offsets, chain, &err);
		//重新检查后发现不缺少块，与文件读取时候的逻辑类似
        if (!partial) {//重新执行ext2_get_branch发现不缺数据块
			count++;
			mutex_unlock(&ei->truncate_mutex);
			goto got_it;
		}

		if (err) {
			mutex_unlock(&ei->truncate_mutex);
			goto cleanup;
		}
	}
    //----------------------------------------------------------------------------

	if (S_ISREG(inode->i_mode) && (!ei->i_block_alloc_info))
		ext2_init_block_alloc_info(inode);
	//找到一个优先分配的位置，寻找一个分配的块号，goal是返回的目标块号。成功返回0
	goal = ext2_find_goal(inode, iblock, partial);
	//indirect_blks：计算得到需要分配的数据块数，
	indirect_blks = (chain + depth) - partial - 1;
	//如果goal在inode的预留窗口中，并且预留窗口空间足够大，则在预留窗口中分配。
	//如果goal在inode的预留窗口中，但是预留窗口空间不足，
    //则通过try_to_extend_reservation（）扩展预留窗口。
	count = ext2_blks_to_allocate(partial, indirect_blks,
					maxblocks, blocks_to_boundary);
	//调用ext2_alloc_branch()->ext2_alloc_blocks()->ext2_new_blocks()
    //进行*partial之后节点物理地址、间接直接块的物理地址分配
	err = ext2_alloc_branch(inode, indirect_blks, &count, goal,
				offsets + (partial - chain), partial);
	//错误排查
	if (err) {
		mutex_unlock(&ei->truncate_mutex);
		goto cleanup;
	}

	if (IS_DAX(inode)) {
		......
	}
	*new = true;
	//间接块与下一级数据块间，只有首地址建立了链接，下面对之后的存储建立起链接关系。
	ext2_splice_branch(inode, iblock, partial, indirect_blks, count);
	mutex_unlock(&ei->truncate_mutex);
    
got_it: ...err = count;...//进入got_it生成err返回值，判断是否到边界，之后“Clean up and exit”
cleanup: ...return err;...//收尾工作，清缓存，返回错误等 
}		
```



### 2.static inline ext2_fsblk_t ext2_find_goal()

##### （1）代码执行逻辑

​		用来找一个block数据块作为新块分配时最适合的目标goal,，在这里并不考察得到的最合适目标goal是否已经被占用，只要能找到这个最理想的目标即可，目标goal是否能够被分配，是否已经被占用将在之后由函数`ext2_alloc_branch`进行处理处理。

<img src="https://i0.hdslb.com/bfs/album/a5ee1bcd93aaceb7ce3d41258ed78e2646e0987c.png" alt="image-20220526104132476" style="zoom:67%;" />      

##### （2）输入介绍

`Input struct inode *inode`：目标文件的索引数据结构。

`Input long block`：目标文件的逻辑号iblock。

`Input Indirect *partial`：由函数`*ext2_get_branch()`得到的partial指针。

##### （3）输出介绍

ext2_fsblk_t 类型传回`goal`变量（ext2_fsblk_t 定义为：unsigned long）

##### （4）代码详情

```c
//kernel-openEuler-1.0-LTS/fs/ext2/inode.c
static inline ext2_fsblk_t ext2_find_goal(struct inode *inode, long block,
					  Indirect *partial)
{
	//块分配里常用ext2_inode_info结构体
    struct ext2_block_alloc_info *block_i;
	block_i = EXT2_I(inode)->i_block_alloc_info;
    //尝试启发式的顺序分配，如果做不到，至少尝试一下适当的本地化
	if (block_i && (block == block_i->last_alloc_logical_block + 1)
		&& (block_i->last_alloc_physical_block != 0)) {
		return block_i->last_alloc_physical_block + 1;
	}
	//找足够大的附近位置进行分配，具体分配规则见5
	return ext2_find_near(inode, partial);
}
```



### 3.static int ext2_blks_to_allocate()

##### （1）代码执行逻辑

查找块的映射关系，并计算需要为给定分配的直接块的数量。

<img src="https://i0.hdslb.com/bfs/album/7fbe66793fd75d4da191c547b1926102411b7bd6.png" alt="image-20220526102649237" style="zoom:67%;" /> 

##### （2）输入介绍

`Input Indirect * branch`：传入的指针partial。

`Input int k`：所需的间接块块数，计算方法见1。

`Input unsigned long blks`：数据块最大数目限制maxblocks。

`Input int blocks_to_boundary`：间接块相对于边界的距离blocks_to_boundary。

##### （3）输出介绍

`int` 输出得到creat变量，给出给定分配的直接块的数量。

##### （4）代码详情

```c
//kernel-openEuler-1.0-LTS/fs/ext2/inode.c
static int
ext2_blks_to_allocate(Indirect * branch, int k, unsigned long blks,
		int blocks_to_boundary)
{
	unsigned long count = 0;
	//如果不是直接块
	if (k > 0) {
        //取count += min{maxblocks,blocks_to_boundary+1}
		if (blks < blocks_to_boundary + 1)
			count += blks;//受maxblocks限制
		else
			count += blocks_to_boundary + 1;//受blocks_to_boundary限制
		return count;
	}
	//else:k==0
    //计算需要为给定分配的直接块的数量
    //首先与min{maxblocks,blocks_to_boundary+1}以及指向数据块一系列空间为0
	count++;
	while (count < blks && count <= blocks_to_boundary
		&& le32_to_cpu(*(branch[0].p + count)) == 0) {
		count++;
	}
	return count;
}
```



### 4.static int ext2_alloc_branch()

##### （1）代码执行逻辑

​		为数据块建立映射路径：调用函数`ext2_alloc_blocks`得到长度为4的new_blocks[4]，然后基于new_blocks[4]，采用for循环的方式，填补*partial指针指向的缺少块的信息。虽然new_blocks[4]会定向于一个实际物理块的首地址，但是，可以基于首地址对于一整个块物理地址进行分配，并分配指向的物理地址。

<img src="https://i0.hdslb.com/bfs/album/439c94d1fe4abfa82d204b62f1464d3432e99b6b.png" alt="image-20220526104100960" style="zoom:67%;" />  	

##### （2）输入介绍

`Input truct inode *inode`：目标文件的索引数据结构。

`Input int indirect_blks`：所需的间接块块数，计算方法见1。

`Input int *blks`：count的引用，需要分配的数据块数量。

`Input ext2_fsblk_t goal`：ext2_find_goal()得到是最适合目标。

`Input int *offsets`：offsets + (partial - chain)缺块位置的相对位置。

`Input Indirect *branch`：传入的指针partial。

##### （3）输出介绍

int型，ext2_get_blocks()中的`err`。

##### （4）代码详情

```c
//kernel-openEuler-1.0-LTS/fs/ext2/inode.c
static int ext2_alloc_branch(struct inode *inode,
			int indirect_blks, int *blks, ext2_fsblk_t goal,
			int *offsets, Indirect *branch)
{
	//变量定义
    int blocksize = inode->i_sb->s_blocksize;
	int i, n = 0;
	int err = 0;
	struct buffer_head *bh;
	int num;
	ext2_fsblk_t new_blocks[4];
	ext2_fsblk_t current_block;
	
    //分配间接块和数据块，在函数ext2_allocate_blocks()中完成
	num = ext2_alloc_blocks(inode, goal, indirect_blks,
				*blks, new_blocks, &err);
	//错误处理
    if (err)
		return err;

	//通过for循环的方式，建立映射关系，将映射路径上的映射关系进行填充
    //new_blocks[]数组中存储了ext2_allocate_blocks()分配的间接块和直接块的索引块号
    //new_blocks[0]中保存的就是第一个间接块的物理块号，只记录数据块的起始块号
    branch[0].key = cpu_to_le32(new_blocks[0]);
	for (n = 1; n <= indirect_blks;  n++) {
        //获取父块的缓冲区头，将其归零并将指针设置为新块，然后将父块发送到磁盘。
		bh = sb_getblk(inode->i_sb, new_blocks[n-1]);
        //错误检查
		if (unlikely(!bh)) {
			err = -ENOMEM;
			goto failed;
		}
		branch[n].bh = bh;
		lock_buffer(bh);
		memset(bh->b_data, 0, blocksize);
        //于每个间接块，从磁盘读出其内容，然后将在相应的偏移处记录下一个间接块的物理块号
		branch[n].p = (__le32 *) bh->b_data + offsets[n];//将指针指向间接块的相应偏移处
		branch[n].key = cpu_to_le32(new_blocks[n]);//设置key，即下一级映射的间接块物理块号
		*branch[n].p = branch[n].key;//将内容写入间接块的相应位置
        //如果到了最后一个间接块，那么需要在该块中记录数据块的块号，分配的数据块是连续的
		if ( n == indirect_blks) {
			current_block = new_blocks[n];
			for (i=1; i < num; i++)
				*(branch[n].p + i) = cpu_to_le32(++current_block);
		}
		set_buffer_uptodate(bh);
		unlock_buffer(bh);
		mark_buffer_dirty_inode(bh, inode);
		if (S_ISDIR(inode->i_mode) && IS_DIRSYNC(inode))
			sync_dirty_buffer(bh);
	}
	*blks = num;
	return err;

failed://执行失败时的错误处理
}
```



### 5.static int ext2_alloc_blocks()

##### （1）代码执行逻辑

​		为数据块建立映射路径：调用函数`ext2_alloc_blocks`得到长度为4的new_blocks[4]，然后基于new_blocks[4]，采用for循环的方式，填补*partial指针指向的缺少块的信息。虽然new_blocks[4]会定向于一个实际物理块的首地址，但是，可以基于首地址对于一整个块物理地址进行分配，并分配指向的物理地址。ext2_alloc_branch()保证分配的直接数据块在磁盘上连续，但有可能无法分配所需的全部数据块。

<img src="https://i0.hdslb.com/bfs/album/ef4fd640ed116c0a9e66875152f552b0f5cd2371.png" alt="image-20220526105024607" style="zoom:67%;" />  	

##### （2）输入介绍

`Input truct inode *inode`：目标文件的索引数据结构。

`Input ext2_fsblk_t goal`：ext2_find_goal()得到是最适合目标。

`Input int indirect_blks`：所需的间接块块数，计算方法见1。

`Input int *blks`：count的引用，需要分配的数据块数量。

`Output ext2_fsblk_t new_blocks[4]`：存储函数执行后分配的间接块和直接块的索引块号。

`Output int *err`：返回错误信息。

##### （3）输出介绍

int型，ext2_alloc_branch()中的`num`。

##### （4）代码详情

```c
//kernel-openEuler-1.0-LTS/fs/ext2/inode.c
static int ext2_alloc_blocks(struct inode *inode,
			ext2_fsblk_t goal, int indirect_blks, int blks,
			ext2_fsblk_t new_blocks[4], int *err)
{
	//变量初始化
    int target, i;
	unsigned long count = 0;
	int index = 0;
	ext2_fsblk_t current_block = 0;
	int ret = 0;
	target = blks + indirect_blks;

	while (1) {
        //循环调用ext2_new_blocks()，来分配想要数量的磁盘块。
        //函数返回的磁盘块数量不一定足够，但其返回的磁盘块一定连续。
		count = target;
		current_block = ext2_new_blocks(inode,goal,&count,err);
		if (*err)
			goto failed_out;

		target -= count;
        //分配的磁盘块记录在返回结果中，首先将间接块号记录在返回结果（new_blocks数组）的前几项。
		while (index < indirect_blks && count) {
			new_blocks[index++] = current_block++;
			count--;
		}
		//判断所需数量的间接块是否分配完成
        //完成结束循环，未完成继续循环。
		if (count > 0)
			break;
	}

	new_blocks[index] = current_block;

	ret = count;
	*err = 0;
	return ret;
    
failed_out: ...return ret;//执行失败处理。
}
```



### 6.ext2_find_near()与ext2_new_blocks()

##### （1）ext2_find_near()	

​		`ext2_find_near()`：找到一个足够大的位置进行分配，分配的规则是，首先寻找这个块所在的指针块，分配其左边附近的块，如果没找到，就去参数ind所在的间接块内找，如果还没有找到，就去inode所在的磁道块组内找一个。

```c
//kernel-openEuler-1.0-LTS/fs/ext2/balloc.c
static ext2_fsblk_t ext2_find_near(struct inode *inode, Indirect *ind)
{
	struct ext2_inode_info *ei = EXT2_I(inode);
	__le32 *start = ind->bh ? (__le32 *) ind->bh->b_data : ei->i_data;
	__le32 *p;
	ext2_fsblk_t bg_start;
	ext2_fsblk_t colour;

	for (p = ind->p - 1; p >= start; p--)
		if (*p)
			return le32_to_cpu(*p);

	if (ind->bh)
		return ind->bh->b_blocknr;

	bg_start = ext2_group_first_block_no(inode->i_sb, ei->i_block_group);
	colour = (current->pid % 16) *
			(EXT2_BLOCKS_PER_GROUP(inode->i_sb) / 16);
	return bg_start + colour;
}
```



##### （2）ext2_new_blocks()	

​		`ext2_new_blocks()`：这个函数非常复杂，并且好像和ext2预留窗口机制的红黑树结合，因此这里在阅读源码及相关解析之后，进行简单的描述。首先从goal所在的块组中分配磁盘块，计算该块组的空闲磁盘块数量，如果该文件的预分配窗口为空并且当前空闲磁盘块数量少于预留窗口的大小，那么我们对该文件在这个块组中分配就不使用预留窗口（my_rsv = NULL）接下来从该块组中去分配磁盘块。如果分配成功了直接返回即可。

`Input inode`：代表该需要分配磁盘块的文件。

`Input goal`：代表调用者建议最好从哪开始分配，以达到文件数据块的最佳连续性。

`Input count`：调用者传入的欲分配的磁盘块数量，同时这个值也作为返回值告知调用者时机分配的磁盘块数量。

`Output errorp`：该函数执行过程中可能产生的 错误。

函数的返回值为分配到的起始物理磁盘块号

参考：[ext2_new_blocks解析](https://www.xuebuyuan.com/1470688.html)

```c
//kernel-openEuler-1.0-LTS/fs/ext2/balloc.c
ext2_fsblk_t ext2_new_blocks(struct inode *inode, ext2_fsblk_t goal,
		    unsigned long *count, int *errp)
{
......
}
```



