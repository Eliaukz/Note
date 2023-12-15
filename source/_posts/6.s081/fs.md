---
title: xv6-fs
tags:
  - 6s081
category: 6s081
---
## Large files

在现有的xv6文件系统中，文件的大小被限制在268个块（12个直接索引+1个一级索引）
实验要求我们实现二级索引，这样文件的大小就会被设置为65803个块（11个直接索引+1个一级索引+1个二级索引）

首先先修改fs.h中的文件设置信息，将直接索引修改为11，将文件大小设置为65803，同时修改dinode结构体

```c
//kernel/fs.h
#define NDIRECT 11 // 12 ->  11
#define NINDIRECT (BSIZE / sizeof(uint))   //256
#define MAXFILE (NDIRECT + NINDIRECT + NINDIRECT * NINDIRECT)    //11+256+256*56  直接索引+一级索引+二级索引

// On-disk inode structure
struct dinode {
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1+1];   // Data block addresses
};
```


然后修改bmap函数实现映射即可，二级索引的映射方式可参照一级索引的映射方式进行修改

```c
//kernel/fs.c
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){  // 直接索引
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT) {  // 一级索引
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  bn -= NINDIRECT;

  if(bn < NINDIRECT * NINDIRECT) { // 二级索引
    // Load indirect block, allocating if necessary.
    if ((addr = ip->addrs[NDIRECT + 1]) == 0)
      ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if ((addr = a[bn / NINDIRECT]) == 0) {
      a[bn / NINDIRECT] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    bn %= NINDIRECT;
    bp = bread(ip->dev, addr);
    a = (uint *)bp->data;
    if ((addr = a[bn]) == 0) {
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

## Symbolic links

这里需要实现软链接--符号连接，首先还是添加系统调用，不再赘述。

首先在sata.h中添加一个新的文件类型

```c
//kernel/sata.f
#define T_DIR     1   // Directory
#define T_FILE    2   // File
#define T_DEVICE  3   // Device
#define T_SYMLINK 4   // 软符号链接

```

在kernel/fcntl.h中添加一个打开文件的标记。

```c
//kernel/fcntl.h
#define O_RDONLY  0x000
#define O_WRONLY  0x001
#define O_RDWR    0x002
#define O_CREATE  0x200
#define O_TRUNC   0x400
#define O_NOFOLLOW 0x800

```

然后就可以在sysfile中实现sys_symlink函数

```c
//kernel/sysfile.c

uint64
sys_symlink(void)
{
  struct inode *ip;
  char target[MAXPATH], path[MAXPATH];
  if (argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
    return -1;

  begin_op();

  ip = create(path, T_SYMLINK, 0, 0);
  if (ip == 0) {
    end_op();
    return -1;
  }

  // use the first data block to store target path.
  if(writei(ip, 0, (uint64)target, 0, strlen(target)) < 0) {
    end_op();
    return -1;
  }

  iunlockput(ip);

  end_op();
  return 0;
}
```


修改open系统调用在遇到符号连接时应该递归的继续寻找下去。

```c
//kernel/sysfile.c

uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  if((n = argstr(0, path, MAXPATH)) < 0 || argint(1, &omode) < 0)
    return -1;

  begin_op();

  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    int symlink_depth = 0;
    while (1) {  // recursively follow symlinks
      if((ip = namei(path)) == 0){
          end_op();
          return -1;
      }
      ilock(ip);
      if(ip->type == T_SYMLINK && (omode & O_NOFOLLOW) == 0) {
        if(++symlink_depth > 10) {
          // too many layer of symlinks, might be a loop
          iunlockput(ip);
          end_op();
          return -1;
        }
        if(readi(ip, 0, (uint64)path, 0, MAXPATH) < 0) {
          iunlockput(ip);
          end_op();
          return -1;
        }
        iunlockput(ip);
      } else {
        break;
      }
    }
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
  }
	///...
}

```







