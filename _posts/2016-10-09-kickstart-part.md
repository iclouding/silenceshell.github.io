---
layout: post
title: "kickstart从autopart改为自行分区"
date: 2016-10-09 00:11:12
author: 伊布
categories: tech
tags: kickstart
cover:  "/assets/instacode.png"
---

**背景**

在前面[如何为linux集群批量装机？](http://www.datastart.cn/tech/2016/08/20/kickstart.html)这篇文章里介绍了我们的服务器装机流程，其中ks文件中磁盘分区的配置是这样的:

```
# Allow anaconda to partition the system as needed
clearpart --all --initlabel
autopart
ignoredisk --only-use=sda
```

使用autopart的原因是我们的磁盘容量不一定（可能是400G的ssd，也可能是4T/6T的sata），通过autopart交给anaconda去决定分区是最省事的。autopart默认会为根分区分配50GiB，为boot分区分配约512MiB，剩下的全部分配给home分区。

但实际使用时，发现很多组件的默认日志路径是/var/log/xxx，实际是占用跟分区的空间的，而有的开源组件的日志大小/个数默认无上限，在系统震荡时很容易会将根分区写满（即使不震荡，天长日久也会慢慢写满），而根分区写满是很可怕的事情，会导致很多问题。

不过这个问题可以通过修改日志路径到home分区的方式来解决，真正导致下决心不用autopart的稻草是其默认文件系统为xfs（从centos7开始，Redhat将默认的文件系统改为了xfs）。

我们在客户处有一个spark集群，国庆节前反馈SQL执行速度极慢，但从cpu/内存/io/网络来看，使用率都很低，但是top显示系统负载有800多(一般闲置状态0.0x)。但查看系统进程时发现有大量crond发起的重复进程，其中某些进程的状态为D（uninterrupted），推测可能是这些进程影响了系统调度，spark任务的进程得不到时间片，因此执行比较慢。

既然不是资源不足，那么只要将crond的进程关闭即可，但crond发起的子进程状态为D，无法杀死(-9, -15都是向linux进程发送SIGTERM，进程不响应信号自然无法杀死)。从问题出现的时间来判断，可能跟当时根分区被日志写满有关系，查看系统日志/var/log/messages，有这么一段：

```
Sep 29 19:35:36 node0 kernel: INFO: task kworker/u66:1:13269 blocked for more than 120 seconds.
Sep 29 19:35:36 node0 kernel: "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
Sep 29 19:35:36 node0 kernel: kworker/u66:1   D 0000000000000000     0 13269      2 0x00000080
Sep 29 19:35:36 node0 kernel: Workqueue: writeback bdi_writeback_workfn (flush-253:4)
Sep 29 19:35:36 node0 kernel: ffff881f023737e8 0000000000000046 ffff881cd2f04500 ffff881f02373fd8
Sep 29 19:35:36 node0 kernel: ffff881f02373fd8 ffff881f02373fd8 ffff881cd2f04500 ffff882028989c00
Sep 29 19:35:36 node0 kernel: ffff88165098b088 ffff882028989dc0 00000000000209cc 0000000000000000
Sep 29 19:35:36 node0 kernel: Call Trace:
Sep 29 19:35:36 node0 kernel: [<ffffffff8163a909>] schedule+0x29/0x70
Sep 29 19:35:36 node0 kernel: [<ffffffffa02c566d>] xlog_grant_head_wait+0x9d/0x180 [xfs]
Sep 29 19:35:36 node0 kernel: [<ffffffffa02c57ee>] xlog_grant_head_check+0x9e/0x110 [xfs]
Sep 29 19:35:36 node0 kernel: [<ffffffffa02c9182>] xfs_log_reserve+0xc2/0x1b0 [xfs]
Sep 29 19:35:36 node0 kernel: [<ffffffffa02c3ad5>] xfs_trans_reserve+0x1b5/0x1f0 [xfs]
Sep 29 19:35:36 node0 kernel: [<ffffffffa02a00c0>] xfs_setfilesize_trans_alloc.isra.10+0x40/0xa0 [xfs]
Sep 29 19:35:36 node0 kernel: [<ffffffffa02a10e1>] xfs_vm_writepage+0x291/0x5d0 [xfs]
Sep 29 19:35:36 node0 kernel: [<ffffffff81173a63>] __writepage+0x13/0x50
Sep 29 19:35:36 node0 kernel: [<ffffffff81174581>] write_cache_pages+0x251/0x4d0
Sep 29 19:35:36 node0 kernel: [<ffffffff81173a50>] ? global_dirtyable_memory+0x70/0x70
Sep 29 19:35:36 node0 kernel: [<ffffffff812c73e3>] ? __blk_run_queue+0x33/0x40
Sep 29 19:35:36 node0 kernel: [<ffffffff812c74a7>] ? queue_unplugged+0x37/0xa0
Sep 29 19:35:36 node0 kernel: [<ffffffff8117484d>] generic_writepages+0x4d/0x80
Sep 29 19:35:36 node0 kernel: [<ffffffffa02a0993>] xfs_vm_writepages+0x43/0x50 [xfs]
Sep 29 19:35:36 node0 kernel: [<ffffffff811758fe>] do_writepages+0x1e/0x40
Sep 29 19:35:36 node0 kernel: [<ffffffff81208410>] __writeback_single_inode+0x40/0x220
Sep 29 19:35:36 node0 kernel: [<ffffffff81208e7e>] writeback_sb_inodes+0x25e/0x420
Sep 29 19:35:36 node0 kernel: [<ffffffff812090df>] __writeback_inodes_wb+0x9f/0xd0
Sep 29 19:35:36 node0 kernel: [<ffffffff81209923>] wb_writeback+0x263/0x2f0
Sep 29 19:35:36 node0 kernel: [<ffffffff811f875c>] ? get_nr_inodes+0x4c/0x70
Sep 29 19:35:36 node0 kernel: [<ffffffff8120bbab>] bdi_writeback_workfn+0x2cb/0x460
Sep 29 19:35:36 node0 kernel: [<ffffffff8109d5fb>] process_one_work+0x17b/0x470
Sep 29 19:35:36 node0 kernel: [<ffffffff8109e3cb>] worker_thread+0x11b/0x400
Sep 29 19:35:36 node0 kernel: [<ffffffff8109e2b0>] ? rescuer_thread+0x400/0x400
Sep 29 19:35:36 node0 kernel: [<ffffffff810a5aef>] kthread+0xcf/0xe0
Sep 29 19:35:36 node0 kernel: [<ffffffff810a5a20>] ? kthread_create_on_node+0x140/0x140
Sep 29 19:35:36 node0 kernel: [<ffffffff81645858>] ret_from_fork+0x58/0x90
Sep 29 19:35:36 node0 kernel: [<ffffffff810a5a20>] ? kthread_create_on_node+0x140/0x140
```

而在此之前，有大量的XFS异步写入失败的问题。

```
Sep 29 19:25:24 node0 kernel: XFS (dm-4): metadata I/O error: block 0x3c6478 ("xfs_buf_iodone_callbacks") error 5 numblks 8
Sep 29 19:25:24 node0 kernel: loop: Write error at byte offset 3744792576, length 4096.
Sep 29 19:25:24 node0 kernel: XFS (dm-4): Failing async write on buffer block 0x3c6478. Retrying async write.
Sep 29 19:25:24 node0 kernel: loop: Write error at byte offset 3744858112, length 4096.
Sep 29 19:25:24 node0 kernel: loop: Write error at byte offset 3744923648, length 4096.
Sep 29 19:25:24 node0 kernel: XFS (dm-4): Failing async write on buffer block 0x3c6478. Retrying async write.
Sep 29 19:25:24 node0 kernel: loop: Write error at byte offset 3744989184, length 4096.
Sep 29 19:25:25 node0 kernel: loop: Write error at byte offset 3745054720, length 4096.
```

原因可能是因为根分区没有写入空间，导致docker里的crond任务（一分钟一次）全部挂起，产生了大量D状态的进程。但是当删除日志释放空间之后，D状态进程并没有恢复。用内核的堆栈去google找到了这么个问题：[direct-lvm with xfs causes Docker to hang when disk is full](https://github.com/docker/docker/issues/20707)，磁盘满的时候，docker会挂住，推测可能docker容器中启动的进程也无法恢复（猜的，请指正）。在这个问题中有答主表示切回了ext4（xfs有一个优势是格式化更快，但他们的rootfs不大，只有10GiB），由于我们的根分区只是用来跑操作系统和进程，所以使用裸的ext4相对来说更直接和稳妥一些。

另外从[这篇评测](http://www.cnblogs.com/tommyli/p/3201047.html)来看，xfs的性能并不比ext4更好。

**解决方法**

最简单的办法是仍然autopart，但lvm的文件系统由xfs改为ext4即可。查阅[Redhat的手册](https://access.redhat.com/documentation/zh-CN/Red_Hat_Enterprise_Linux/7/html/Installation_Guide/sect-kickstart-syntax.html)，autopart并不支持修改文件系统；从anaconda的代码来看，应该是实现的人偷懒了：

```python
def setDefaultPartitioning(self, storage):
    autorequests = [PartSpec(mountpoint="/", fstype=storage.default_fstype,
                             size=Size("1GiB"),
                             max_size=Size("50GiB"),
                             grow=True,
                             btr=True, lv=True, thin=True, encrypted=True),
                    PartSpec(mountpoint="/home",
                             fstype=storage.default_fstype,
                             size=Size("500MiB"), grow=True,
                             required_space=Size("50GiB"),
                             btr=True, lv=True, thin=True, encrypted=True)]

    bootreqs = platform.set_default_partitioning()
    if bootreqs:
        autorequests.extend(bootreqs)


    disk_space = getAvailableDiskSpace(storage)
    swp = swap_suggestion(disk_space=disk_space)
    autorequests.append(PartSpec(fstype="swap", size=swp, grow=False,
                                 lv=True, encrypted=True))

    for autoreq in autorequests:
        if autoreq.fstype is None:
            if autoreq.mountpoint == "/boot":
                autoreq.fstype = storage.default_boot_fstype
            else:
                autoreq.fstype = storage.default_fstype

    storage.autopart_requests = autorequests
```
其中storage.default_fstype是从xml配置文件中读取的，不能通过参数来修改，只能改xml配置文件后重新打包anaconda的rpm包，但这样维护性就比较差了。

autopart不行，只能手工part了。

**手工part**

下面直接贴：

```bash
part biosboot --fstype=biosboot --size=1
part /boot --fstype ext4 --size=500
part swap --size=4096
part / --fstype ext4 --size=102400
part /home --fstype ext4 --size=51200 --grow
```

比较简单，着重提两点：

1. 参数grow：让逻辑卷使用所有可用空间（若有），或使用设置的最大值。所以，home使用了所有剩余的空间，而root定死为50GiB。
2. biosboot分区：由于我们的sda盘有6TiB，所以只能使用GPT分区（MBR最大支持容量2.2TiB，分区数3P+1E，而GPT最大支持18EB），而GPT分区要求有biosboot分区[扩展阅读](https://bugzilla.redhat.com/show_bug.cgi?id=1032428)。
