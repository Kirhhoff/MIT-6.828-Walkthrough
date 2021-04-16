# Homework 7: xv6 log
这里首先是让我们体验一下log错误实现带来的错误，然后对log系统做一点小优化。

我们首先来演示一下这里制造的错误，修改commit和recover_from_log后，得到错误如下：
![]()
同时，我们重启xv6，查看a，得到panic：
![]()

这里也比较清晰，因为创建文件时，第一个被写的block是inode所在的block（只creat文件不会分配磁盘block），因此这里我们把第一个log的block号设为0后，就不会更新inode的实际数据，这样一来，我们cat显示的时候，就会查看一个type为0的inode

接下来我们对log系统做一些优化，主要是避免不必要的读取。在这里可以避免的读取为commit调用的install_trans中从log块到目标块的拷贝工作，因为当从commit中调用时，install_trans的目标块内容本身一定在cache中，无需从log块中拷贝，直接写入即可。因此这里我们修改一下对install_trans的定义，追加一个参数来表示是否是从commit中调用

原函数
```
// Copy committed blocks from log to their home location
static void
install_trans(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *lbuf = bread(log.dev, log.start+tail+1); // read log block
    struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst
    memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
    bwrite(dbuf);  // write dst to disk
    brelse(lbuf);
    brelse(dbuf);
  }
}
```
修改后
```
// Copy committed blocks from log to their home location
static void
install_trans(int from_commit)
{
  int tail;

  if(from_commit)
    for (tail = 0; tail < log.lh.n; tail++) {
      struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst
      bwrite(dbuf);  // write dst to disk
      brelse(dbuf);
    }
  else
    for (tail = 0; tail < log.lh.n; tail++) {
      struct buf *lbuf = bread(log.dev, log.start+tail+1); // read log block
      struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst
      memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
      bwrite(dbuf);  // write dst to disk
      brelse(lbuf);
      brelse(dbuf);
    }
}
```
我们简单测试一下：
![]()
没啥问题