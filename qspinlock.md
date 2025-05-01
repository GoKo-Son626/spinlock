<!--
 * @Date: 2025-03-25
 * @LastEditors: Goko Son
 * @LastEditTime: 2025-04-30
 * @FilePath: /spinlock/qspinlock.md
 * @Description: 
--> 
## spinlock

原子操作几乎是基于硬件实现的，但是保护的范围有限，比如CAS指令最大支持64bit（8字节大小），但是实际链表，二叉树等操作集合很大，对其操作是一个过程，不适合，自旋锁会让内核代码路径（进程或中断处理程序）自旋等待，锁可以分为两种：**忙等待和睡眠等待**，自旋锁的临界区中的代码要尽快执行完成，不能调度或者睡眠，自旋锁可以在（软，硬件，不可屏蔽）中断上下文或进程上下文使用，即CPU只能在自旋锁的临界区里面执行。


#### 1. spinlock：

- 有公平性问题，缓存一致性开销，CPU核心越大，cache需求越厉害，缺乏可扩展性

假设CPU0....同时抢占一个经典的spinlock锁
![alt text](image-6.png)

#### 2. Ticket spinlock：
```diff
diff --git a/arch/riscv/include/asm/spinlock_types.h b/arch/riscv/include/asm/spinlock_types.h
index f398e76..d7b38bf 100644
--- a/arch/riscv/include/asm/spinlock_types.h
+++ b/arch/riscv/include/asm/spinlock_types.h 
@@ -10,16 +10,21 @@
 # error "please don't include this file directly"
 #endif
 
+#define TICKET_NEXT	16
+
 typedef struct {
-	volatile unsigned int lock;
+	union {
+		u32 lock;
+		struct __raw_tickets {
+			/* little endian */
+			u16 owner;
+			u16 next;
+		} tickets;
+	};
 } arch_spinlock_t;
```
- 解决了公平问题，但所有核都轮询同一个owner变量，read cache line成热点，限制扩展性


#### 3. MCS lock：

- 本质上是一种基于链表结构的自旋锁，每个CPU有一个对应的节点，基于各自不同的变量进行等待，那么每个CPU平时只需要查询自己对应的这个变量所在的本地cache line，仅在这个变量发生变化的时候，才需要读取内存和刷新这条cache line, 无共享变量轮询（不像 classic/ticket）

```c
struct mcs_spinlock {
	struct mcs_spinlock *next;
	int locked; /* 1 if lock acquired */
	int count;  /* nesting count, see qspinlock.c */
};
```
- 每个 CPU 线程创建的 node 结构都是一样的，但它们是独立的，每个线程都有自己的 node 实例。“内存开销问题”


#### 4. qspinlock
**include/asm-generic/qspinlock_types.h:** 锁数据结构
```c
typedef struct qspinlock {
        union {
                atomic_t val;

                /*
                 * By using the whole 2nd least significant byte for the
                 * pending bit, we can allow better optimization of the lock
                 * acquisition for the pending bit holder.
                 */
#ifdef __LITTLE_ENDIAN
                struct {
                        u8      locked;
                        u8      pending;
                };
                struct {
                        u16     locked_pending;
                        u16     tail;
                };
#else
                struct {
                        u16     tail;
                        u16     locked_pending;
                };
                struct {
                        u8      reserved[2];
                        u8      pending;
                        u8      locked;
                };
#endif
        };
} arch_spinlock_t;

/*
 * Initializier
 */
#define __ARCH_SPIN_LOCK_UNLOCKED       { { .val = ATOMIC_INIT(0) } }

/*
 * Bitfields in the atomic value:
 *
 * When NR_CPUS < 16K
 *  0- 7: locked byte
 *     8: pending
 *  9-15: not used
 * 16-17: tail index
 * 18-31: tail cpu (+1)
 *
 * When NR_CPUS > = 16K
 *  0- 7: locked byte
 *     8: pending
 *  9-10: tail index
 * 11-31: tail cpu (+1)
 */
#define _Q_SET_MASK(type)       (((1U << _Q_ ## type ## _BITS) - 1)\
                                      << _Q_ ## type ## _OFFSET)
#define _Q_LOCKED_OFFSET        0
#define _Q_LOCKED_BITS          8
#define _Q_LOCKED_MASK          _Q_SET_MASK(LOCKED)
```
![alt text](image-2.png)
- bit16-17: tail_index: 获取节点，目前Cpu支持4种节点(1234)
- cpu编号，这里用来标识mcis链表末尾的节点

**kernel/locking/mcs_spinlock.h**
```c
struct mcs_spinlock {
        struct mcs_spinlock *next;
        int locked; /* 1 if lock acquired */
        int count;  /* nesting count, see qspinlock.c */
};
```
`locked = 1`:只是说锁传到了当前加节点，但是当前节点还需要主动申请锁(qspinlock-> locked = 1)
`count`：四种上下文

**kernel/locking/qspinlock.c:**
```c
#define MAX_NODES       4

struct qnode {
        struct mcs_spinlock mcs;
#ifdef CONFIG_PARAVIRT_SPINLOCKS
        long reserved[2];
#endif
};

/*
 * Per-CPU queue node structures; we can never have more than 4 nested
 * contexts: task, softirq, hardirq, nmi.
 *
 * Exactly fits one 64-byte cacheline on a 64-bit architecture.
 *
 * PV doubles the storage and uses the second cacheline for PV state.
 */
static DEFINE_PER_CPU_ALIGNED(struct qnode, qnodes[MAX_NODES]);
```
- 嵌套四种上下文情况下的锁，例：进程上下文中发生中断再次获取锁



**申请锁：**
**include/asm-generic/qspinlock.h**
![CPU快速申请](image-3.png)
```c
/**
 * queued_spin_lock - acquire a queued spinlock
 * @lock: Pointer to queued spinlock structure
 */
static __always_inline void queued_spin_lock(struct qspinlock *lock)
{
	int val = 0;

	if (likely(atomic_try_cmpxchg_acquire(&lock->val, &val, _Q_LOCKED_VAL)))
		return;

	queued_spin_lock_slowpath(lock, val);
}
```
![CPU中速申请](image-4.png)

- 快速申请失败，queue中为空时，设置锁的pending位
- 再次检测（检查中间是否有其它cpu进入）
- 一直循环检测locked位
- 清除pending位



![alt text](image-5.png)
0,1,1(自旋等待锁) ->  0,0,1(得到锁)：退出循环

| 申请         | 操作                                                                                                                                                |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| 快速申请 | 这个锁当前没有人持有，直接通过cmpxchg()设置locked域即可获取了锁。                                                                                   |
| 中速申请 | 锁已经被人持有，但是MCS链表没有其他人，有且仅有一个人在等待这个锁。设置pending域，表示是第一顺位继承者，自旋等待lock-> locked清0，即锁持有者释放锁。 |
|慢速申请| 进入到queue中自旋等待


- 如果只有1个或2个CPU试图获取锁，那么只需要一个4字节的qspinlock就可以了，其所占内存的大小和ticket spinlock一样。当有3个以上的CPU试图获取锁，需要一个qspinlock加上(N-2)个MCS node。
在大多数情况下，锁的争抢都不应该太激烈，最大概率是只有1个CPU试图获得锁，其次是2个，依次递减。

- 这也是qspinlock中加入"pending"位域的意义，如果是两个CPU试图获取锁，那么第二个CPU只需要简单地设置"pending"为1，而不用创建一个MCS node。这是另一个优化。
- 试图加锁的CPU数目超过3个，使用ticket spinlock机制就会造成多个CPU的cache line刷新的问题，而qspinlock可以利用MCS node队列来解决这个问题。






- val == _Q_PENDING_VAL:0,1,0 ->  锁处于临界状态，锁的持有者正在释放锁

