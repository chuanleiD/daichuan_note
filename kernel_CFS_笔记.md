# CFS代码分析

姓名：代传磊      学号：519010910013      邮箱：daichuan@sjtu.edu.cn      代码：OpenEule kernel-4.19

[TOC]

## 概述

### CFS算法介绍

​		参见`kernel-4.19`中的CFS相关文档：**`Documentation\scheduler\sched-design-CFS.txt`** 可以了解到CFS算法的相关信息。CFS“完全公平调度程序”，在`Linux 2.6.23`之后被引入linux系统内核，作为 `SCHED_OTHER` 进程调度算法的替代算法。其80% 的设计可以用一句话概括：CFS 基本上是在真实硬件上建模了一个“理想的、精确的多任务 CPU”。

```
80% of CFS's design can be summed up in a single sentence: CFS basically models
an "ideal, precise multi-tasking CPU" on real hardware.
```

​		该算法的逻辑思想大致概括如下：假设当前系统中存在 $n$ 个权重相同的就绪程序，在一个调度周期的时间`r_running`内，“**理想的多任务CPU**”希望能够对于每一个就绪进程采用 $100/n$ 的CPU运行功率运行`r_running`时间。由于在**真实硬件**上，同一时间单个CPU只能运行同一任务，所以**CFS采用分时的思路**，每个进程运行 $1/n$ 个`r_running`时间来起到等效的效果。

```
"Ideal multi-tasking CPU" is a (non-existent  :-)) CPU that has 100% physical
power and which can run each task at precise equal speed, in parallel, each at
1/nr_running speed.
```

### CFS与课程中所学进程调度算法的对比

相比于课程中所学的几种进程调度算法，CFS采用了许多不同的设计思想，因此下面对于其中几个重点的不同作出介绍。

1.与时间片轮转法、SPF等采用**时间片**进行分时运行来实现抢占机制不同，CFS采用了根据当前就绪进程数生成的**调度周期**来进行时间控制，各个进程在一个调度周期内分配运行时间，在下一个调度周期开始时更新状态。

```c
static u64 __sched_period(unsigned long nr_running)
```

2.与**优先级调度**算法为进程设置动态优先级、静态优先级不同。CFS为每个进程赋予了一个取值在[-20,19]的 $nice$ 值**权重**，用于在调度周期之中分配进程的运行时间。

```c
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
```

3.与SPF、SRT、HRRT算法根据当前进程运行状态，通过一定函数决定下一个运行进程类似，CFS定义了一个变量：**`vruntime`**。该变量是进程运行时间、进程权重的函数，通过该变量来决定进程在一个调度周期内的运行顺序。

```c
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
```

4.采用维护一棵**红黑树**的方法在调度点决定任务的运行，就绪状态进程根据`vruntime`被放在红黑树的叶子节点上, 从小到大一次排列在从左到右。发生调度行为时, 调度器会从红黑树从左至右选择进程，这种方法的时间复杂度是O(log2N)。



## CFS的重要数据结构

### CFS调度类

CFS调度类采用面向对象的思想，其中：

`const struct sched_class *next;`指向下一级调度策略的调度类，`enqueue_task`函数将进程加入就绪队列，`dequeue_task`将进程从就绪队列中移除。`pick_next_task`选取下一个要运行的进程，`put_prev_task`获得当前进程之前的进程，等等。

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\sched.h
//为方便展示，删除掉一些信息
struct sched_class {
	const struct sched_class *next;

	void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
	void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);

	struct task_struct * (*pick_next_task)(struct rq *rq,
					       struct task_struct *prev,
					       struct rq_flags *rf);
	void (*put_prev_task)(struct rq *rq, struct task_struct *p);
    //等等
};
```

针对该调度类，定义了五种使用场景

```c
extern const struct sched_class stop_sched_class;//停机调度
extern const struct sched_class dl_sched_class;//限期调度
extern const struct sched_class rt_sched_class;//实时调度
extern const struct sched_class fair_sched_class;//公平调度,我们主要研究的
extern const struct sched_class idle_sched_class;//空闲调度
```



### CFS调度实体

每一个进程都是一个`sched_entity`结构体的实例。

`load`就是此进程的权重，`run_node`该进程在红黑树上的位置（地址），`on_rq`:值1在运行队列中，0不在，`exec_start`: 记录这个进程在CPU上开始执行任务的时间，`sum_exec_runtime`: 记录这个进程总的运行时间，`vruntime`: 虚拟运行时间。

```c
//位置；kernel-openEuler-1.0-LTS\include\linux\sched.h
//为方便展示，删除掉一些信息
struct sched_entity {
	/* For load-balancing: */
	struct load_weight		load;
	struct rb_node			run_node;
	struct list_head		group_node;
	unsigned int			on_rq;

	u64				exec_start;
	u64				sum_exec_runtime;
	u64				vruntime;
//进程的红黑树上位置的相关信息
#ifdef CONFIG_FAIR_GROUP_SCHED
	int				depth;//树中位置的深度
	struct sched_entity		*parent;//父节点
	struct cfs_rq			*cfs_rq;//所处哪棵树

#endif

	//等等
};
```



### CFS运行队列

该结构体的实例代表了当前正在执行CFS调度策略的运行队列。

`load`CFS就绪队列总体的权重，`nr_running`CFS就绪队列总体的可运行进程数，`min_vruntime`当前CFS就绪队列中各个进程的最小虚拟时间，用于对新进入进程进行惩罚。

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\sched.h
//为方便展示，删除掉一些信息
/* CFS-related fields in a runqueue */
struct cfs_rq {
	struct load_weight	load;
	unsigned long		runnable_weight;
	unsigned int		nr_running;
	unsigned int		h_nr_running;      /* SCHED_{NORMAL,BATCH,IDLE} */
    
    u64			exec_clock;
	u64			min_vruntime;
#ifndef CONFIG_64BIT
	u64			min_vruntime_copy;
#endif
	//等等
};

```



### 进程有关结构体

#### 调度类

```c
//位置；kernel-openEuler-1.0-LTS\include\linux\sched.h
//为方便展示，删除掉一些信息
struct task_struct {
    //等等
    int on_rq;

    int prio, static_prio, normal_prio;
    
    unsigned int rt_priority;
    
    const struct sched_class *sched_class;
    
    struct sched_entity se;
    
    struct sched_rt_entity rt;
#ifdef CONFIG_CGROUP_SCHED
    struct task_group *sched_task_group;
#endif
    //等等
}

```

#### 服务器的用户分组

当系统为服务器等多用户同时使用时，首先为用户分配CPU时间，再对单个用户的进程进行分配。

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\sched.h
//为方便展示，删除掉一些信息
/* Task group related information */
struct task_group {
	struct cgroup_subsys_state css;
	//等等
};
```



## CFS相关重要函数及参数

### 权重映射数组

该数组将取值区间为[-20,19]的 $nice$ 值映射为真实运算中使用的权重值，这样做既方便设置，又方便计算。

由代码可以观察到，`sched_prio_to_wmult[i]:sched_prio_to_wmult[i-1]`约等于 $9/11$。两个相邻权重进程分时比相差 $10%$。

由 $nice$ 得到 $weight$：

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\core.c
const int sched_prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};
```

${2^{32}}/weight$ 结果为：

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\core.c
const u32 sched_prio_to_wmult[40] = {
 /* -20 */     48388,     59856,     76040,     92818,    118348,
 /* -15 */    147320,    184698,    229616,    287308,    360437,
 /* -10 */    449829,    563644,    704093,    875809,   1099582,
 /*  -5 */   1376151,   1717300,   2157191,   2708050,   3363326,
 /*   0 */   4194304,   5237765,   6557202,   8165337,  10153587,
 /*   5 */  12820798,  15790321,  19976592,  24970740,  31350126,
 /*  10 */  39045157,  49367440,  61356676,  76695844,  95443717,
 /*  15 */ 119304647, 148102320, 186737708, 238609294, 286331153,
};
```



### `vruntime`计算函数

`calc_delta_fair`根据输入的实际运行时间、进程结果体，返回一个该结构体的`vruntime`值。

`NICE_0_LOAD`是一个常量，代表 $nice$ 值为0的权重

$vruntime = (delta\_time*NICE\_0\_LOAD)/weight$

具体运算中为避免浮点运算，采用公式：

$vruntime = \left( {(delta\_time*NICE\_0\_LOAD)*({2^{32}}/weight)} \right) >  > 32$

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\fair.c
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
	if (unlikely(se->load.weight != NICE_0_LOAD))//如果nice=0，虚拟、实际时间相同不计算
		delta = __calc_delta(delta, NICE_0_LOAD, &se->load);//计算虚拟时间

	return delta;
}
```

```c
static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw){
    ......
    return mul_u64_u32_shr(delta_exec, fact, shift);//返回计算得到的虚拟时间
}
```



### 调度周期内进程分配到的时间

`sched_slice`函数，通过输入的CFS队列、进程结构体，给出该进程在一个调度周期内的实际运行时间。

**调度周期**：首先根据进程是否在CFS队列中，确定下一个调度周期运行的进程数。根据进程数，如果小于`sched_nr_latency`（8）那么设置调度周期为6ms，如果大于8，设置为：进程数*0.75ms。这种得到调度周期的方法，有助于防止进程过快的进行上下文切换。

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\fair.c
static unsigned int sched_nr_latency = 8;
unsigned int sysctl_sched_latency			= 6000000ULL;
```

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\fair.c
static u64 __sched_period(unsigned long nr_running)
{
	if (unlikely(nr_running > sched_nr_latency))//如果进程数>8，0.75ms*进程数
		return nr_running * sysctl_sched_min_granularity;
	else
		return sysctl_sched_latency;//6ms
}
```

**实际运行时间的计算公式**：进程的运行时间 = 调度周期 * (进程的weight / CFS运行队列的总weigth)

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\fair.c
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);//获得调度周期长度

	for_each_sched_entity(se) {
		struct load_weight *load;
		struct load_weight lw;
		//CFS队列运行权重
		cfs_rq = cfs_rq_of(se);
		load = &cfs_rq->load;

		if (unlikely(!se->on_rq)) {
			lw = cfs_rq->load;

			update_load_add(&lw, se->load.weight);
			load = &lw;
		}
		slice = __calc_delta(slice, se->load.weight, load);//计算运行时间
	}
	return slice;
}
```



### 进程惩罚/照顾

在CFS算法中，可以看到`vruntime`决定了一个进程在CFS红黑树中的位置，也即运行的先后顺序。

当**新进程**进入时，`vruntime=0`会在一段时间内一直处于占用CPU的状态，因此要对其`vruntime`加以惩罚。具体操作为：获取当前CFS队列的最小的`vruntime`，并将其赋给新的进程。

当**进程刚刚被唤醒**，表示进程希望能够及时被执行，这种进程需要被照顾使之可以及时运行，此时将其`vruntime`减去调度延时的一半。

在如下程序中，输入`initial`用来决定是惩罚还是照顾。

```c
unsigned int sysctl_sched_latency			= 6000000ULL;
```

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\fair.c
//为方便展示，删除掉一些信息
static void
place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
{
	u64 vruntime = cfs_rq->min_vruntime;//获得该CFS队列的min_vruntime

	if (initial && sched_feat(START_DEBIT))//新进程惩罚
		vruntime += sched_vslice(cfs_rq, se);

	if (!initial) {//进程唤醒照顾
		unsigned long thresh = sysctl_sched_latency;

		if (sched_feat(GENTLE_FAIR_SLEEPERS))
			thresh >>= 1;

		vruntime -= thresh;
	}
	......
	se->vruntime = max_vruntime(se->vruntime, vruntime);
}
```



## CFS代码执行流程

### 新进程的初始化

当一个进程被创建时，`sched_fork()`实现流程的流程图如下：

<img src="https://i0.hdslb.com/bfs/album/c7722bc26a089ade393ba6cfccbe180921406023.png" alt="image-20220504094508656" style="zoom:66%;" />  

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\core.c
//为方便展示，删除掉一些信息
int sched_fork(unsigned long clone_flags, struct task_struct *p)
{
	__sched_fork(clone_flags, p);//调度实体，运行时间等初始化
	
	p->state = TASK_NEW;//设置为新进程

	p->prio = current->normal_prio;//继承父进程的优先级
	//如果需要，重新设置优先级
	if (unlikely(p->sched_reset_on_fork)) {
		if (task_has_dl_policy(p) || task_has_rt_policy(p)) {
			p->policy = SCHED_NORMAL;
			p->static_prio = NICE_TO_PRIO(0);
			p->rt_priority = 0;
		} else if (PRIO_TO_NICE(p->static_prio) < 0)
			p->static_prio = NICE_TO_PRIO(0);

		p->prio = p->normal_prio = __normal_prio(p);
		set_load_weight(p, false);//设置进程权重
	
		p->sched_reset_on_fork = 0;
	}
	//设置调度类的类型，普通进程/实时进程
	if (dl_prio(p->prio))
		return -EAGAIN;
	else if (rt_prio(p->prio))
		p->sched_class = &rt_sched_class;
	else
		p->sched_class = &fair_sched_class;
	......
}

```

在`sched_fork()`函数以后，如果进程被标记为普通进程，则调用`task_fork_fair()`为其使用CFS策略做准备。

（`se->vruntime -= cfs_rq->min_vruntime;`这一步操作有点看不明白）

<img src="https://i0.hdslb.com/bfs/album/d533fc899892376a4491a397716a863490e96853.png" alt="image-20220504100847865" style="zoom:66%;" /> 

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\fair.c
//为方便展示，删除掉一些信息
static void task_fork_fair(struct task_struct *p)
{
	struct cfs_rq *cfs_rq;//CFS调度队列
	struct sched_entity *se = &p->se, *curr;//获得CFS调度实体
	struct rq *rq = this_rq();
	struct rq_flags rf;

	cfs_rq = task_cfs_rq(current);//定位到父进程所在的CFS队列
	curr = cfs_rq->curr;
    
	if (curr) {//将父进程虚拟时间赋给新进程
		update_curr(cfs_rq);//更新
		se->vruntime = curr->vruntime;
	}
	place_entity(cfs_rq, se, 1);//新进程虚拟时间惩罚
	//将进程放置在该CFS红黑树的'current'位置
	if (sysctl_sched_child_runs_first && curr && entity_before(curr, se)) {
		swap(curr->vruntime, se->vruntime);
		resched_curr(rq);
	}
	//减去min_vruntime，因为当前周期未结束，min_vruntime未更新，进程还没加入调度队列
	se->vruntime -= cfs_rq->min_vruntime;//place_entity与这一步是不是多余？
	rq_unlock(rq, &rf);
    ......
}
```



### 新进程加入就绪队列

在`do_fork()`中的`wake_up_new_task(p)`，新进程正式加入CFS队列之中。

<img src="https://i0.hdslb.com/bfs/album/d0b95ce0f3a7db9123bb0a43c83d9e0f44a36bb5.png" alt="image-20220504103415504" style="zoom:67%;" />  

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\core.c
//为方便展示，删除掉一些信息
void wake_up_new_task(struct task_struct *p)
{
	struct rq_flags rf;
	struct rq *rq;

	raw_spin_lock_irqsave(&p->pi_lock, rf.flags);
	p->state = TASK_RUNNING;//设置进程状态为就绪态
    
#ifdef CONFIG_SMP
	p->recent_used_cpu = task_cpu(p);
	rseq_migrate(p);
	__set_task_cpu(p, select_task_rq(p, task_cpu(p), SD_BALANCE_FORK, 0));//选择CPU
#endif
    //进程加入CPU队列
	post_init_entity_util_avg(&p->se);
	//将新创建的进程加入到就绪队列
	activate_task(rq, p, ENQUEUE_NOCLOCK);
	......
}
```

加入就绪队列具体操作：

`enqueue_entity()`具体实现调度实体的入队工作。

CFS进程数加一，调度实体修改其`vruntime`值，**调度实体加入红黑树**。

```c
//将新创建的进程加入到就绪队列
void activate_task(struct rq *rq, struct task_struct *p, int flags)
static inline void enqueue_task(struct rq *rq, struct task_struct *p, int flags)
    
//通过enqueue_entity函数将调度实体入队
//增加CFS运行队列的可运行个数h_nr_running      
static void
enqueue_task_fair(struct rq *rq, struct task_struct *p, int flags)
    
//为新进程调度实体添加cfs_rq->min_vruntime 
//更新当前调度实体的运行时间以及CFS的min_vruntime
//__enqueue_entity将调度实体添加到CFS红黑树中
static void
enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
	bool renorm = !(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_MIGRATED);
	bool curr = cfs_rq->curr == se;

	if (renorm && curr)//vruntime修正
		se->vruntime += cfs_rq->min_vruntime;

	update_curr(cfs_rq);//更新CFS状态

	if (renorm && !curr)
		se->vruntime += cfs_rq->min_vruntime;

	......
        
	if (!curr)
		__enqueue_entity(cfs_rq, se);//加入红黑树
	se->on_rq = 1;//设置该调度实体的状态为：已加入队列

	if (cfs_rq->nr_running == 1) {//处理队列中只有一个进程的情况
		list_add_leaf_cfs_rq(cfs_rq);
		check_enqueue_throttle(cfs_rq);
	}
}
```



### 就绪队列中进程的选择运行

当CFS开始选择进程进行运行时，具体的操作如下：

<img src="https://i0.hdslb.com/bfs/album/c7a53616763c80e24116716e0f506414e5f352de.png" alt="image-20220504151140078" style="zoom:70%;" />  

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\core.c
//为方便展示，删除掉一些信息
static void __sched notrace __schedule(bool preempt)
{
	struct task_struct *prev, *next;
	......

	cpu = smp_processor_id();//获取CPU
	rq = cpu_rq(cpu);//该CPU的队列
	prev = rq->curr;//通过指针找到当前运行进程
	......

	next = pick_next_task(rq, prev, &rf);//获得下一个CFS进程指针
	......

	if (likely(prev != next)) {//当前进程与下一个进程不同，则发生上下文切换
		......
		rq = context_switch(rq, prev, next, &rf);
	} else {
		......
	}
	......
}  
```

`pick_next_task()`函数，如果CFS队列进程个数等于当前进程调度类的进程个数，直接调用`fair_sched_class`中的`pick_next`，否则按照调度类的优先级从高往低依次遍历调用`pick_next_task`函数指针。

```c
//获得下一个CFS进程指针
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
```

如果CFS队列没有进程，返回idle。

`pick_next_entity`获得CFS红黑树最左边的实体。

`set_next_entity`使`next`设置在当前的指针

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\fair.c
//为方便展示，删除掉一些信息
static struct task_struct *
pick_next_task_fair(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
	struct cfs_rq *cfs_rq = &rq->cfs;
	struct sched_entity *se;
	struct task_struct *p;
	int new_tasks;
	unsigned long time;

again:
	if (!cfs_rq->nr_running)
		goto idle;

	......

	do {
		se = pick_next_entity(cfs_rq, NULL);
		set_next_entity(cfs_rq, se);
		cfs_rq = group_cfs_rq(se);
	} while (cfs_rq);
    
	......
}
```



### 进程的周期性调度

Linux内核中，进程会在**每一个tick**的时候更新进程的状态并周期性地检查当前进程是否耗尽了自己应该分得的一个调度周期里面对应的时间片，并以此判断是否需要调度。

每一个tick发生一次对当前进程的调度考察，具体流程如下：

<img src="https://i0.hdslb.com/bfs/album/d14f99557c924a1d37baf47c6adde57a3ea7a05a.png" alt="image-20220504171609539" style="zoom:67%;" /> 

```c
static void
entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued){
	......
    //更新当前进程的执行时间，vruntime。CFS的min_vruntime
    update_curr(cfs_rq);
    ......
}
```

```c
//位置：kernel-openEuler-1.0-LTS\kernel\sched\fair.c
//为方便展示，删除掉一些信息
static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
	unsigned long ideal_runtime, delta_exec;
	struct sched_entity *se;
	s64 delta;
	//获取当前进程在一个调度周期中理想的运行时间
	ideal_runtime = sched_slice(cfs_rq, curr);
    //sum_exec_runtime 进程到一次调度时，总共执行的时间；
    //prev_sum_exec_runtime 上次调度出去的执行时间
    //delta_exec 本次调度周期中实际运行的时间
	delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    //若delta_exec大于ideal_runtime，置need_resched，应该发生调度
	if (delta_exec > ideal_runtime) {
		resched_curr(rq_of(cfs_rq));
		clear_buddies(cfs_rq, curr);
		return;
	}
    //防止ideal_runtime分配时间过少，频繁过度抢占
	//若delta_exec小于sysctl_sched_min_granularity调度周期中最少运行时间，不需要调度
	if (delta_exec < sysctl_sched_min_granularity)
		return;
	
	se = __pick_first_entity(cfs_rq);
	delta = curr->vruntime - se->vruntime;
	//比较当前进程vruntime与CFS红黑树最左进程的vruntime
	if (delta < 0)
		return;
    //理解为：当前进程领先其他进程一个调度周期，所以应当礼让
	//如果delta大于ideal_runtime，需要调度。
	if (delta > ideal_runtime)
		resched_curr(rq_of(cfs_rq));
}
```



## 总结

​		**CFS**根据**权重**、**运行时间**抽象出了**`vruntime`**的概念，通过维护一个从左到右`vruntime`递增的**红黑树**来对就绪进程的先后运行顺序进行调度。在具体执行调度时，所有就绪进程在一个**调度周期**内根据权重**分配运行时间**，并在每一个**`tick`**检测是否需要被调度。
