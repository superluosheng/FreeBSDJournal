# ZFS 简介

ZFS 将卷管理器和独立的文件系统的角色合二为一，与独立的文件系统相比具有多种优势。它以速度、灵活性而闻名，最重要的是，它非常注意防止数据丢失。许多传统的文件系统不得不一次只存在于一个磁盘上，而 ZFS 知道磁盘的底层结构，并创建一个可用的存储池，甚至可用于多个磁盘。当额外的磁盘被添加到池中时，现有的文件系统将自动增长，立即成为文件系统的可用部分。

## 开始使用

FreeBSD 可以在系统初始化时挂载 ZFS 池和数据集。要启用它，请在 /etc/rc.conf 中添加这一行：
```
zfs_enable=”YES”
```

然后启动服务：

```
service  zfs  start
```

## 识别硬件

在设置 ZFS 之前，请识别与系统相关的磁盘的设备名称。一种快速方法是:
```
#  egrep   da[0-9] |cd[0-9]    /var/run/dmesg.boot
```

这个输出应识别出设备名称，在本指南的其余部分中的例子将使用默认的 SCSI 名称：da0、da1 和 da2。如果硬件不同，请确保使用正确的设备名称来代替。

## 创建一个单磁盘池

使用单个磁盘设备创建一个简单的、非冗余的池：
```
#  zpool  create  example  /dev/da0
```

创建文件供用户在池内浏览：
```
#  cd  /example
#  ls
#  touch  testfile
```

可以用 ls 查看新文件：
```
#  ls  -al
```

我们已经可以开始使用更高级的 ZFS 功能和属性。在池中创建一个启用了压缩功能的数据集：
```
#  zfs  create  example/compressed
#  zfs  set  compression=on  example/compressed
```

example/compressed 数据集现在是一个 ZFS 压缩文件系统。

用以下方法禁用压缩：
```
#  zfs  set  compression=off  example/compressed
```

要卸载文件系统，使用 zfs umount，然后用 df 验证：
```
#  zfs  umount  example/compressed
#  df
```

验证 example/compressed 是否作为挂载文件包含在输出下。可以用 zfs 重新挂载该文件系统：
```
#  zfs  mount  example/compressed
#  df
```

在文件系统被挂载的情况下，其输出应该包括与下面类似的一行：

```
example/compressed  17547008  0  17547008  0%  /example/compressed
```

ZFS 数据集的创建就像其他文件系统一样，下面的例子创建了一个名为 data 的新文件系统：
```
#  zfs  create  example/data
```

使用 df 查看数据和空间使用情况（为了清晰起见，已经删除了部分输出）：
```
#  df

example/    compressed  17547008        0   17547008        0%  /example/compressed
example/    data        17547008        0   17547008        %   /example/data
```

因为这些文件系统是建立在 ZFS 上的，它们从同一个存储池中提取。这消除了对其他文件系统所依赖的卷和分区的需要。

要销毁文件系统，然后销毁不再需要的池：
```
#  zfs  destroy  example/compressed
#  zfs  destroy  example/data
#  zpool  destroy  example
```

## RAID-Z
RAID-Z 池需要三个或更多的磁盘，但在一个磁盘发生故障时提供数据丢失的保护。因为 ZFS 池可以使用多个磁盘，对 RAID 的支持是文件系统设计中固有的。

要创建一个 RAID-Z 池，指定要添加到池中的磁盘：
```
#  zpool  create  storage  raidz  da0  da1  da2
```

在创建了 zpool 之后，可以在该池中创建一个新的文件系统：
```
#  zfs  create  storage/home
```

启用压缩功能，并存储一个额外的目录和文件副本：
```
#  zfs  set	copies=2  storage/home
#  zfs set compression=gzip  storage/home
```

RAID-Z 池是一个存储关键系统文件的好地方，例如用户的主目录：
```
#  cp  -rp  /home/*  /storage/home
#  rm  -rf  /home  /usr/home
#  ln  -s  /storage/home  /home
#  ln  -s  /storage/home  /usr/home
```

可以创建文件系统快照，以便以后回滚，快照名称用黄色标记，可以是你想要的任何名称：
```
#  zfs  snapshot  storage/home@ 11-01-22
```

ZFS 创建数据集的快照，允许用户备份文件系统，以便将来进行回滚或数据恢复：
```
#  zfs  rollback  storage/home@ 11-01-22
```

要列出所有可用的快照，可以使用 zfs list：
```
#  zfs  list  -t  snapshot  storage/home
```

## 恢复RAID-Z
每个软件 RAID 都有一个监控其状态的方法。使用以下方法查看 RAID-Z 的状态：

```
#  zpool  status  -x
```

如果所有池子都是在线的，而且一切正常，则显示这个消息：

```
all  pools  are  healthy
```

如果有问题产生，可能是某个磁盘处于离线状态，池的状态会看起来像这样：
```
pool:  storage
state:  DEGRADED
status:  One  or  more  devices  has  been  taken  offline  by  the  administrator .      Sufficient  replicas  exist  for  the  pool  to  continue  functioning  in  a degraded  state .
action:  Online  the  device  using   ‘zpool  online’  or  replace  the  device  with 'zpool  replace' .
scrub:  none  requested
config:

     NAME      STATE     READ WRITE     CHKSUM
     storage   DEGRADED  0    0         0
     raidz1    DEGRADED  0    0         0
          da0  ONLINE    0    0         0
          da1  OFFLINE    0    0         0
          da2  ONLINE    0    0         0


errors:  No  known  data  errors
```

“OFFLINE” 显示，管理员可使用以下方式使 da1 脱机：
```
#  zpool  offline  storage  da1
```

立即关闭计算机电源并更换 da1：打开计算机电源并将 da1 返回到池中：
```
#  zpool  replace  storage  da1
```

接下来，再次检查状态，这次不使用 -x 来显示所有池：
```
#  zpool  status  storage
pool:  storage
state:  ONLINE
scrub:  resilver  completed  with  0  errors  on  Fri  Nov  4  11:12:03  2022 config:

     NAME      STATE     READ WIRTE     CKSUM
     storage   ONLINE    0    0         0
     raidz1    ONLINE    0    0         0
     da0       ONLINE    0    0         0
     da1       ONLINE    0    0         0
     da2       ONLINE    0    0         0


errors:  No  known  data  errors
```

### 数据验证
ZFS 使用校验和来验证存储数据的完整性，可以验证这些数据校验和(称为清洗)以确保存储池的完整性。
```
#  zpool  scrub  storage
```

由于大量的输入/输出要求，一次只能运行一个 scrub。清洗的时间取决于池中存储的数据量。清洗完成后，使用 zpool status 查看状态：
```
#  zpool  status  storage
pool:  storage
state:  ONLINE
scrub:  scrub  completed  with  0  errors  on  Fri  Nov  4  11:19:52  2022
config:
     NAME      STATE     READ WRITE     CKSUM
     storage   ONLINE    0    0         0
     raidz1    ONLINE    0    0         0
     da0       ONLINE    0    0         0
     da1       ONLINE    0    0         0
     da2       ONLINE    0    0         0


errors:  No  known  data  errors
```
显示上次清理的完成日期有助于决定何时开始另一次清理。常规的 scrub 有助于保护数据免受静默式损坏，并确保池的完整性。

## ZFS 管理
ZFS 有两个主要的管理工具：zpool 工具控制池的操作，并允许添加、删除、替换和管理磁盘。zfs 工具允许创建、销毁和管理数据集，包括文件系统和卷。
虽然本入门指南不涉及 ZFS 管理，但您可以参考 zfs（8）和 zpool（8）了解其他 ZFS 选项。

## 其他资源
虽然使用本指南创建的非冗余池和 RAID-Z 池在大多数用例中都可以工作，但更复杂或更专业的系统可能需要进一步的 ZFS 管理和设置。ZFS 是一个功能强大且可自定义的文件系统，本指南几乎没有提及使用它可以做什么。OpenZFS wiki 有大量关于安装、ZFS 系统管理和手册页面的文档。如果由于系统体系结构的原因需要进行调优，那么可以在 OpenZFS 和 FreeBSD 的 wiki 页面上找到 ZFS 调优指南。

---
**DREW GURKOWSKI**：来自FreeBSD 基金会
