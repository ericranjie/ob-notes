
作者：[安庆](http://www.wowotech.net/author/539 "oppo混合云内核&虚拟化负责人，架构并孵化了oppo的云游戏，云手机等产品。") 发布于：2021-5-8 8:36 分类：[Linux内核分析](http://www.wowotech.net/sort/linux_kenrel)

# 已经运行多年的jbd2，它还是死锁了

## 背景：这个是在centos7的环境上复现的，内核版本为3.10.0-957.27.2.el7

### 下面列一下我们是怎么排查并解这个问题的。

#### 一、故障现象

oppo云内核团队接到运维兄弟收集的测试环境一例crash,

现象是load很高,卡得没法操作：

```c

      KERNEL: /usr/lib/debug/lib/modules/3.10.0-957.27.2.el7.x86_64/vmlinux

    DUMPFILE: vmcore  [PARTIAL DUMP]

        CPUS: 40

        DATE: Mon Apr 19 15:46:07 2021----收集crash的日期

      UPTIME: 312 days, 00:49:56

LOAD AVERAGE: 72886.67, 72873.82, 72735.93

  

```

我们授权后登陆oppo云宿主，查看最早报hung的日志如下：

```c

Apr 16 19:46:22 [ERR] :  - [26652536.550311] INFO: task jbd2/dm-12-8:547025 blocked for more than 120 seconds.

Apr 16 19:46:22 [ERR] :  - [26652536.550342] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.

Apr 16 19:46:22 [WARNING] :  - [26652536.550372] Call Trace:

Apr 16 19:46:22 [WARNING] :  - [26652536.550380]  [<ffffffff8a76ae39>] schedule_preempt_disabled+0x29/0x70

Apr 16 19:46:22 [WARNING] :  - [26652536.550383]  [<ffffffff8a768db7>] __mutex_lock_slowpath+0xc7/0x1d0

Apr 16 19:46:22 [WARNING] :  - [26652536.550385]  [<ffffffff8a76819f>] mutex_lock+0x1f/0x2f

Apr 16 19:46:22 [WARNING] :  - [26652536.550405]  [<ffffffffc03f17b8>] jbd2_update_log_tail+0x28/0x60 [jbd2]

Apr 16 19:46:22 [WARNING] :  - [26652536.550410]  [<ffffffffc03ea5dd>] jbd2_journal_commit_transaction+0x155d/0x19b0 [jbd2]

Apr 16 19:46:22 [WARNING] :  - [26652536.550417]  [<ffffffffc03efe89>] kjournald2+0xc9/0x260 [jbd2]---主函数

Apr 16 19:46:22 [WARNING] :  - [26652536.550423]  [<ffffffff8a0c3f50>] ? wake_up_atomic_t+0x30/0x30

Apr 16 19:46:22 [WARNING] :  - [26652536.550428]  [<ffffffffc03efdc0>] ? commit_timeout+0x10/0x10 [jbd2]

Apr 16 19:46:22 [WARNING] :  - [26652536.550430]  [<ffffffff8a0c2e81>] kthread+0xd1/0xe0

Apr 16 19:46:22 [WARNING] :  - [26652536.550433]  [<ffffffff8a0c2db0>] ? insert_kthread_work+0x40/0x40

Apr 16 19:46:22 [WARNING] :  - [26652536.550437]  [<ffffffff8a776c1d>] ret_from_fork_nospec_begin+0x7/0x21

Apr 16 19:46:22 [WARNING] :  - [26652536.550439]  [<ffffffff8a0c2db0>] ? insert_kthread_work+0x40/0x40

Apr 16 19:46:22 [ERR] :  - [26652536.550467] INFO: task java:12610 blocked for more than 120 seconds.

```

# 二、故障现象分析

既然最早报hung的进程为jbd2/dm-12-8:547025，那就需要先看下该进程的堆栈，

```c
crash> bt 547025
PID: 547025  TASK: ffff91792aff1040  CPU: 16  COMMAND: "jbd2/dm-12-8"
 #0 [ffff91a907247b68] __schedule at ffffffff8a769a72
 #1 [ffff91a907247bf8] schedule_preempt_disabled at ffffffff8a76ae39
 #2 [ffff91a907247c08] __mutex_lock_slowpath at ffffffff8a768db7
 #3 [ffff91a907247c60] mutex_lock at ffffffff8a76819f
 #4 [ffff91a907247c78] jbd2_update_log_tail at ffffffffc03f17b8 [jbd2]
 #5 [ffff91a907247ca8] jbd2_journal_commit_transaction at ffffffffc03ea5dd [jbd2]
 #6 [ffff91a907247e48] kjournald2 at ffffffffc03efe89 [jbd2]
 #7 [ffff91a907247ec8] kthread at ffffffff8a0c2e81
 #8 [ffff91a907247f50] ret_from_fork_nospec_begin at ffffffff8a776c1d
crash> ps -m 547025

[  2 19:52:41.384] [UN]  PID: 547025  TASK: ffff91792aff1040  CPU: 16  COMMAND: "jbd2/dm-12-8"

```

从上面ps -m的打印看，该进程最后一次调度的时间是67时52分之前，也就是\[  2 19:52:41.384\]：

按照当前的时间点减去报hung的时间点，说明这个进程报hung之后，没有再经历调度。

下面就需要看，这个进程卡在什么地方,代码如下

```c

void jbd2_update_log_tail(journal_t *journal, tid_t tid, unsigned long block)

{

  mutex_lock(&journal->j_checkpoint_mutex);//caq:这里需要拿锁

  if (tid_gt(tid, journal->j_tail_sequence))

    __jbd2_update_log_tail(journal, tid, block);

  mutex_unlock(&journal->j_checkpoint_mutex);

}

```

从堆栈和代码可以确定，是在等待 journal->j_checkpoint_mutex。

然后看mutex放在栈里的位置：

```c

crash> dis -l jbd2_update_log_tail |grep mutex_lock

0xffffffffc03f17b3 <jbd2_update_log_tail+35>: callq  0xffffffff8a768180 <mutex_lock>

crash> dis -l jbd2_update_log_tail |grep mutex_lock -B 10

0xffffffffc03f1796 <jbd2_update_log_tail+6>:  mov    %rsp,%rbp

0xffffffffc03f1799 <jbd2_update_log_tail+9>:  push   %r14

0xffffffffc03f179b <jbd2_update_log_tail+11>: mov    %rdx,%r14

0xffffffffc03f179e <jbd2_update_log_tail+14>: push   %r13

0xffffffffc03f17a0 <jbd2_update_log_tail+16>: mov    %esi,%r13d

0xffffffffc03f17a3 <jbd2_update_log_tail+19>: push   %r12

0xffffffffc03f17a5 <jbd2_update_log_tail+21>: lea    0xf0(%rdi),%r12

0xffffffffc03f17ac <jbd2_update_log_tail+28>: push   %rbx

0xffffffffc03f17ad <jbd2_update_log_tail+29>: mov    %rdi,%rbx

0xffffffffc03f17b0 <jbd2_update_log_tail+32>: mov    %r12,%rdi---x86的arch，说明是r12是第一个入参

0xffffffffc03f17b3 <jbd2_update_log_tail+35>: callq  0xffffffff8a768180 <mutex_lock>

```

然后看mutex_lock里面，并没有人动r12寄存器，

```c

dis -l mutex_lock|grep r12 ---没有输出，说明这个函数里面没有人动r12

```

再往下面调用链看：

```c

crash> dis -l __mutex_lock_slowpath |head -12

/usr/src/debug/kernel-3.10.0-957.27.2.el7/linux-3.10.0-957.27.2.el7.x86_64/kernel/mutex.c: 771

0xffffffff8a768cf0 <__mutex_lock_slowpath>: nopl   0x0(%rax,%rax,1) [FTRACE NOP]

0xffffffff8a768cf5 <__mutex_lock_slowpath+5>: push   %rbp

0xffffffff8a768cf6 <__mutex_lock_slowpath+6>: mov    %rsp,%rbp

0xffffffff8a768cf9 <__mutex_lock_slowpath+9>: push   %r15

0xffffffff8a768cfb <__mutex_lock_slowpath+11>:  push   %r14

0xffffffff8a768cfd <__mutex_lock_slowpath+13>:  push   %r13

/usr/src/debug/kernel-3.10.0-957.27.2.el7/linux-3.10.0-957.27.2.el7.x86_64/arch/x86/include/asm/current.h: 14

0xffffffff8a768cff <__mutex_lock_slowpath+15>:  mov    %gs:0x10e80,%r13

/usr/src/debug/kernel-3.10.0-957.27.2.el7/linux-3.10.0-957.27.2.el7.x86_64/kernel/mutex.c: 771

0xffffffff8a768d08 <__mutex_lock_slowpath+24>:  push   %r12----r12在此被压栈，从这取mutex就行

0xffffffff8a768d0a <__mutex_lock_slowpath+26>:  push   %rbx

```

根据上面分析，从栈中取得mutex如下：

```c

crash> struct mutex ffff91793cbbe0f0----从栈中取的mutex

struct mutex {----从源码的上下文看，这个其实就是journal_t.j_checkpoint_mutex

  count = {

    counter = -9

  }, 

  wait_lock = {

    {

      rlock = {

        raw_lock = {

          val = {

            counter = 0

          }

        }

      }

    }

  }, 

  wait_list = {

    next = 0xffff91a907247c10, 

    prev = 0xffff91612cb139c0

  }, 

  owner = 0xffff9178e0ad1040, ----对应的owner 

  {

    osq = {

      tail = {

        counter = 0

      }

    }, 

    __UNIQUE_ID_rh_kabi_hide1 = {

      spin_mlock = 0x0

    }, 

    {<No data fields>}

  }

}

```

下面，就需要根据mutex的owner，看一下对应的进程为什么持有锁之后，长时间不放锁了。

```c

crash> bt 0xffff9178e0ad1040----对应的进程为13027

PID: 13027  TASK: ffff9178e0ad1040  CPU: 10  COMMAND: "java"

 #0 [ffff91791321f828] __schedule at ffffffff8a769a72

 #1 [ffff91791321f8b8] schedule at ffffffff8a769f19

 #2 [ffff91791321f8c8] jbd2_log_wait_commit at ffffffffc03ef7c5 [jbd2]

 #3 [ffff91791321f940] jbd2_log_do_checkpoint at ffffffffc03ec0bf [jbd2]

 #4 [ffff91791321f9a8] __jbd2_log_wait_for_space at ffffffffc03ec4bf [jbd2]---持有j_checkpoint_mutex

 #5 [ffff91791321f9f0] add_transaction_credits at ffffffffc03e63d3 [jbd2]

 #6 [ffff91791321fa50] start_this_handle at ffffffffc03e65e1 [jbd2]

 #7 [ffff91791321fae8] jbd2__journal_start at ffffffffc03e6a93 [jbd2]

 #8 [ffff91791321fb30] __ext4_journal_start_sb at ffffffffc05abc69 [ext4]

 #9 [ffff91791321fb70] ext4_da_write_begin at ffffffffc057e099 [ext4]

#10 [ffff91791321fbf8] generic_file_buffered_write at ffffffff8a1b73a4

#11 [ffff91791321fcc0] __generic_file_aio_write at ffffffff8a1b9ce2

#12 [ffff91791321fd40] generic_file_aio_write at ffffffff8a1b9f59

#13 [ffff91791321fd80] ext4_file_write at ffffffffc0573322 [ext4]

#14 [ffff91791321fdf0] do_sync_write at ffffffff8a241c13

#15 [ffff91791321fec8] vfs_write at ffffffff8a242700

#16 [ffff91791321ff08] sys_write at ffffffff8a24351f

#17 [ffff91791321ff50] system_call_fastpath at ffffffff8a776ddb

    RIP: 00007fd9b3b0569d  RSP: 00007fd8b1f6d680  RFLAGS: 00000293

    RAX: 0000000000000001  RBX: 00007fd8b1f6d820  RCX: 0000000000000010

    RDX: 0000000000000046  RSI: 00007fd8b1f6b7c0  RDI: 00000000000000ee

    RBP: 00007fd8b1f6b790   R8: 00007fd8b1f6b7c0   R9: 00000005c5741848

    R10: 0000000000216546  R11: 0000000000000293  R12: 0000000000000046

    R13: 00007fd8b1f6b7c0  R14: 00000000000000ee  R15: 0000000000000046

    ORIG_RAX: 0000000000000001  CS: 0033  SS: 002b

```

13027也在等待，处于UN状态，它的最后一次调度时间为：

```c

crash> ps -m 13027

[  2 19:52:46.519] [UN]  PID: 13027  TASK: ffff9178e0ad1040  CPU: 10  COMMAND: "java"

对比一下报hung的jbd2进程：

crash> ps -m 547025

[  2 19:52:41.384] [UN]  PID: 547025  TASK: ffff91792aff1040  CPU: 16  COMMAND: "jbd2/dm-12-8"

```

我们发现，其实13027处于UN的时间应该比547025还要多5秒，为啥不是它触发了hung的检测呢？这个问题留在后面

解释。

下面，我们就需要继续分析 13027 为什么持有journal_t.j_checkpoint_mutex锁却没有及时释放。

从代码分析 \_\_jbd2_log_wait_for_space

```c

void __jbd2_log_wait_for_space(journal_t *journal)

{

  ......

  while (jbd2_log_space_left(journal) < nblocks) {//caq：空间不够

    write_unlock(&journal->j_state_lock);

    mutex_lock(&journal->j_checkpoint_mutex);

  

    /*

     * Test again, another process may have checkpointed while we

     * were waiting for the checkpoint lock. If there are no

     * transactions ready to be checkpointed, try to recover

     * journal space by calling cleanup_journal_tail(), and if

     * that doesn't work, by waiting for the currently committing

     * transaction to complete.  If there is absolutely no way

     * to make progress, this is either a BUG or corrupted

     * filesystem, so abort the journal and leave a stack

     * trace for forensic evidence.

     */

    write_lock(&journal->j_state_lock);

    if (journal->j_flags & JBD2_ABORT) {

      mutex_unlock(&journal->j_checkpoint_mutex);

      return;

    }

    spin_lock(&journal->j_list_lock);

    nblocks = jbd2_space_needed(journal);

    space_left = jbd2_log_space_left(journal);//caq:再次获取一次数据

    if (space_left < nblocks) {//caq:空间还是不够

      int chkpt = journal->j_checkpoint_transactions != NULL;

      tid_t tid = 0;

  

      if (journal->j_committing_transaction)

        tid = journal->j_committing_transaction->t_tid;

      spin_unlock(&journal->j_list_lock);

      write_unlock(&journal->j_state_lock);

      if (chkpt) {

        jbd2_log_do_checkpoint(journal);//caq:此时持有 j_checkpoint_mutex

......

}

```

\_\_jbd2_log_wait_for_space在持有 j_checkpoint_mutex的情况下，进入

jbd2_log_do_checkpoint并在等待log提交，等待完成的条件是当前提交的

tid必须要不大于 j_commit_sequence。

```c

int jbd2_log_wait_commit(journal_t *journal, tid_t tid)

{

  int err = 0;

  

  read_lock(&journal->j_state_lock);

#ifdef CONFIG_JBD2_DEBUG

  if (!tid_geq(journal->j_commit_request, tid)) {

    printk(KERN_ERR

           "%s: error: j_commit_request=%d, tid=%d\n",

           __func__, journal->j_commit_request, tid);

  }

#endif

  while (tid_gt(tid, journal->j_commit_sequence)) {---只要当前tid大于j_commit_sequence

    jbd_debug(1, "JBD2: want %d, j_commit_sequence=%d\n",

          tid, journal->j_commit_sequence);

    read_unlock(&journal->j_state_lock);

    wake_up(&journal->j_wait_commit);

    wait_event(journal->j_wait_done_commit,

      !tid_gt(tid, journal->j_commit_sequence));--就会等到tid不大于j_commit_sequence为止

    read_lock(&journal->j_state_lock);

  }

  read_unlock(&journal->j_state_lock);

  

  if (unlikely(is_journal_aborted(journal)))

    err = -EIO;

  return err;

}

```

根据栈中的详细数据：

```c

crash> bt -f 13027

PID: 13027  TASK: ffff9178e0ad1040  CPU: 10  COMMAND: "java"

 #0 [ffff91791321f828] __schedule at ffffffff8a769a72

    ffff91791321f830: 0000000000000082 ffff91791321ffd8 

    ffff91791321f840: ffff91791321ffd8 ffff91791321ffd8 

    ffff91791321f850: 000000000001ab80 ffff917a3761e180 

    ffff91791321f860: ffff91791321f870 ffffffff8a75e2cb 

    ffff91791321f870: ffff91791321f8f8 0000000065353fc1 

    ffff91791321f880: 0000000000000246 ffff91793cbbe000 

    ffff91791321f890: 000000006f19bbe9 ffff91793cbbe090 

    ffff91791321f8a0: ffff91793cbbe028 ffff91791321f8f8 

    ffff91791321f8b0: ffff91791321f8c0 ffffffff8a769f19 

 #1 [ffff91791321f8b8] schedule at ffffffff8a769f19

    ffff91791321f8c0: ffff91791321f938 ffffffffc03ef7c5 

 #2 [ffff91791321f8c8] jbd2_log_wait_commit at ffffffffc03ef7c5 [jbd2]

    ffff91791321f8d0: ffff9178e0ad1040 ffff91793cbbe0a8 

    ffff91791321f8e0: 0000000000000000 ffff9178e0ad1040 

    ffff91791321f8f0: ffffffff8a0c3f50 ffff91793cbbe098 

    ffff91791321f900: ffff91793cbbe098 0000000065353fc1 

    ffff91791321f910: ffff91a925027700 ffff91793cbbe000 ---r12就是journal_t

    ffff91791321f920: ffff91793cbbe3a0 ffff9186a0a3a1a0 

    ffff91791321f930: 000000006f19bbe9 ffff91791321f9a0 ---r15就是000000006f19bbe9

    ffff91791321f940: ffffffffc03ec0bf 

 #3 [ffff91791321f940] jbd2_log_do_checkpoint at ffffffffc03ec0bf [jbd2]

```

当前的tid值，可以从栈中取出为：

```c

查看当前tid为：

crash> p 0x6f19bbe9

$2 = 1863957481

crash> journal_t.j_commit_sequence ffff91793cbbe000

  j_commit_sequence = 1863957480

```

熟悉jbd2代码的同学到此肯定能看出问题来了，因为要想j_commit_sequence 不停地增长并完成，得交给

对应设备的jbd2进程，而刚好，它又在等待我们的13027 持有的 j_checkpoint_mutex，两者明显死锁了。

#### 三、故障复现&修复

1.\_\_jbd2_log_wait_for_space调用 jbd2_log_wait_commit 的两个分支，

一个是jbd2_log_do_checkpoint-->\_\_process_buffer-->jbd2_log_wait_commit,

在这个分支是持有j_checkpoint_mutex的，

而在另外一个分支：

jbd2_log_do_checkpoint-->jbd2_log_wait_commit，是先释放了j_checkpoint_mutex再进入的

jbd2_log_wait_commit。

```c

void __jbd2_log_wait_for_space(journal_t *journal)

{

  int nblocks, space_left;

  /* assert_spin_locked(&journal->j_state_lock); */

  

  nblocks = jbd2_space_needed(journal);

  while (jbd2_log_space_left(journal) < nblocks) {//caq：空间不够

    write_unlock(&journal->j_state_lock);

    mutex_lock(&journal->j_checkpoint_mutex);---持有锁

  

    /*

     * Test again, another process may have checkpointed while we

     * were waiting for the checkpoint lock. If there are no

     * transactions ready to be checkpointed, try to recover

     * journal space by calling cleanup_journal_tail(), and if

     * that doesn't work, by waiting for the currently committing

     * transaction to complete.  If there is absolutely no way

     * to make progress, this is either a BUG or corrupted

     * filesystem, so abort the journal and leave a stack

     * trace for forensic evidence.

     */

    write_lock(&journal->j_state_lock);

    if (journal->j_flags & JBD2_ABORT) {

      mutex_unlock(&journal->j_checkpoint_mutex);

      return;

    }

    spin_lock(&journal->j_list_lock);

    nblocks = jbd2_space_needed(journal);

    space_left = jbd2_log_space_left(journal);//caq:再次获取一次数据

    if (space_left < nblocks) {//caq:空间还是不够

      int chkpt = journal->j_checkpoint_transactions != NULL;

      tid_t tid = 0;

  

      if (journal->j_committing_transaction)

        tid = journal->j_committing_transaction->t_tid;

      spin_unlock(&journal->j_list_lock);

      write_unlock(&journal->j_state_lock);

      if (chkpt) {----这个分支，持有j_checkpoint_mutex

        jbd2_log_do_checkpoint(journal);//caq:注意此时持有 j_checkpoint_mutex

      } else if (jbd2_cleanup_journal_tail(journal) == 0) {

        /* We were able to recover space; yay! */

        ;

      } else if (tid) {---而这个分支，同样是可能会调用 jbd2_log_wait_commit，正常释放了锁

        /*

         * jbd2_journal_commit_transaction() may want

         * to take the checkpoint_mutex if JBD2_FLUSHED

         * is set.  So we need to temporarily drop it.

         */

        mutex_unlock(&journal->j_checkpoint_mutex);//caq:这个分支在此释放了锁

        jbd2_log_wait_commit(journal, tid);

        write_lock(&journal->j_state_lock);

        continue;

      } else {

        printk(KERN_ERR "%s: needed %d blocks and "

               "only had %d space available\n",

               __func__, nblocks, space_left);

        printk(KERN_ERR "%s: no way to get more "

               "journal space in %s\n", __func__,

               journal->j_devname);

        WARN_ON(1);

        jbd2_journal_abort(journal, 0);

      }

      write_lock(&journal->j_state_lock);

    } else {

      spin_unlock(&journal->j_list_lock);

    }

    mutex_unlock(&journal->j_checkpoint_mutex);

  }

}

```

而在linux主线中，有一个14年的补丁其实是去掉了\_\_process_buffer中复杂的持锁流程的。

```c

[root@custom-16-126 jbd2]# git log -p be1158cc615fd723552f0d9912087423c7cadda5

commit be1158cc615fd723552f0d9912087423c7cadda5

Author: Theodore Ts'o <tytso@mit.edu>

Date:   Mon Sep 1 21:19:01 2014 -0400

  

    jbd2: fold __process_buffer() into jbd2_log_do_checkpoint()

    __process_buffer() is only called by jbd2_log_do_checkpoint(), and it

    had a very complex locking protocol where it would be called with the

    j_list_lock, and sometimes exit with the lock held (if the return code

    was 0), or release the lock.

    This was confusing both to humans and to smatch (which erronously

    complained that the lock was taken twice).

    Folding __process_buffer() to the caller allows us to simplify the

    control flow, making the resulting function easier to read and reason

    about, and dropping the compiled size of fs/jbd2/checkpoint.c by 150

    bytes (over 4% of the text size).

    Signed-off-by: Theodore Ts'o <tytso@mit.edu>

    Reviewed-by: Jan Kara <jack@suse.cz>

  

```

那是不是说，把\_\_process_buffer重构成上面补丁这样就可以了呢？

其实不是的，重构并没有改变拿锁的逻辑，真正修改的话，需要参照下面

这个补丁。

```c

commit 53cf978457325d8fb2cdecd7981b31a8229e446e

Author: Xiaoguang Wang <xiaoguang.wang@linux.alibaba.com>

Date:   Thu Jan 31 23:42:11 2019 -0500

  

    jbd2: fix deadlock while checkpoint thread waits commit thread to finish

    This issue was found when I tried to put checkpoint work in a separate thread,

    the deadlock below happened:

             Thread1                                |   Thread2

    __jbd2_log_wait_for_space                       |

    jbd2_log_do_checkpoint (hold j_checkpoint_mutex)|

      if (jh->b_transaction != NULL)                |

        ...                                         |

        jbd2_log_start_commit(journal, tid);        |jbd2_update_log_tail

                                                    |  will lock j_checkpoint_mutex,

                                                    |  but will be blocked here.

                                                    |

        jbd2_log_wait_commit(journal, tid);         |

        wait_event(journal->j_wait_done_commit,     |

         !tid_gt(tid, journal->j_commit_sequence)); |

         ...                                        |wake_up(j_wait_done_commit)

      }                                             |

    then deadlock occurs, Thread1 will never be waken up.

    To fix this issue, drop j_checkpoint_mutex in jbd2_log_do_checkpoint()

    when we are going to wait for transaction commit.

  

```

回到上面遗留的小问题，为什么jbd2/dm-12-8:547025的最后调度时间比 13027 要晚，

但是它却先报出hung呢？这个需要看 khungtaskd的检测机制有关，可以查看

check_hung_uninterruptible_tasks的源码，具体跟task的串接位置相关，

同时跟当时处于UN状态的进程数也有些关系。

#### 四、故障规避或解决

我们的解决方案是：

1.在红帽的当前3.10.0-957.27.2内核中合入对应的补丁，或者偷懒直接升级到kernel-3.10.0-1127.el7及以上。

[![](http://www.wowotech.net/content/uploadfile/201605/ef3e1463542768.png)](http://www.wowotech.net/support_us.html)

« [关于numa loadbance的死锁分析](http://www.wowotech.net/linux_kenrel/482.html) | [一例hardened_usercopy的bug解析](http://www.wowotech.net/linux_kenrel/480.html)»

**评论：**

**石榴**\
2021-05-18 10:30

终于更新了

[回复](http://www.wowotech.net/linux_kenrel/481.html#comment-8238)

**发表评论：**

昵称

邮件地址 (选填)

个人主页 (选填)

![](http://www.wowotech.net/include/lib/checkcode.php)

- ### 站内搜索

  蜗窝站内  互联网

- ### 功能

  [留言板\
  ](http://www.wowotech.net/message_board.html)[评论列表\
  ](http://www.wowotech.net/?plugin=commentlist)[支持者列表\
  ](http://www.wowotech.net/support_list)

- ### 最新评论

  - ja\
    [@dream：我看完這段也有相同的想法，引用 @dream ...](http://www.wowotech.net/kernel_synchronization/spinlock.html#8922)
  - 元神高手\
    [围观首席power managerment专家](http://www.wowotech.net/pm_subsystem/device_driver_pm.html#8921)
  - 十七\
    [内核空间的映射在系统启动时就已经设定好，并且在所有进程的页表...](http://www.wowotech.net/process_management/context-switch-arch.html#8920)
  - lw\
    [sparse模型和disconti模型没看出来有什么本质区别...](http://www.wowotech.net/memory_management/memory_model.html#8919)
  - 肥饶\
    [一个没设置好就出错](http://www.wowotech.net/linux_kenrel/516.html#8918)
  - orange\
    [点赞点赞，对linuxer的文章总结到位](http://www.wowotech.net/device_model/dt-code-file-struct-parse.html#8917)

- ### 文章分类

  - [Linux内核分析(25)](http://www.wowotech.net/sort/linux_kenrel) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=4)
    - [统一设备模型(15)](http://www.wowotech.net/sort/device_model) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=12)
    - [电源管理子系统(43)](http://www.wowotech.net/sort/pm_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=13)
    - [中断子系统(15)](http://www.wowotech.net/sort/irq_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=14)
    - [进程管理(31)](http://www.wowotech.net/sort/process_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=15)
    - [内核同步机制(26)](http://www.wowotech.net/sort/kernel_synchronization) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=16)
    - [GPIO子系统(5)](http://www.wowotech.net/sort/gpio_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=17)
    - [时间子系统(14)](http://www.wowotech.net/sort/timer_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=18)
    - [通信类协议(7)](http://www.wowotech.net/sort/comm) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=20)
    - [内存管理(31)](http://www.wowotech.net/sort/memory_management) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=21)
    - [图形子系统(2)](http://www.wowotech.net/sort/graphic_subsystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=23)
    - [文件系统(5)](http://www.wowotech.net/sort/filesystem) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=26)
    - [TTY子系统(6)](http://www.wowotech.net/sort/tty_framework) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=27)
  - [u-boot分析(3)](http://www.wowotech.net/sort/u-boot) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=25)
  - [Linux应用技巧(13)](http://www.wowotech.net/sort/linux_application) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=3)
  - [软件开发(6)](http://www.wowotech.net/sort/soft) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=1)
  - [基础技术(13)](http://www.wowotech.net/sort/basic_tech) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=6)
    - [蓝牙(16)](http://www.wowotech.net/sort/bluetooth) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=10)
    - [ARMv8A Arch(15)](http://www.wowotech.net/sort/armv8a_arch) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=19)
    - [显示(3)](http://www.wowotech.net/sort/display) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=22)
    - [USB(1)](http://www.wowotech.net/sort/usb) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=28)
  - [基础学科(10)](http://www.wowotech.net/sort/basic_subject) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=7)
  - [技术漫谈(12)](http://www.wowotech.net/sort/tech_discuss) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=8)
  - [项目专区(0)](http://www.wowotech.net/sort/project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=9)
    - [X Project(28)](http://www.wowotech.net/sort/x_project) [![订阅该分类](http://www.wowotech.net/content/templates/default/images/rss.png)](http://www.wowotech.net/rss.php?sort=24)

- ### 随机文章

  - [Meltdown论文翻译](http://www.wowotech.net/basic_subject/meltdown.html)
  - [X-008-UBOOT-支持命令行(Bubblegum-96平台)](http://www.wowotech.net/x_project/bubblegum_uboot_cmdline.html)
  - [教育漫谈(1)\_《乌合之众》摘录](http://www.wowotech.net/tech_discuss/jymt1.html)
  - [一例centos7.6内核hardlock的解析](http://www.wowotech.net/linux_kenrel/479.html)
  - [基本电路概念之（二）：什么是电容？](http://www.wowotech.net/basic_subject/what-is-capacitor.html)

- ### 文章存档

  - [2024年2月(1)](http://www.wowotech.net/record/202402)
  - [2023年5月(1)](http://www.wowotech.net/record/202305)
  - [2022年10月(1)](http://www.wowotech.net/record/202210)
  - [2022年8月(1)](http://www.wowotech.net/record/202208)
  - [2022年6月(1)](http://www.wowotech.net/record/202206)
  - [2022年5月(1)](http://www.wowotech.net/record/202205)
  - [2022年4月(2)](http://www.wowotech.net/record/202204)
  - [2022年2月(2)](http://www.wowotech.net/record/202202)
  - [2021年12月(1)](http://www.wowotech.net/record/202112)
  - [2021年11月(5)](http://www.wowotech.net/record/202111)
  - [2021年7月(1)](http://www.wowotech.net/record/202107)
  - [2021年6月(1)](http://www.wowotech.net/record/202106)
  - [2021年5月(3)](http://www.wowotech.net/record/202105)
  - [2020年3月(3)](http://www.wowotech.net/record/202003)
  - [2020年2月(2)](http://www.wowotech.net/record/202002)
  - [2020年1月(3)](http://www.wowotech.net/record/202001)
  - [2019年12月(3)](http://www.wowotech.net/record/201912)
  - [2019年5月(4)](http://www.wowotech.net/record/201905)
  - [2019年3月(1)](http://www.wowotech.net/record/201903)
  - [2019年1月(3)](http://www.wowotech.net/record/201901)
  - [2018年12月(2)](http://www.wowotech.net/record/201812)
  - [2018年11月(1)](http://www.wowotech.net/record/201811)
  - [2018年10月(2)](http://www.wowotech.net/record/201810)
  - [2018年8月(1)](http://www.wowotech.net/record/201808)
  - [2018年6月(1)](http://www.wowotech.net/record/201806)
  - [2018年5月(1)](http://www.wowotech.net/record/201805)
  - [2018年4月(7)](http://www.wowotech.net/record/201804)
  - [2018年2月(4)](http://www.wowotech.net/record/201802)
  - [2018年1月(5)](http://www.wowotech.net/record/201801)
  - [2017年12月(2)](http://www.wowotech.net/record/201712)
  - [2017年11月(2)](http://www.wowotech.net/record/201711)
  - [2017年10月(1)](http://www.wowotech.net/record/201710)
  - [2017年9月(5)](http://www.wowotech.net/record/201709)
  - [2017年8月(4)](http://www.wowotech.net/record/201708)
  - [2017年7月(4)](http://www.wowotech.net/record/201707)
  - [2017年6月(3)](http://www.wowotech.net/record/201706)
  - [2017年5月(3)](http://www.wowotech.net/record/201705)
  - [2017年4月(1)](http://www.wowotech.net/record/201704)
  - [2017年3月(8)](http://www.wowotech.net/record/201703)
  - [2017年2月(6)](http://www.wowotech.net/record/201702)
  - [2017年1月(5)](http://www.wowotech.net/record/201701)
  - [2016年12月(6)](http://www.wowotech.net/record/201612)
  - [2016年11月(11)](http://www.wowotech.net/record/201611)
  - [2016年10月(9)](http://www.wowotech.net/record/201610)
  - [2016年9月(6)](http://www.wowotech.net/record/201609)
  - [2016年8月(9)](http://www.wowotech.net/record/201608)
  - [2016年7月(5)](http://www.wowotech.net/record/201607)
  - [2016年6月(8)](http://www.wowotech.net/record/201606)
  - [2016年5月(8)](http://www.wowotech.net/record/201605)
  - [2016年4月(7)](http://www.wowotech.net/record/201604)
  - [2016年3月(5)](http://www.wowotech.net/record/201603)
  - [2016年2月(5)](http://www.wowotech.net/record/201602)
  - [2016年1月(6)](http://www.wowotech.net/record/201601)
  - [2015年12月(6)](http://www.wowotech.net/record/201512)
  - [2015年11月(9)](http://www.wowotech.net/record/201511)
  - [2015年10月(9)](http://www.wowotech.net/record/201510)
  - [2015年9月(4)](http://www.wowotech.net/record/201509)
  - [2015年8月(3)](http://www.wowotech.net/record/201508)
  - [2015年7月(7)](http://www.wowotech.net/record/201507)
  - [2015年6月(3)](http://www.wowotech.net/record/201506)
  - [2015年5月(6)](http://www.wowotech.net/record/201505)
  - [2015年4月(9)](http://www.wowotech.net/record/201504)
  - [2015年3月(9)](http://www.wowotech.net/record/201503)
  - [2015年2月(6)](http://www.wowotech.net/record/201502)
  - [2015年1月(6)](http://www.wowotech.net/record/201501)
  - [2014年12月(17)](http://www.wowotech.net/record/201412)
  - [2014年11月(8)](http://www.wowotech.net/record/201411)
  - [2014年10月(9)](http://www.wowotech.net/record/201410)
  - [2014年9月(7)](http://www.wowotech.net/record/201409)
  - [2014年8月(12)](http://www.wowotech.net/record/201408)
  - [2014年7月(6)](http://www.wowotech.net/record/201407)
  - [2014年6月(6)](http://www.wowotech.net/record/201406)
  - [2014年5月(9)](http://www.wowotech.net/record/201405)
  - [2014年4月(9)](http://www.wowotech.net/record/201404)
  - [2014年3月(7)](http://www.wowotech.net/record/201403)
  - [2014年2月(3)](http://www.wowotech.net/record/201402)
  - [2014年1月(4)](http://www.wowotech.net/record/201401)

[![订阅Rss](http://www.wowotech.net/content/templates/default/images/rss.gif)](http://www.wowotech.net/rss.php "RSS订阅")

Copyright @ 2013-2015 [蜗窝科技](http://www.wowotech.net/ "wowotech") All rights reserved. Powered by [emlog](http://www.emlog.net/ "采用emlog系统")
