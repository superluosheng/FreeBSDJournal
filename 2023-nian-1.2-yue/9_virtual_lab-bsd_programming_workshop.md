# 虚拟实验室 BSD 编程研讨会

我们的虚拟实验室将由一个 FreeBSD 主机系统组成，它使用 FreeBSD Jail 技术，为我们要安装在实验室中的每个系统提供独立的环境来运行服务并履行其职责。这些职责可以是任何数量的事情，比如提供网页服务、存储和检索数据库记录、查询和应答 DNS 请求、缓存系统更新文件等等。我们的想法是打一个坚实的基础，使我们的虚拟实验室能够在未来扩展。鉴于 FreeBSD Jail 的本质是提供一个轻量的系统来容纳我们的服务，我们可以明确地知道，我们可能会发现自己身处于任何数量的兔子洞中（译者注：兔子洞即未知、不确定的事物，出自爱丽丝梦游仙境），不会因为资源达到限制而限制我们的创造力或探索。我们可以为每个想法提供一个独立的环境，而不必担心为支持该想法的发展所付出的服务而配置另一个操作系统产生的费用。我在工作中经历了许多不同的操作系统安装托管方法，在配置新的 jail 时，我感到惊喜。它是如此的环保! 我不再为财务问题和这笔费用将持续多久的问题而烦恼，相反，这只是我不断发展的实验室中的另一个环境，本质上没有最低的月租。我不想搞得太专业化，而在电力、网络链接和机器硬件成本方面陷入财务困境；是的，我同意这些东西是有成本的，但是一旦它们就位，增加新的主机并不像最初的实验室设置那样需要考虑和支出。我建议重新利用一台现有的机器作为主机，甚至可以是一台笔记本电脑，因为它有一个内置的电池做电力备份，使你有时间在一直停电的情况下优雅地关闭你的系统。在我们的虚拟实验室中，需要多个物理网络接口来连接物理以太网交换机，你的物理主机使用的以太网电缆和接入点的问题并不是一个实质性的问题。这些接口可以用文本文件中的文字来创建，并且可以通过创建虚拟以太网电缆来连接我们的虚拟网络的各个部分。

## FreeBSD 主机
这个主机需要一些网络接口。我们希望主机能有自己的方式来连接互联网。这可以是一个由本地路由器提供的 DHCP 分配好的地址，或者由你的主机来扮演路由器的角色，通过调制解调器直接连接到外部世界。无论你的主机如何获得互联网连接，我们都希望有一个额外的网络接口，只用于我们的实验室网络。这个接口的名称可能是**em0，igb0** 或类似其他的，这取决于网卡驱动程序和安装的接口数量。即使我们没有多个物理网络接口，我们也可以单独做一个网络接口来使用 **cloned_interfaces**。在我的例子中，我正在重新利用一些笔记本电脑。其中一台有一块可用的无线网卡，我用来连接互联网。这是一台 2010 年代早期的华硕 ROG 笔记本电脑。它有一个内置的以太网接口。在我的 X1 Carbon 第七代 ThinkPad 上，没有内置的以太网接口。我只是使用 USB 接口和我的安卓 USB 热点。然而，我对戴尔 Precision 笔记本电脑最感兴趣。它是一个很棒的实验室主机，因为它配备了英特尔至强处理器，可以容纳 128GB 的内存和多个 NVME 硬盘。它的笔记本后面有一个内置的以太网接口，第二个是 USB-C 加密狗，上面有一个英特尔 **igb0** 以太网接口，我用它作为实验室网络的专用接口。最后，旧的塔式机也可以作为主机使用，有一些神奇的 PCI 网卡，你可以安装多个以太网接口。这里的关键是找到适合你的东西，安装 FreeBSD 并开始使用我们的虚拟实验室。如果你想获得一些安装和配置 FreeBSD 的技巧，请看 2022 年 7/8 月的 FreeBSD 杂志中的 FreeBSD 入门研讨会。

## 虚拟网络设计
就像在物理服务器世界中，我们在路由器/防火墙系统上有多个物理以太网接口，我们可以为我们的实验室路由器/防火墙系统提供多个接口，从现在开始我将其称为网关。这些接口将通过网桥相互连接。想象一下来自物理网络的以太网交换机，网桥提供了类似的功能。而对于我们的以太网电缆，我们将使用 epair(4) 接口。这些接口由两侧组件，A 侧和 B 侧。我们将 A 侧连接到网桥，B 侧连接到我们的 Jail。这样，所有的 jail 都可以通过虚拟网桥相互通信，就像网络中的物理主机通过以太网电缆连接到交换机一样，可以相互通信。网关 jail 将有一条额外的虚拟电缆连接到一个单独的网桥，该桥可以连接到实验室外和互联网。所有 jail 都将其默认路由指向该网关。我们给这个独立的网桥提供连接到外部世界的能力的方法是通过指定一个物理接口作为网桥的成员。同样， 这类似于把以太网电缆插入交换机,但在这种情况下,以太网电缆被插入 FreeBSD 主机系统， 但实际上插入桥接器。通过这样做，同一网桥的其他虚拟以太网电缆将能够与之通信，并使其数据包在实验室环境中流动。

让我们把它分解一下，每个 jail 使用一种叫做 vnet 的技术获得自己的网络。我们将 FreeBSD 主机上的物理接口连接到一个虚拟网桥上，并为我们的实验室网络创建第二个虚拟网桥，该网桥上只有来自实验室 jail 的虚拟接口与之相连。然后我们使用路由和防火墙规则来按我们认为合适的方式推送数据包。

## 防火墙
我们将使用 PF 作为防火墙。FreeBSD Jail 的工作方式是，每个 jail 都共享主机内核，所以如果有一个内核模块你想在 jail 里访问，你只需要加载它，然后在特定 jail 的 **jail.conf** 设置中配置来访问。请查看 **/etc/defaults/devfs.rules** 文件。为了让我们的网关使用 PF 防火墙，我们需要将 jail 的配置设置为使用该文件中列出的 pf 规则集。我们稍后将设置自己的自定义 devfs 规则，并包括 **devfsrules_jail_vnet** 规则中的配置。

## 配置
既然我们已经介绍了我们将要做的事情，让我们开始做吧。我们从一个运行 13.1-RELEASE 的 FreeBSD 主机开始。如上所述，这个主机应该有一个链接的互联网连接。我们将使用这个网络来下载一些文件，以用于创建我们的 jail。该主机正在运行 NTPD，所以它能获得准确的时间。用 **sockstat -46** 检查在主机上监听的任何服务，如果没有使用，就把它关掉。记住，主机应该限制它所做的事情——在我们的实验室里，我们会有很多有趣的事情要做，所以要尽你所能去限制主机上的服务。我打算用附带的键盘和屏幕登录来亲自管理我的主机，所以我没有在主机上启用 SSH。

现在我们已经准备好启用 jail 了。一个简单的 **sysrc jail_enable=YES** 就可以了。不需要安装任何软件包，jail 管理已内置到 FreeBSD 中。查看 **/usr/share/examples/jails** 中的 README 文件，了解一些如何配置 jail 的例子。正如你所看到的，有很多方法可以进行 jail 配置。我已经做了研究，并挑选了你将在这里看到的配置方法。欢迎你尝试一下其他的方法，看看有什么合适的。如果你真的用其他方法来完成这项任务，请考虑把它写下来，这样别人就可以看到你发现的有用方法，并自己尝试一下。在这一点上，我们已经验证了我们的主机已经为托管 jail 做好了准备，并且已经启用了 jail 服务，所以我们可以快速 **reboot**，仔细检查主机上是否有最低限度的服务在监听，然后继续创建我们所有 jail 要使用的基本配置。在编辑配置文件时，我们将使用vim，对于基本的编辑任务，你真的只需要知道一些东西，花几分钟时间通过 **vimtutor** 命令中的交互式练习来了解你的方向，你将在短时间内成为 vim 的新手。

注意：我们将以 root 用户的身份运行以下所有命令。键入 sudo-i 以成为 root 用户。

**编辑 jail.conf**
```
vim /etc/jail.conf
```

**将以下内容放入 /etc/jail.conf**
```
$labdir=”/lab”;
$domain=”lab.bsd.pw”;
path=”$labdir/$name”;
host.hostname=”$name.$domain”;
exec.clean;
exec.start=”sh /etc/rc”;
exec.stop=”sh /etc/rc.shutdown”;
exec.timeout=90;
stop.timeout=30;
mount.devfs;
exec.consolelog=”/var/tmp/${host.hostname}”;
```

### **base.txz**
```
mkdir -p /lab/media/13.1-RELEASE
cd /lab/media/13.1-RELEASE
fetch http://ftp.freebsd.org/pub/FreeBSD/releases/amd64/13.1-RELEASE/base.txz
```

### **Gateway Jail**
```
mkdir /lab/gateway
tar -xpf /lab/media/13.1-RELEASE/base.txz -C /lab/gateway
```

**编辑 jail.conf**
```
vim /etc/jail.conf
```

**添加到文件底部**
```
gateway {
  ip4=inherit;
}
```

**可以使用以下可选命令将用户帐户添加到 jail 中，在本文中，我们将使用 root 用户**
```
chroot /lab/gateway adduser
```

**设置 jail 的 root 密码**
```
chroot /lab/gateway passwd root
```

**使用 OpenDNS 服务器设置 DNS 解析**
```
vim /lab/gateway/etc/resolv.conf
```
**在 resolv.conf 中添加以下行**
```
nameserver  208.67.222.222
nameserver  208.67.220.220
```

**复制主机时区设置**
```
cp  /etc/localtime  /lab/gateway/etc/
```

**创建一个空的文件系统列表**
```
touch /lab/gateway/etc/fstab
```

**启动 jail**
```
jail  -vc  gateway
```

**登录 jail**
```
jexec  -l  gateway  login  -f  root
```

**退出 jail 登录**
```
logout
```

**列出 jail**
```
jls
```

**停止 jail**
```
jail  -vr  gateway
```
**创建 devfs.rules**
```
vim  /etc/devfs .rules
```
**添加如下行到 devfs.rules**
```
[devfsrules_jail_gateway=666]
add  include  $devfsrules_jail_vnet
add  path   'bpf*'  unhide
```

**重启 devfs**
```
service  devfs  restart
```

**验证 devfs rules**
```
devfs  rule  showsets
```

**将我们的新规则集分配给 gateway jail**
```
vim  /etc/jail .conf
```

**将以下行添加到 gateway｛｝配置块**
```
devfs_ruleset=666;
```
**重启 gateway jail**
```
service  jail  restart  gateway
```

**验证规则集是否应用于 gateway jail**
```
jls  -j  gateway  devfs_ruleset
```

**我们希望看到上述命令的输出为 666**

**在主机上加载 PF 的内核模块**
```
sysrc  -f  /boot/loader .conf  pf_load=YES kldload pf
```

**在 gateway jail 上启用 pF**
```
sysrc  -j  gateway  pf_enable=YES
```

**在 gateway jail 上编辑 pf.conf**
```
vim  /lab/gateway/etc/pf .conf
```

**添加如下配置**
```
ext_if  =  “e0b_gateway”
int_if  =  “e1b_gateway”
table  <rfc1918>  const  {  192 . 168 .0 .0/16,  172 . 16 .0 .0/12,  10 .0 .0 .0/8  }

#Allow  anything  on  loopback
set  skip  on  lo0

#Scrub  all  incoming  traffic
scrub  in
no  nat  on  $ext_if  from  $int_if:network  to  <rfc1918>

#NAT  outgoing  traffic
nat  on  $ext_if  inet  from  $int_if:network  to  any  ->  ($ext_if:0)

#Reject  anything  with  spoofed  addresses
antispoof  quick  for  {  $int_if,  lo0  }  inet

#Default  to  blocking  incoming  traffic,  but  allowing  outgoing  traffic block  all
pass  out  all

#Allow  LAN  to  access  the  rest  of  the  world
pass  in  on  $int_if  from  any  to  any
block  in  on  $int_if  from  any  to  self

#Allow  LAN  to  ping  us
pass  in  on  $int_if  inet  proto  icmp  to  self  icmp-type  echoreq
```

## 配置虚拟网络
设置一个接口。这里我们使用名为 **alc0** 的专用网卡，并将签名其命名为 lab0。所有进一步的配置都将使用接口名称 lab0，它被分配到的实际物理设备可以通过编辑一行配置来改变。这样，如果我们把实验室搬到另一台主机上，或者在以后添加一个新的网络接口，并使用不同的名称，如 **igb0、re0或em0**，就很容易切换我们的接口：

```
sysrc  ifconfig_alc0_name=lab0
sysrc  ifconfig_lab0=up
service  netif  restart
```

### 将 jail 接口桥接自动化脚本复制到我们的实验室脚本目录中，并使其可执行

```
mkdir  /lab/scripts
cp  /usr/share/examples/jails/jib  /lab/scripts/
chmod  +x  /lab/scripts/jib
```

**编辑 jail.conf**
```
vim  /etc/jail .conf
```

## gateway jail.conf
**在这一点上，我们准备从继承主机的 ip4 网络转向使用 vnet，从 /etc/jail.conf 中删除 gateway {} 配置块并用以下内容代替**

```
gateway  {
vnet;
vnet .interface=e0b_$name,  e1b_$name;
exec .prestart+=”/lab/scripts/jib  addm  $name  lab0  labnet”;
exec .poststop+=”/lab/scripts/jib  destroy  $name”;
devfs_ruleset=666;
}
```

**为实验室中的 jail 创建内部局域网网络**
```
sysrc  cloned_interfaces=vlan2
sysrc  ifconfig_vlan2_name=labnet
sysrc  ifconfig_labnet=up
service  netif  restart
```

**销毁并重新创建 gateway**
```
jail  -vr  gateway
jail  -vc  gateway
```

**为 gateway jail 配置网络**
```
sysrc  -j  gateway  gateway_enable=YES
sysrc  -j  gateway  ifconfig_e0b_gateway=SYNCDHCP
sysrc  -j  gateway  ifconfig_e1b_gateway=”inet  10 .66 .6 . 1/24”
service  jail  restart  gateway
jexec  -l  gateway  login  -f  root
```

**测试连通性**
```
host  bsd .pw
ping  -c  3  bsd .pw
```

**离开 jail**
```
logout
```

**创建另一个 jail，该 jail 只有一个连接到 labnet 局域网的接口**
```
vim  /etc/jail .conf
```

**将以下内容添加到文件底部**
```
client1  {
vnet;
vnet .interface=”e0b_$name”;
exec .prestart+=”/lab/scripts/jib  addm  $name  labnet”;
exec .poststop+=”/lab/scripts/jib  destroy  $name”;
devfs_ruleset=4;
depend=”gateway”;
}
```

**制定新 jail 的目录结构**
```
mkdir  /lab/client1
tar  -xpf  /lab/media/13 . 1-RELEASE/base .txz  -C  /lab/client1
```

**设置 root 密码**
```
chroot  /lab/client1  passwd  root
```

**使用 OpenDNS 服务器设置 DNS 解析**
```
vim  /lab/client1/etc/resolv .conf
```

**添加如下行到 resolv.conf**
```
nameserver  208.67.222.222
nameserver  208.67.220.220
```

**复制主机时区设置**
```
cp  /etc/localtime  /lab/client1/etc/
```

**创建一个空的文件系统列表**
```
touch  /lab/client1/etc/fstab
start jail
jail  -vc  client1
```

**为 client1 jail 配置网络**
```
sysrc  -j  client1  ifconfig_e0b_client1=”inet  10.66.6.2/24”
sysrc  -j  client1  defaultrouter=”10.66 .6.1”
service jail restart client1
```

**登录 jail**
```
jexec  -l  client1  login  -f  root
```

***测试连通性*
```
host  bsd .pw
ping  -c  3  bsd .pw
ping  -c  3  10.66.6.1
```

**获取 tcsh 配置文件示例**
```
fetch  -o  .tcshrc  http://bsd .pw/config/tcshrc
chsh  -s  tcsh
```

**退出 jail**
```
logout
```

下次登录时，由于 tcshrc 的设置，你会有一个绿色的提示，请尽情享受吧。现在你有了一个虚拟实验室，它有自己的虚拟网络，有一个物理接口用于向外连接互联网。由于我们把这个接口命名为 lab0，我们可以很容易地更新它。来吧，试一试。例如，你可以插入一个有互联网连接的安卓手机，WiFi 或蜂窝电话，都可以实现。在你把手机插入主机后，进入手机的网络设置，启用 USB 以太网链接。现在将有一个 ue0 接口可供使用。将 **/etc/rc.conf** 中的 **ifconfig_alc0_name="lab0"** 一行更新为 **ifconfig_ue0_name="lab0"**。重启。登录到任一 jail 并测试连通性。你的网络已经被换掉了。你的实验室现在可以移动了！

我希望你在阅读这篇文章的时倍感开心。我对 FreeBSD 充满热情，与他人分享我所学到的东西给我带来了快乐。我希望你能在你的实验室里用 FreeBSD 做一些了不起的的事情，我期待着在一个精彩的 BSD 会议上与你们中的许多人交流。

---
**ROLLER ANGEL** 大部分时间都在帮助人们学习如何使用技术来实现他们的目标。他是一个狂热的 FreeBSD 系统管理员和 Python 爱好者，喜欢学习使用令人惊叹的开源技术(尤其是 FreeBSD 和 Python) 解决问题。他坚信，人们可以学习任何他们想学的东西。Roller 一直在寻求创造性的解决问题的方法，并乐于接受良好的挑战。他有学习的动力和动机，探索新的想法，并保持他的技术敏锐性。他喜欢参与研究社区并分享自己的想法。
