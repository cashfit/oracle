# 使用VirtualBox在Oracle Linux 7上安装Oracle Database 19c RAC

**最新更新为2019-11-20，发布于2020-5-4**

本文介绍了使用VirtualBox（6.0.10）在Linux（Oracle Linux 7.7 64位）上安装Oracle Database 19c RAC，而没有其他共享磁盘设备的情况。

## 介绍

阻碍人们建立测试RAC环境的最大障碍之一是对共享存储的需求。在生产环境中，共享存储通常由SAN或高端NAS设备提供，但是当您要做的只是获得一些安装和使用RAC的经验时，这两种选择都非常昂贵。一种更便宜的选择是使用FireWire磁盘盒，以允许两台计算机访问相同的磁盘，但这仍然要花钱，并且需要两台服务器。第三种选择是使用虚拟化来伪造共享存储。

使用VirtualBox，您可以在一台服务器上运行多个虚拟机（VM），从而使您可以在一台计算机上运行两个RAC节点。此外，它还允许您设置共享虚拟磁盘，从而克服了昂贵的共享存储的障碍。

![19c_RAC_install](./19c_RAC_install/19c_arch_Diagram.png "Introduction")


在开始进行此安装之前，请考虑以下几点。
- 完成的系统包括主机操作系统，三个来宾操作系统，两套Oracle Grid Infrastructure（集群软件+ ASM）和两个数据库实例，它们都在同一台计算机上。可以想象，这需要大量的磁盘空间，CPU和内存。
- 从最后一点开始，每个RAC节点VM至少需要6G的RAM，但是您会看到我为每个使用了10G，但它仍然很慢。不要以为您可以在小型PC或笔记本电脑上运行它。你不会的
- 此过程提供了基本的安装，以使RAC正常工作。 Grid Infrastructure安装或ASM安装中没有冗余。要添加此文件，只需创建两倍数量的共享磁盘，然后在提供时选择“普通”冗余选项。当然，这将占用更多的磁盘空间。
- 这不是（也不应视为）生产就绪系统的说明。只是为了让您了解安装RAC所需的条件并为您提供一个可以进行试验的系统。
- 需要DNS以支持扫描侦听器。在以前的版本中，我建议在主机服务器上运行DNS，但这比较容易。
- 本文使用Oracle Linux和Oracle 19c的64位版本。
- 在服务器上进行此安装时，我将虚拟磁盘拆分为不同的物理磁盘。这不是必需的，但是会使运行更快。
- 这可能需要三个多小时才能完成。如果存在严重的内存或磁盘速度限制，则可能更长一些。

## 下载软件

下载以下软件。
- [Oracle Linux 7](http://edelivery.oracle.com/linux) (使用最新版本，例如7.7)
- [VirtualBox (6.0.10)](http://www.virtualbox.org/wiki/Downloads)
- [Oracle 19c (19.3) 软件 (64 bit)](http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html)

本文已针对19c版本进行了更新，但自12.2.0.1起安装基本上没有变化。 19c特有的任何变化都将被注意。

根据您的VirtualBox和Oracle Linux版本，屏幕快照的外观可能会略有不同。

## VirtualBox安装

首先，安装VirtualBox软件。在RHEL及其克隆上，您可以使用以下类型的命令作为root用户来执行此操作。

```console
# rpm -Uvh VirtualBox*.rpm
```

程序包名称将根据所使用的主机分发版本而有所不同。完成后，从菜单启动VirtualBox。

## VirtualBox网络设置
我们需要确保配置了仅主机的网络，并检查/修改了该网络的IP范围。这将是我们RAC安装的公共网络。

- 从菜单中启动VirtualBox。
- 选择“工具”选项。
- 在弹出菜单中单击“网络”。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-113524@2x.jpg "VirtualBox Network Setup")
- 点击屏幕右侧尺寸上的“创建”按钮。将创建一个名为“vboxnet0”的网络。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-113536@2x.jpg "VirtualBox Network Setup")
- 如果您要为公共地址使用其他子网，则可以在此处更改网络详细信息。只要确保您选择的子网与网络上的任何实际子网都不匹配即可。我决定使用默认值，对我来说是“192.168.56.X”。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-113559@2x.jpg "VirtualBox Network Setup")

## 虚拟机设置

现在，我们必须定义两个虚拟RAC节点。 我们可以通过定义一个VM，然后在安装它时将其克隆来节省时间。

启动VirtualBox并单击工具栏上的“新建”按钮。输入名称“ol7-19c-rac1”，操作系统“Linux”和版本“Oracle（64位）”，然后单击“继续”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-112613@2x.jpg "Virtual Machine Setup")
输入“ 4096”作为基本内存大小，然后单击“继续”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-112838@2x.jpg "Virtual Machine Setup")
通过单击“创建”按钮，接受默认选项以创建新的虚拟硬盘。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-112911@2x.jpg "Virtual Machine Setup")
通过单击“继续”按钮接受默认的硬盘驱动器文件类型。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-112924@2x.jpg "Virtual Machine Setup")
通过单击“继续”按钮接受“动态分配”选项。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-112939@2x.jpg "Virtual Machine Setup")
接受默认位置并将大小设置为“50G”，然后单击“创建”按钮。如果可以将虚拟磁盘分散到不同的物理磁盘上，则可以提高性能。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-113049@2x.jpg "Virtual Machine Setup")
“ol7-19c-rac1” VM将出现在左侧窗格中。向下滚动右侧的详细信息，然后单击“网络”链接。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-113536@2x.jpg "Virtual Machine Setup")
确保启用了“适配器1”，将其设置为“NAT”，然后单击“适配器2”选项卡。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-113617@2x.jpg "Virtual Machine Setup")
确保启用了“适配器2”，将其设置为“仅主机适配器”，然后单击“适配器3”选项卡。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-113639@2x.jpg "Virtual Machine Setup")
确保启用了“适配器3”，将其设置为“内部网络”，然后单击“系统”部分。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-113708@2x.jpg "Virtual Machine Setup")
将“硬盘”移至启动顺序的顶部，然后取消选中“软盘”选项，然后单击“确定”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-113747@2x.jpg "Virtual Machine Setup")
现在已配置了虚拟机，因此我们可以开始客户机操作系统安装。

## 虚拟机操作系统安装
在突出显示新VM的情况下，单击工具栏上的“开始”按钮。在“选择启动磁盘”屏幕上，选择相关的Oracle Linux ISO映像，然后单击“启动”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-113838@2x.jpg "Guest Operating System Installation")
出现的控制台窗口将包含Oracle Linux引导屏幕。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-113956@2x.jpg "Guest Operating System Installation")

![19c_RAC_install](./19c_RAC_install/Jietu20191119-114118@2x.jpg "Guest Operating System Installation")
像安装基本服务器一样，继续安装Oracle Linux 7。可以在此处找到安装的一般图片指南。更具体地说，应该是至少安装4G +交换分区，禁用防火墙，将SELinux设置为宽松并安装以下软件包组的服务器安装：
- 带有GUI的服务器
- 硬件监控实用程序
- 大型系统性能
- 网络文件系统客户端
- 性能工具
- 兼容性库
- 开发工具

为了与本文的其余部分保持一致，应在安装过程中设置以下信息。
- 主机名：ol7-19c-rac1.localdomain
- enp0s3 (eth0): DHCP（自动连接）
- enp0s8 (eth1): IP=192.168.56.101, Subnet=255.255.255.0, Gateway=192.168.56.1, DNS=192.168.56.1, Search=localdomain（自动连接）
- enp0s9 (eth2): IP=192.168.1.101, Subnet=255.255.255.0, Gateway=<blank>, DNS=<blank>, Search=<blank>（自动连接）

您可以随意更改IP地址以适合您的网络，但是请记住在本文的其余部分中始终与这些调整保持一致。同样，在本文中，我将网络适配器称为enp0s3，enp0s8和enp0s9。在以前的Linux版本中，它们分别是eth0，eth1和eth2。

## Oracle安装先决条件
执行自动设置或手动设置以完成基本先决条件。 所有安装都需要附加设置。

### 自动设置
如果计划使用“oracle-rdbms-server-12cR1-preinstall”软件包执行所有必备安装程序，请发出以下命令。
```console
# yum install -y oracle-database-preinstall-19c
```

早期版本的Oracle Linux需要按照http://public-yum.oracle.com上的说明手动设置Yum存储库。
 
也可能值得进行完整的更新，但是严格来讲这不是必需的。
 ```console
 # yum update -y
 ```

### 手动设置
如果尚未使用“oracle-database-preinstall-19c”软件包执行所有先决条件，则需要手动执行以下安装任务。

将以下行添加到“/etc/sysctl.conf”文件中，或添加到名为“/etc/sysctl.d/98-oracle.conf”的文件中。

```console
fs.file-max = 6815744
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
kernel.panic_on_oops = 1
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
fs.aio-max-nr = 1048576
net.ipv4.ip_local_port_range = 9000 65500
```
运行以下命令来更改当前内核参数。
```console
/sbin/sysctl -p
# or
/sbin/sysctl -p /etc/sysctl.d/98-oracle.conf
```

将以下行添加到名为“/etc/security/limits.d/oracle-database-server-19c-preinstall.conf”的文件中。
```console
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc    16384
oracle   hard   nproc    16384
oracle   soft   stack    10240
oracle   hard   stack    32768
oracle   hard   memlock    134217728
oracle   soft   memlock    134217728
```

除了基本的OS安装外，还必须以root用户身份登录时安装以下软件包。 这包括某些软件包的64位和32位版本。
```console
# From Public Yum or ULN
yum install -y bc    
yum install -y binutils
yum install -y compat-libcap1
yum install -y compat-libstdc++
yum install -y elfutils-libelf
yum install -y elfutils-libelf-devel
yum install -y fontconfig-devel
yum install -y glibc
yum install -y glibc-devel
yum install -y ksh
yum install -y libaio
yum install -y libaio-devel
yum install -y libdtrace-ctf-devel
yum install -y libXrender
yum install -y libXrender-devel
yum install -y libX11
yum install -y libXau
yum install -y libXi
yum install -y libXtst
yum install -y libgcc
yum install -y libstdc++
yum install -y libstdc++-devel
yum install -y libxcb
yum install -y make
yum install -y net-tools # Clusterware
yum install -y nfs-utils # ACFS
yum install -y python # ACFS
yum install -y python-configshell # ACFS
yum install -y python-rtslib # ACFS
yum install -y python-six # ACFS
yum install -y targetcli # ACFS
yum install -y smartmontools
yum install -y sysstat
```

创建新的组和用户。

```console
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 oper
#groupadd -g 54324 backupdba
#groupadd -g 54325 dgdba
#groupadd -g 54326 kmdba
#groupadd -g 54327 asmdba
#groupadd -g 54328 asmoper
#groupadd -g 54329 asmadmin

useradd -u 54321 -g oinstall -G dba,oper oracle
```

您可以定义其他组，并将其分配给“oracle”用户。 这样可以在安装过程中分配各个组。 对于此安装，我只使用了“dba”组。

```console
groupadd -g 54324 backupdba
groupadd -g 54325 dgdba
groupadd -g 54326 kmdba
groupadd -g 54327 asmdba
groupadd -g 54328 asmoper
groupadd -g 54329 asmadmin
groupadd -g 54330 racdba

useradd -u 54321 -g oinstall -G dba,oper,backupdba,dgdba,kmdba,asmdba,asmoper,asmadmin,racdba oracle
```

### 其他设置
无论您是手动还是自动设置，都必须执行以下步骤。
以root用户身份登录到“ol7-19c-rac1”虚拟机时，执行以下步骤。
设置“oracle”用户的密码。
```console
passwd oracle
```

除了本地主机地址外，“/etc/hosts”文件可以保留为空白，但是我更喜欢将这些地址放入以供参考。
```console
127.0.0.1       localhost.localdomain   localhost
# Public
192.168.56.101   ol7-19c-rac1
192.168.56.102   ol7-19c-rac2
# Private
192.168.1.101   ol7-19c-rac1-priv
192.168.1.102   ol7-19c-rac2-priv
# Virtual
192.168.56.103   ol7-19c-rac1-vip
192.168.56.104   ol7-19c-rac2-vip
# SCAN
#192.168.56.105   ol7-19c-scan
```

由于必须使用DNS解析SCAN地址，因此已将其从主机文件中注释掉，因此它可以在与公用IP相同的子网上的3个地址之间循环漫游。 可以使用[BIND](https://oracle-base.com/articles/linux/dns-configuration-for-scan)或[Dnsmasq](https://oracle-base.com/articles/linux/dnsmasq-for-simple-dns-configurations)，这要简单得多。 如果使用的是Dnsmasq，则将RAC特定的条目放在主机的“/etc/host”文件中，并取消注释SCAN条目，然后重新启动Dnsmasq。
确保“/etc/resolv.conf”文件包含指向正确名称服务器的名称服务器条目。 另外，如果同时存在“域”和“搜索”条目，请注释掉其中之一。 对于此安装，我的“/etc/resolv.conf”看起来像这样。
```console
#domain localdomain
search localdomain
nameserver 192.168.56.1
```

由于存在NAT接口，对“resolv.conf”的更改将被网络管理器覆盖。 因此，现在应该在启动时禁用此接口。 如果需要从VM访问Internet，则可以手动启用它。 编辑“/etc/sysconfig/network-scripts/ifcfg-eth0”文件，进行以下更改。 这将在下一次重新启动后生效。
```console
ONBOOT=no
```

无需立即重新启动。 您可以只运行以下命令。
```console
# ifdown enp0s3
```

此时，第一个节点的联网应类似于以下内容。 请注意，enp0s3没有关联的IP地址，因为它已被禁用。
```console
# ifconfig
enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 08:00:27:f6:88:78  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.56.101  netmask 255.255.255.0  broadcast 192.168.56.255
        inet6 fe80::cf8d:317d:534:17d9  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:82:06:32  txqueuelen 1000  (Ethernet)
        RX packets 574  bytes 54444 (53.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 547  bytes 71219 (69.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s9: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.101  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::9a9a:f249:61d1:5447  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:2e:2c:cf  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 29  bytes 4250 (4.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 68  bytes 5780 (5.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 68  bytes 5780 (5.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:4a:12:2f  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

#
```

有了此设置并配置了DNS之后，就可以将SCAN地址解析为所有三个IP地址。
```console
# nslookup ol7-19c-scan
Server:		192.168.56.1
Address:	192.168.56.1#53

Name:	ol7-19c-scan.localdomain
Address: 192.168.56.105
#
```

如下所述修改“/etc/security/limits.d/90-nproc.conf”文件。 请参阅[MOS文档[ID 1487773.1]](https://support.oracle.com/epmos/faces/DocContentDisplay?id=1487773.1)
```console
# Change this
*          soft    nproc    1024

# To this
* - nproc 16384
```

通过编辑“/etc/selinux/config”文件，将SELinux的设置更改为宽松，确保SELINUX标志设置如下。
```console
SELINUX=permissive
```

如果启用了Linux防火墙，则需要禁用或配置它，如[这里](https://oracle-base.com/articles/linux/oracle-linux-6-installation#firewall)或[这里](https://oracle-base.com/articles/linux/linux-firewall#installation)所示。 以下是禁用防火墙的示例。
```console
# systemctl stop firewalld
# systemctl disable firewalld
```

请配置NTP，或确保未配置NTP，以便Oracle集群时间同步服务（ctssd）可以同步RAC节点的时间。

确保启用了NTP（在OL7/RHEL7上为Chrony）。
```console
# systemctl enable chronyd
# systemctl restart chronyd
# chronyc -a 'burst 4/4'
# chronyc -a makestep
```

创建将在其中安装Oracle软件的目录。
```console
mkdir -p /u01/app/19.0.0/grid
mkdir -p /u01/app/oracle/product/19.0.0/db_1
chown -R oracle:oinstall /u01
chmod -R 775 /u01/
```

以“oracle”用户身份登录，并将以下行添加到“/home/oracle/.bash_profile”文件的末尾。
```console
# Oracle Settings
export TMP=/tmp
export TMPDIR=$TMP

export ORACLE_HOSTNAME=ol7-19c-rac1
export ORACLE_UNQNAME=CDBRAC
export ORACLE_BASE=/u01/app/oracle
export GRID_HOME=/u01/app/19.0.0/grid
export DB_HOME=$ORACLE_BASE/product/19.0.0/db_1
export ORACLE_HOME=$DB_HOME
export ORACLE_SID=cdbrac1
export ORACLE_TERM=xterm
export BASE_PATH=/usr/sbin:$PATH
export PATH=$ORACLE_HOME/bin:$BASE_PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib

alias grid_env='. /home/oracle/grid_env'
alias db_env='. /home/oracle/db_env'
```

创建具有以下内容的名为“/home/oracle/grid_env”的文件。
```console
export ORACLE_SID=+ASM1
export ORACLE_HOME=$GRID_HOME
export PATH=$ORACLE_HOME/bin:$BASE_PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```

创建具有以下内容的名为“/home/oracle/db_env”的文件。
```console
export ORACLE_SID=cdbrac1
export ORACLE_HOME=$DB_HOME
export PATH=$ORACLE_HOME/bin:$BASE_PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```

一旦运行了“/home/oracle/.bash_profile”，您就可以在环境之间进行如下切换。
```console
$ grid_env
$ echo $ORACLE_HOME
/u01/app/19.0.0/grid
$ db_env
$ echo $ORACLE_HOME
/u01/app/oracle/product/19.0.0/db_1
$
```

我们进行了很多更改，因此此时值得重新启动VM，以确保所有更改都已生效。
```console
# shutdown -r now
```

## 安装访客附加

单击VM屏幕顶部的“设备>安装来宾添加项”菜单选项。 如果您选择了自动运行，请选择它。 如果没有，请运行以下命令。
```console
cd /run/media/VBOXADDITIONS*
sh ./VBoxLinuxAdditions.run
```

将“oracle”用户添加到“vboxsf”组中，以便它可以访问共享驱动器。
```console
# usermod -G vboxsf,dba oracle
# id oracle
uid=54321(oracle) gid=54321(oinstall) groups=54321(oinstall),54322(dba),54323(vboxsf)
#
```
在虚拟机上创建一个共享文件夹（设备>共享文件夹），指向主机上解压缩Oracle软件的目录。 在单击“确定”按钮之前，请检查“自动安装”和“永久”选项。

需要重新启动VM，才能正确使用来宾添加。 下一部分需要关闭，因此此时无需其他重新启动。 VM重新启动后，“oracle”用户将可以访问名为“/run/media/sf_19.0.0”的共享文件夹。
## 创建共享磁盘

使用以下命令关闭“ol7-19c-rac1”虚拟机。
```console
# shutdown -h now
```
在主机服务器上，创建4个可共享虚拟磁盘，并使用以下命令将它们关联为虚拟介质。 您可以选择其他位置，但请确保它们在现有VM目录之外。
```console
$ mkdir -p /u04/VirtualBox/ol7-19c-rac
$ cd /u04/VirtualBox/ol7-19c-rac
$
$ # Create the disks and associate them with VirtualBox as virtual media.
$ VBoxManage createhd --filename asm1.vdi --size 10240 --format VDI --variant Fixed
$ VBoxManage createhd --filename asm2.vdi --size 10240 --format VDI --variant Fixed
$ VBoxManage createhd --filename asm3.vdi --size 10240 --format VDI --variant Fixed
$ VBoxManage createhd --filename asm4.vdi --size 10240 --format VDI --variant Fixed
$
$ # Connect them to the VM.
$ VBoxManage storageattach ol7-19c-rac1 --storagectl "SATA" --port 1 --device 0 --type hdd \
    --medium asm1.vdi --mtype shareable
$ VBoxManage storageattach ol7-19c-rac1 --storagectl "SATA" --port 2 --device 0 --type hdd \
    --medium asm2.vdi --mtype shareable
$ VBoxManage storageattach ol7-19c-rac1 --storagectl "SATA" --port 3 --device 0 --type hdd \
    --medium asm3.vdi --mtype shareable
$ VBoxManage storageattach ol7-19c-rac1 --storagectl "SATA" --port 4 --device 0 --type hdd \
    --medium asm4.vdi --mtype shareable
$
$ # Make shareable.
$ VBoxManage modifyhd asm1.vdi --type shareable
$ VBoxManage modifyhd asm2.vdi --type shareable
$ VBoxManage modifyhd asm3.vdi --type shareable
$ VBoxManage modifyhd asm4.vdi --type shareable

```
如果使用Windows主机，则必须修改路径，但是过程相同。
```console
C:
mkdir C:\VirtualBox\ol7-19c-rac
cd C:\VirtualBox\ol7-19c-rac

"c:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename asm1.vdi --size 10240 --format VDI --variant Fixed
"c:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename asm2.vdi --size 10240 --format VDI --variant Fixed
"c:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename asm3.vdi --size 10240 --format VDI --variant Fixed
"c:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename asm4.vdi --size 10240 --format VDI --variant Fixed

"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac1 --storagectl "SATA" --port 1 --device 0 --type hdd --medium asm1.vdi --mtype shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac1 --storagectl "SATA" --port 2 --device 0 --type hdd --medium asm2.vdi --mtype shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac1 --storagectl "SATA" --port 3 --device 0 --type hdd --medium asm3.vdi --mtype shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac1 --storagectl "SATA" --port 4 --device 0 --type hdd --medium asm4.vdi --mtype shareable

"c:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd asm1.vdi --type shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd asm2.vdi --type shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd asm3.vdi --type shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd asm4.vdi --type shareable
```
通过单击工具栏上的“开始”按钮来启动“ol7-19c-rac1”虚拟机。 服务器启动后，以root用户身份登录，以便您可以配置共享磁盘。 发出以下命令可以看到当前磁盘。
```console
# cd /dev
# ls sd*
sda  sda1  sda2  sdb  sdc  sdd  sde
#
```
使用“fdisk”命令将磁盘sdb分区到sde。 以下输出显示了sdb磁盘的预期fdisk输出。
```console
# fdisk /dev/sdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x14a4629c.

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-41943039, default 2048): 
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-41943039, default 41943039): 
Using default value 41943039
Partition 1 of type Linux and of size 20 GiB is set

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
#
```
在每种情况下，答案的顺序为“n”，“p”，“1”，“回车”，“回车”和“w”。
对所有磁盘进行分区后，可以通过重复前面的“ls”命令来查看结果。
```console
# cd /dev
# ls sd*
sda  sda1  sda2  sdb  sdb1  sdc  sdc1  sdd  sdd1  sde  sde1
#
```
配置您的UDEV规则，如[此处](https://oracle-base.com/articles/linux/udev-scsi-rules-configuration-in-oracle-linux)所示。
将以下内容添加到“/etc/scsi_id.config”文件中，以将SCSI设备配置为受信任。 创建文件（如果尚不存在）。
```console
options=-g
```
我的磁盘的SCSI ID显示在下面。
```console
# /usr/lib/udev/scsi_id -g -u -d /dev/sdb1
1ATA_VBOX_HARDDISK_VB189c7a69-689f61b0
# /usr/lib/udev/scsi_id -g -u -d /dev/sdc1
1ATA_VBOX_HARDDISK_VBc4ae174e-fc756d12
# /usr/lib/udev/scsi_id -g -u -d /dev/sdd1
1ATA_VBOX_HARDDISK_VBa4e03079-ae751cbd
# /usr/lib/udev/scsi_id -g -u -d /dev/sde1
1ATA_VBOX_HARDDISK_VBf00747dc-10252f06
#
```
使用这些值，编辑“/etc/udev/rules.d/99-oracle-asmdevices.rules”文件，并添加以下4个条目。 单个条目的所有参数必须在同一行上。
```console
KERNEL=="sd?1", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id -g -u -d /dev/$parent", RESULT=="1ATA_VBOX_HARDDISK_VB189c7a69-689f61b0", SYMLINK+="oracleasm/asm-disk1", OWNER="oracle", GROUP="dba", MODE="0660"
KERNEL=="sd?1", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id -g -u -d /dev/$parent", RESULT=="1ATA_VBOX_HARDDISK_VBc4ae174e-fc756d12", SYMLINK+="oracleasm/asm-disk2", OWNER="oracle", GROUP="dba", MODE="0660"
KERNEL=="sd?1", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id -g -u -d /dev/$parent", RESULT=="1ATA_VBOX_HARDDISK_VBa4e03079-ae751cbd", SYMLINK+="oracleasm/asm-disk3", OWNER="oracle", GROUP="dba", MODE="0660"
KERNEL=="sd?1", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id -g -u -d /dev/$parent", RESULT=="1ATA_VBOX_HARDDISK_VBf00747dc-10252f06", SYMLINK+="oracleasm/asm-disk4", OWNER="oracle", GROUP="dba", MODE="0660"
```
加载更新的块设备分区表。
```console
# /sbin/partprobe /dev/sdb1
# /sbin/partprobe /dev/sdc1
# /sbin/partprobe /dev/sdd1
# /sbin/partprobe /dev/sde1
```
测试规则是否按预期工作。
```console
# /sbin/udevadm test /block/sdb/sdb1
```
重新加载UDEV规则并启动UDEV。
```console
# /sbin/udevadm control --reload-rules
```
现在，磁盘应该可见，并使用以下命令具有正确的所有权。 如果看不到它们，则说明您的UDEV配置不正确，必须先进行修复，然后再继续。
```console
# ls -al /dev/oracleasm/*
lrwxrwxrwx. 1 root root 7 Mar  6 17:41 /dev/oracleasm/asm-disk1 -> ../sdb1
lrwxrwxrwx. 1 root root 7 Mar  6 17:41 /dev/oracleasm/asm-disk2 -> ../sdc1
lrwxrwxrwx. 1 root root 7 Mar  6 17:41 /dev/oracleasm/asm-disk3 -> ../sdd1
lrwxrwxrwx. 1 root root 7 Mar  6 17:41 /dev/oracleasm/asm-disk4 -> ../sde1
#
```
符号链接由root拥有，但是它们指向的设备现在具有正确的所有权。
```console
# ls -al /dev/sd*1
brw-rw----. 1 root   disk 8,  1 Apr 25 14:11 /dev/sda1
brw-rw----. 1 oracle dba  8, 17 Apr 25 14:11 /dev/sdb1
brw-rw----. 1 oracle dba  8, 33 Apr 25 14:11 /dev/sdc1
brw-rw----. 1 oracle dba  8, 49 Apr 25 14:11 /dev/sdd1
brw-rw----. 1 oracle dba  8, 65 Apr 25 14:11 /dev/sde1
#
```
现在，已为网格基础结构配置了共享磁盘。

## 克隆虚拟机
更高版本的VirtualBox允许您克隆VM，但这些虚拟机还会尝试克隆共享磁盘，这不是我们想要的。 相反，我们必须手动克隆VM。
使用以下命令关闭“ol7-19c-rac1”虚拟机。
```console
# shutdown -h now
```
在主机服务器上使用以下命令手动克隆“ol7-19c-rac1.vdi”磁盘。
```console
$ mkdir -p /u03/VirtualBox/ol7-19c-rac2
$ VBoxManage clonehd /u01/VirtualBox/ol7-19c-rac1/ol7-19c-rac1.vdi /u03/VirtualBox/ol7-19c-rac2/ol7-19c-rac2.vdi

Rem Windows
mkdir "C:\VirtualBox\ol7-19c-rac2"
"c:\Program Files\Oracle\VirtualBox\VBoxManage" clonehd "C:\VirtualBox\ol7-19c-rac1\ol7-19c-rac1.vdi" "C:\VirtualBox\ol7-19c-rac2\ol7-19c-rac2.vdi"
```
除了使用现有的“ol7-19c-rac2.vdi”虚拟硬盘驱动器外，以与“ol7-19c-rac1”相同的方式在VirtualBox中创建“ol7-19c-rac2”虚拟机。

记住要像在“ol7-19c-rac1” VM上那样添加三个网络适配器。 创建虚拟机后，将共享磁盘连接到该虚拟机。
```console
$ # Linux : Switch to the shared storage location and attach them.
$ cd /u04/VirtualBox/ol7-19c-rac
$
$ VBoxManage storageattach ol7-19c-rac2 --storagectl "SATA" --port 1 --device 0 --type hdd --medium asm1.vdi --mtype shareable
$ VBoxManage storageattach ol7-19c-rac2 --storagectl "SATA" --port 2 --device 0 --type hdd --medium asm2.vdi --mtype shareable
$ VBoxManage storageattach ol7-19c-rac2 --storagectl "SATA" --port 3 --device 0 --type hdd --medium asm3.vdi --mtype shareable
$ VBoxManage storageattach ol7-19c-rac2 --storagectl "SATA" --port 4 --device 0 --type hdd --medium asm4.vdi --mtype shareable


Rem Windows : Switch to the shared storage location and attach them.
C:
cd C:\VirtualBox\ol7-19c-rac

"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac2 --storagectl "SATA" --port 1 --device 0 --type hdd --medium asm1.vdi --mtype shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac2 --storagectl "SATA" --port 2 --device 0 --type hdd --medium asm2.vdi --mtype shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac2 --storagectl "SATA" --port 3 --device 0 --type hdd --medium asm3.vdi --mtype shareable
"c:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach ol7-19c-rac2 --storagectl "SATA" --port 4 --device 0 --type hdd --medium asm4.vdi --mtype shareable
```
通过单击工具栏上的“开始”按钮来启动“ol7-19c-rac2”虚拟机。 在启动过程中忽略任何网络错误。

以“ root”用户身份登录到“ ol7-19c-rac2”虚拟机，以便我们可以重新配置网络设置以匹配以下内容。
- 主机名：ol7-19c-rac2.localdomain
- enp0s3 (eth0): DHCP （*不*自动连接）
- enp0s8 (eth1): IP=192.168.56.102, Subnet=255.255.255.0, Gateway=192.168.56.1, DNS=192.168.56.1, Search=localdomain （自动连接）
- enp0s9 (eth2): IP=192.168.1.102, Subnet=255.255.255.0, Gateway=<blank>, DNS=<blank>, Search=<blank> （自动连接）

修改“/etc/hostname”文件中的主机名。
```console
ol7-19c-rac2.localdomain
```
与以前的Linux版本不同，我们不必编辑与网络适配器关联的MAC地址，但是我们必须更改其IP地址。

编辑“/etc/sysconfig/network-scripts/ifcfg-enp0s8”（eth1），仅如下修改IPADDR设置并删除UUID条目。
```console
IPADDR=192.168.56.102 
```
编辑“/etc/sysconfig/network-scripts/ifcfg-enp0s9”（eth2），仅修改IPADDR设置，如下所示，并删除UUID条目。
```console
IPADDR=192.168.1.102 
```
重新启动虚拟机。
```console
# shutdown -r now
```
此时，第二个节点的联网应如下所示。 请注意，enp0s3没有关联的IP地址，因为它已被禁用。
```console
# ifconfig
enp0s3 : flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 08:00:27:dc:7c:74  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.56.102  netmask 255.255.255.0  broadcast 192.168.56.255
        inet6 fe80::a00:27ff:fed9:c89a  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:d9:c8:9a  txqueuelen 1000  (Ethernet)
        RX packets 197  bytes 19460 (19.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 178  bytes 27171 (26.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s9: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.102  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::a00:27ff:feb4:6bf  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:b4:06:bf  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 30  bytes 4112 (4.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 4  bytes 420 (420.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4  bytes 420 (420.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

#
```
编辑“ol7-19c-rac2”节点上的“/home/oracle/.bash_profile”文件，以更正ORACLE_SID和ORACLE_HOSTNAME值。
```console
export ORACLE_SID=cdbrac2
export ORACLE_HOSTNAME=ol7-19c-rac2.localdomain
```
另外，修改“/home/oracle/db_env”和“/home/oracle/grid_env”文件中的ORACLE_SID设置。

重新启动“ol7-19c-rac2”虚拟机，然后启动“ol7-19c-rac1”虚拟机。 当两个节点都启动后，请使用以下命令检查它们是否可以ping通所有公用和专用IP地址。
```console
ping -c 3 ol7-19c-rac1
ping -c 3 ol7-19c-rac1-priv
ping -c 3 ol7-19c-rac2
ping -c 3 ol7-19c-rac2-priv
```
检查两个节点上的SCAN地址是否仍能正确解析。
```console
# nslookup ol7-19c-scan
Server:		192.168.56.1
Address:	192.168.56.1#53

Name:	ol7-19c-scan.localdomain
Address: 192.168.56.105

#
```
此时，在“/etc/hosts”文件中定义的虚拟IP地址将不起作用，因此不必费心测试它们。
检查UDEV规则是否在两台计算机上都起作用。
```console
# ls -al /dev/oracleasm/*
lrwxrwxrwx. 1 root root 7 Sep 18 08:19 /dev/oracleasm/asm-disk1 -> ../sdb1
lrwxrwxrwx. 1 root root 7 Sep 18 08:19 /dev/oracleasm/asm-disk2 -> ../sdc1
lrwxrwxrwx. 1 root root 7 Sep 18 08:19 /dev/oracleasm/asm-disk3 -> ../sdd1
lrwxrwxrwx. 1 root root 7 Sep 18 08:19 /dev/oracleasm/asm-disk4 -> ../sde1
#
```
在11gR2之前，我们可能会在集群件根目录中使用“runcluvfy.sh”实用程序来检查是否满足先决条件。如果打算使用安装程序配置SSH连接，则应省略此检查，因为它总是会失败。如果要[手动设置SSH连接](https://oracle-base.com/articles/linux/user-equivalence-configuration-on-linux)，则一旦完成，您可以运行“runcluvfy.sh”使用以下命令。
```console
/mountpoint/clusterware/runcluvfy.sh stage -pre crsinst -n ol7-19c-rac1,ol7-19c-rac2 -verbose
```
如果遇到任何故障，请确保在继续操作之前进行纠正。
虚拟机设置现已完成。
在继续之前，您可能应该关闭虚拟机并对其快照。如果在此之后发生任何故障，最好切换回这些快照，清理共享驱动器，然后重新开始网格安装。 [清理共享磁盘](https://oracle-base.com/articles/rac/clean-up-a-failed-grid-infrastructure-installation#asm_disks)的替代方法是现在使用zip，发生故障时只需更换它们。
```console
$ # Linux
$ cd /u04/VirtualBox/ol7-19c-rac
$ zip PreGrid.zip *.vdi

Rem Windows
C:
cd C:\VirtualBox\ol7-19c-rac
zip PreGrid.zip *.vdi
```

##  安装网格基础架构

确保两个虚拟机都已启动。GI现在是映像安装，因此请以“oracle”用户的身份在第一个节点上执行以下操作。
```console
export SOFTWARE_LOCATION=/media/sf_19.0.0/
cd /u01/app/19.0.0/grid
unzip -q $SOFTWARE_LOCATION/linuxx64_1900_grid_home.zip
```
或将网格软件解压缩到第一个节点上的目标目录。不要在第二个节点上执行此操作。
```console
unzip -d /u01/app/19.0.0/grid V982068-01.zip
```
在所有节点上以“root”用户身份从网格主目录安装以下软件包。
```console
su -
# Local node.
cd /u01/app/19.0.0/grid/cv/rpm
rpm -Uvh cvuqdisk*

# Remote node.
scp ./cvuqdisk* root@ol7-19c-rac2:/tmp
ssh root@ol7-19c-rac2 rpm -Uvh /tmp/cvuqdisk*
exit
```
如果您打算使用AFD驱动程序（新的ASMLib），则可以使用asmcmd命令配置共享磁盘，如下所示。 我们正在使用UDEV，因此没有必要。
```console
# !!!! I did not do this! !!!!
su -

# Set environment.
export ORACLE_HOME=/u01/app/19.0.0/grid
export ORACLE_BASE=/tmp

# Mark disks.
$ORACLE_HOME/bin/asmcmd afd_label DISK1 /dev/oracleasm/asm-disk1 --init
$ORACLE_HOME/bin/asmcmd afd_label DISK2 /dev/oracleasm/asm-disk2 --init
$ORACLE_HOME/bin/asmcmd afd_label DISK3 /dev/oracleasm/asm-disk3 --init
$ORACLE_HOME/bin/asmcmd afd_label DISK4 /dev/oracleasm/asm-disk4 --init

# Test Disks.
$ORACLE_HOME//bin/asmcmd afd_lslbl /dev/oracleasm/asm-disk1
$ORACLE_HOME//bin/asmcmd afd_lslbl /dev/oracleasm/asm-disk2
$ORACLE_HOME//bin/asmcmd afd_lslbl /dev/oracleasm/asm-disk3
$ORACLE_HOME//bin/asmcmd afd_lslbl /dev/oracleasm/asm-disk4

# unset environment.
unset ORACLE_BASE

exit
```
通过以“oracle”用户身份运行以下内容来配置Grid Infrastructure。

我可以使用此编辑的响应文件（grid_config.rsp）通过以下命令以静默方式运行配置。
```console
cd /u01/app/19.0.0/grid
./gridSetup.sh -silent -responseFile /tmp/grid_config.rsp
```
相反，这是交互式配置。
```console
cd /u01/app/19.0.0/grid
./gridSetup.sh
```
![19c_RAC_install](./19c_RAC_install/Jietu20191119-184019@2x.jpg "Install the Grid Infrastructure")
选择“为新集群配置Oracle Grid Infrastructure”选项，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-184049@2x.jpg "Install the Grid Infrastructure")
通过单击“下一步”按钮接受“配置Oracle独立集群”选项。

输入群集名称“ol7-19c-cluster”，SCAN名称“ol7-19c-scan”和SCAN端口“1521”，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-184214@2x.jpg "Install the Grid Infrastructure")
在“群集节点信息”屏幕上，单击“添加”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-184312@2x.jpg "Install the Grid Infrastructure")
单击“SSH连接...”按钮，然后输入“oracle”用户的密码。 单击“设置”按钮以配置SSH连接，单击“测试”按钮以对其进行测试。 测试完成后，单击“下一步”按钮。

检查公共和专用网络是否正确指定。 确保enp0s9用于“ASM和专用”，如果显示NAT接口，请记住将其标记为“请勿使用”。 点击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-184917@2x.jpg "Install the Grid Infrastructure")
通过单击“下一步”按钮接受“使用Oracle Flex ASM进行存储”选项。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-184858@2x.jpg "Install the Grid Infrastructure")
选择“否”选项，因为在这种情况下，我们不想为GIMR创建单独的磁盘组。 点击“下一步”按钮。

将冗余设置为“外部”，单击“更改发现路径”按钮，然后将路径设置为“/dev/oracleasm/*”。 返回主屏幕并选择所有4个磁盘。 取消选中“配置Oracle ASM筛选器驱动程序”选项，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-185004@2x.jpg "Install the Grid Infrastructure")
输入凭据，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-185024@2x.jpg "Install the Grid Infrastructure")
通过单击“下一步”按钮接受默认IPMI选项。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-185035@2x.jpg "Install the Grid Infrastructure")
不要在EM注册。 点击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-185045@2x.jpg "Install the Grid Infrastructure")
我们使用的是单个用户，并且组管理这两个ASM都添加了数据库，因此将组设置为“dba”并单击“下一步”按钮。 单击“是”按钮，接受后续对话框中的警告。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-185132@2x.jpg "Install the Grid Infrastructure")
输入Oracle Base位置“/u01/app/oracle”，然后单击“下一步”按钮。 我们已经为以后的数据库安装预先创建了目录，因此通过单击“是”按钮忽略有关Oracle Base不为空的后续警告。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-185158@2x.jpg "Install the Grid Infrastructure")
通过单击“下一步”按钮接受默认清单目录。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-185518@2x.jpg "Install the Grid Infrastructure")
等待先决条件检查完成。如果您有任何问题，请使用“修复并再次检查”按钮。 完成可能的修复后，选中“全部忽略”复选框，然后单击“下一步”按钮。对于这种类型的安装，“物理内存”和“网络时间协议（NTP）”测试可能会失败。这不影响后续安装过程。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-185737@2x.jpg "Install the Grid Infrastructure")
通过选中“全部忽略”以继续安装。

等待安装。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-190342@2x.jpg "Install the Grid Infrastructure")
出现提示时，请在每个节点上运行配置脚本。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-190914@2x.jpg "Install the Grid Infrastructure")
来自“orainstRoot.sh”文件的输出应类似于以下所列。
```console
[root@ol7-19c-rac1 ~]# /u01/app/oraInventory/orainstRoot.sh 
Changing permissions of /u01/app/oraInventory.
Adding read,write permissions for group.
Removing read,write,execute permissions for world.

Changing groupname of /u01/app/oraInventory to oinstall.
The execution of the script is complete.
[root@ol7-19c-rac1 ~]#
```

“root.sh”的输出将有所不同，具体取决于运行该节点的节点。示例输出可以在这里看到。
```console
[root@ol7-19c-rac1 ~]# /u01/app/19.0.0/grid/root.sh
Performing root user operation.

The following environment variables are set as:
    ORACLE_OWNER= oracle
    ORACLE_HOME=  /u01/app/19.0.0/grid

Enter the full pathname of the local bin directory: [/usr/local/bin]: 
   Copying dbhome to /usr/local/bin ...
   Copying oraenv to /usr/local/bin ...
   Copying coraenv to /usr/local/bin ...


Creating /etc/oratab file...
Entries will be added to the /etc/oratab file as needed by
Database Configuration Assistant when a database is created
Finished running generic part of root script.
Now product-specific root actions will be performed.
Relinking oracle with rac_on option
Using configuration parameter file: /u01/app/19.0.0/grid/crs/install/crsconfig_params
The log of current session can be found at:
  /u01/app/oracle/crsdata/ol7-19c-rac1/crsconfig/rootcrs_ol7-19c-rac1_2019-11-19_06-10-11AM.log
2019/11/19 06:10:25 CLSRSC-594: Executing installation step 1 of 19: 'SetupTFA'.
2019/11/19 06:10:25 CLSRSC-594: Executing installation step 2 of 19: 'ValidateEnv'.
2019/11/19 06:10:25 CLSRSC-363: User ignored prerequisites during installation
2019/11/19 06:10:25 CLSRSC-594: Executing installation step 3 of 19: 'CheckFirstNode'.
2019/11/19 06:10:27 CLSRSC-594: Executing installation step 4 of 19: 'GenSiteGUIDs'.
2019/11/19 06:10:29 CLSRSC-594: Executing installation step 5 of 19: 'SetupOSD'.
2019/11/19 06:10:29 CLSRSC-594: Executing installation step 6 of 19: 'CheckCRSConfig'.
2019/11/19 06:10:30 CLSRSC-594: Executing installation step 7 of 19: 'SetupLocalGPNP'.
2019/11/19 06:11:14 CLSRSC-594: Executing installation step 8 of 19: 'CreateRootCert'.
2019/11/19 06:11:22 CLSRSC-594: Executing installation step 9 of 19: 'ConfigOLR'.
2019/11/19 06:11:31 CLSRSC-4002: Successfully installed Oracle Trace File Analyzer (TFA) Collector.
2019/11/19 06:11:46 CLSRSC-594: Executing installation step 10 of 19: 'ConfigCHMOS'.
2019/11/19 06:11:46 CLSRSC-594: Executing installation step 11 of 19: 'CreateOHASD'.
2019/11/19 06:11:55 CLSRSC-594: Executing installation step 12 of 19: 'ConfigOHASD'.
2019/11/19 06:11:55 CLSRSC-330: Adding Clusterware entries to file 'oracle-ohasd.service'
2019/11/19 06:12:20 CLSRSC-594: Executing installation step 13 of 19: 'InstallAFD'.
2019/11/19 06:12:29 CLSRSC-594: Executing installation step 14 of 19: 'InstallACFS'.
2019/11/19 06:12:37 CLSRSC-594: Executing installation step 15 of 19: 'InstallKA'.
2019/11/19 06:12:45 CLSRSC-594: Executing installation step 16 of 19: 'InitConfig'.

ASM has been created and started successfully.

[DBT-30001] Disk groups created successfully. Check /u01/app/oracle/cfgtoollogs/asmca/asmca-191119AM061319.log for details.

2019/11/19 06:14:25 CLSRSC-482: Running command: '/u01/app/19.0.0/grid/bin/ocrconfig -upgrade oracle oinstall'
CRS-4256: Updating the profile
Successful addition of voting disk 82896a1404774fa1bf7404bea502e526.
Successfully replaced voting disk group with +DATA.
CRS-4256: Updating the profile
CRS-4266: Voting file(s) successfully replaced
##  STATE    File Universal Id                File Name Disk group
--  -----    -----------------                --------- ---------
 1. ONLINE   82896a1404774fa1bf7404bea502e526 (/dev/sdb1) [DATA]
Located 1 voting disk(s).
2019/11/19 06:16:33 CLSRSC-594: Executing installation step 17 of 19: 'StartCluster'.
2019/11/19 06:18:09 CLSRSC-343: Successfully started Oracle Clusterware stack
2019/11/19 06:18:09 CLSRSC-594: Executing installation step 18 of 19: 'ConfigNode'.
2019/11/19 06:22:17 CLSRSC-594: Executing installation step 19 of 19: 'PostConfig'.
2019/11/19 06:23:13 CLSRSC-325: Configure Oracle Grid Infrastructure for a Cluster ... succeeded
[root@ol7-19c-rac1 ~]# su - oracle
Last login: Tue Nov 19 06:23:11 EST 2019 on pts/1
ss[oracle@ol7-19c-rac1 ~]$ ssh ol7-19c-rac2
Last login: Tue Nov 19 06:09:27 2019 from ol7-19c-rac1
[oracle@ol7-19c-rac2 ~]$ su - 
Password: 
Last login: Tue Nov 19 06:09:30 EST 2019 on pts/1
[root@ol7-19c-rac2 ~]# /u01/app/19.0.0/grid/root.sh
Performing root user operation.

The following environment variables are set as:
    ORACLE_OWNER= oracle
    ORACLE_HOME=  /u01/app/19.0.0/grid

Enter the full pathname of the local bin directory: [/usr/local/bin]: 
   Copying dbhome to /usr/local/bin ...
   Copying oraenv to /usr/local/bin ...
   Copying coraenv to /usr/local/bin ...


Creating /etc/oratab file...
Entries will be added to the /etc/oratab file as needed by
Database Configuration Assistant when a database is created
Finished running generic part of root script.
Now product-specific root actions will be performed.
Relinking oracle with rac_on option
Using configuration parameter file: /u01/app/19.0.0/grid/crs/install/crsconfig_params
The log of current session can be found at:
  /u01/app/oracle/crsdata/ol7-19c-rac2/crsconfig/rootcrs_ol7-19c-rac2_2019-11-19_06-24-56AM.log
2019/11/19 06:25:07 CLSRSC-594: Executing installation step 1 of 19: 'SetupTFA'.
2019/11/19 06:25:07 CLSRSC-594: Executing installation step 2 of 19: 'ValidateEnv'.
2019/11/19 06:25:07 CLSRSC-363: User ignored prerequisites during installation
2019/11/19 06:25:07 CLSRSC-594: Executing installation step 3 of 19: 'CheckFirstNode'.
2019/11/19 06:25:09 CLSRSC-594: Executing installation step 4 of 19: 'GenSiteGUIDs'.
2019/11/19 06:25:10 CLSRSC-594: Executing installation step 5 of 19: 'SetupOSD'.
2019/11/19 06:25:10 CLSRSC-594: Executing installation step 6 of 19: 'CheckCRSConfig'.
2019/11/19 06:25:11 CLSRSC-594: Executing installation step 7 of 19: 'SetupLocalGPNP'.
2019/11/19 06:25:14 CLSRSC-594: Executing installation step 8 of 19: 'CreateRootCert'.
2019/11/19 06:25:14 CLSRSC-594: Executing installation step 9 of 19: 'ConfigOLR'.
2019/11/19 06:25:33 CLSRSC-594: Executing installation step 10 of 19: 'ConfigCHMOS'.
2019/11/19 06:25:34 CLSRSC-594: Executing installation step 11 of 19: 'CreateOHASD'.
2019/11/19 06:25:48 CLSRSC-594: Executing installation step 12 of 19: 'ConfigOHASD'.
2019/11/19 06:25:51 CLSRSC-330: Adding Clusterware entries to file 'oracle-ohasd.service'
2019/11/19 06:26:36 CLSRSC-4002: Successfully installed Oracle Trace File Analyzer (TFA) Collector.
2019/11/19 06:26:41 CLSRSC-594: Executing installation step 13 of 19: 'InstallAFD'.
2019/11/19 06:26:44 CLSRSC-594: Executing installation step 14 of 19: 'InstallACFS'.
2019/11/19 06:26:48 CLSRSC-594: Executing installation step 15 of 19: 'InstallKA'.
2019/11/19 06:26:52 CLSRSC-594: Executing installation step 16 of 19: 'InitConfig'.
2019/11/19 06:27:12 CLSRSC-594: Executing installation step 17 of 19: 'StartCluster'.
2019/11/19 06:28:23 CLSRSC-343: Successfully started Oracle Clusterware stack
2019/11/19 06:28:23 CLSRSC-594: Executing installation step 18 of 19: 'ConfigNode'.
2019/11/19 06:29:24 CLSRSC-594: Executing installation step 19 of 19: 'PostConfig'.
2019/11/19 06:29:56 CLSRSC-325: Configure Oracle Grid Infrastructure for a Cluster ... succeeded
[root@ol7-19c-rac2 ~]#
```

脚本执行完成后，返回“ol7-19c-rac1”上的“执行配置脚本”屏幕，然后单击“确定”按钮。

等待配置助手完成。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-193235@2x.jpg "Install the Grid Infrastructure")
如果任何配置步骤失败，则应检查指定的日志，以查看错误是否为show-stopper。

如果您没有任何显示停止符，则可以通过单击“下一步”按钮来忽略错误。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-193606@2x.jpg "Install the Grid Infrastructure")
单击“关闭”按钮退出安装程序。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-193620@2x.jpg "Install the Grid Infrastructure")
网格基础结构安装现已完成。我们可以使用以下命令检查安装状态。
```console
[root@ol7-19c-rac1 ~]# /u01/app/19.0.0/grid/bin/crsctl stat res -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.LISTENER.lsnr
               ONLINE  ONLINE       ol7-19c-rac1             STABLE
               ONLINE  ONLINE       ol7-19c-rac2             STABLE
ora.chad
               ONLINE  ONLINE       ol7-19c-rac1             STABLE
               ONLINE  ONLINE       ol7-19c-rac2             STABLE
ora.net1.network
               ONLINE  ONLINE       ol7-19c-rac1             STABLE
               ONLINE  ONLINE       ol7-19c-rac2             STABLE
ora.ons
               ONLINE  ONLINE       ol7-19c-rac1             STABLE
               ONLINE  ONLINE       ol7-19c-rac2             STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.ASMNET1LSNR_ASM.lsnr(ora.asmgroup)
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
      2        ONLINE  ONLINE       ol7-19c-rac2             STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.DATA.dg(ora.asmgroup)
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
      2        ONLINE  ONLINE       ol7-19c-rac2             STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.LISTENER_SCAN1.lsnr
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
ora.asm(ora.asmgroup)
      1        ONLINE  ONLINE       ol7-19c-rac1             Started,STABLE
      2        ONLINE  ONLINE       ol7-19c-rac2             Started,STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.asmnet1.asmnetwork(ora.asmgroup)
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
      2        ONLINE  ONLINE       ol7-19c-rac2             STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.cvu
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
ora.ol7-19c-rac1.vip
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
ora.ol7-19c-rac2.vip
      1        ONLINE  ONLINE       ol7-19c-rac2             STABLE
ora.qosmserver
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
ora.scan1.vip
      1        ONLINE  ONLINE       ol7-19c-rac1             STABLE
--------------------------------------------------------------------------------
[root@ol7-19c-rac1 ~]#
```

此时，最好关闭两个VM并进行快照。 切记要在主机上重新压缩ASM磁盘，如果恢复到网格后快照，则需要将其还原。
```console
$ cd /u04/VirtualBox/ol7-19c-rac
$ zip PostGrid.zip *.vdi
```

## 安装数据库软件
确保启动了“ol7-19c-rac1”和“ol7-19c-rac2”虚拟机，然后以oracle用户身份登录到“ol7-19c-rac1”，并将数据库软件解压缩到第一个节点上的目标目录。不要在第二个节点上执行此操作。
```console
unzip -d /u01/app/oracle/product/19.0.0/db_1 V982063-01.zip
```
然后启动Oracle安装程序。如前所述，使用“crsctl stat res -t”检查所有服务是否正常运行。
```console
$ db_env
$ cd $ORACLE_HOME
$ ./runInstaller
```
选择“仅设置软件”选项，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-194047@2x.jpg "Install the Database Software")
通过单击“下一步”按钮接受“Oracle Real Application Clusters数据库安装”选项。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-194138@2x.jpg "Install the Database Software")
确保两个节点都被选中，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-194210@2x.jpg "Install the Database Software")
选择“企业版”选项，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-194630@2x.jpg "Install the Database Software")
输入“/u01/app/oracle”作为Oracle base，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-194639@2x.jpg "Install the Database Software")
选择所需的操作系统组，然后单击“下一步”按钮。 在这种情况下，我们仅使用“dba”组。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-194725@2x.jpg "Install the Database Software")
接受默认选项，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-194734@2x.jpg "Install the Database Software")
等待先决条件检查完成。 如果有任何问题，请单击“修复并再次检查”按钮，或选中“全部忽略”复选框，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-195031@2x.jpg "Install the Database Software")

![19c_RAC_install](./19c_RAC_install/Jietu20191119-195100@2x.jpg "Install the Database Software")
如果您对摘要信息感到满意，请单击“安装”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-195114@2x.jpg "Install the Database Software")
等待安装。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-195943@2x.jpg "Install the Database Software")
出现提示时，请在每个节点上运行配置脚本。 在每个节点上运行脚本后，单击“确定”按钮。

单击“关闭”按钮退出安装程序。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-200437@2x.jpg "Install the Database Software")
关闭两个VM并拍摄快照。 切记在主机上重新压缩ASM磁盘，如果还原到数据库后快照，则需要恢复该压缩。
```console
$ cd /u04/VirtualBox/ol7-19c-rac
$ zip PostDB.zip *.vdi
```

## 创建一个数据库
确保已启动“ol7-19c-rac1”和“ol7-19c-rac2”虚拟机，然后以oracle用户身份登录“ol7-19c-rac1”并启动数据库创建辅助（DBCA）。
```console
$ db_env
$ dbca
```
选择“创建数据库”选项，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-200513@2x.jpg "Create a Database")
选择“高级模式”选项。 点击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-200545@2x.jpg "Create a Database")
检查“自定义数据库”选项，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-200603@2x.jpg "Create a Database")
确保两个节点都被选中，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-200615@2x.jpg "Create a Database")
输入容器数据库名称（cdbrac），可插入数据库名称（pdb）和管理员密码。 点击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-200740@2x.jpg "Create a Database")
接受默认值，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-200818@2x.jpg "Create a Database")
接受默认值，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-200859@2x.jpg "Create a Database")
接受默认值，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-200942@2x.jpg "Create a Database")
接受默认值，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-201022@2x.jpg "Create a Database")

![19c_RAC_install](./19c_RAC_install/Jietu20191119-201044@2x.jpg "Create a Database")
取消选择CVU和EM选项，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-201119@2x.jpg "Create a Database")
输入dba密码，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-201137@2x.jpg "Create a Database")

![19c_RAC_install](./19c_RAC_install/Jietu20191119-201149@2x.jpg "Create a Database")

![19c_RAC_install](./19c_RAC_install/Jietu20191119-201216@2x.jpg "Create a Database")

![19c_RAC_install](./19c_RAC_install/Jietu20191119-201309@2x.jpg "Create a Database")
选择“全部忽略”，然后单击“下一步”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-201424@2x.jpg "Create a Database")
如果您对摘要信息感到满意，请单击“完成”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-201435@2x.jpg "Create a Database")
等待数据库创建完成。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-202201@2x.jpg "Create a Database")
如果要修改密码，请单击“密码管理”按钮。 完成后，单击“关闭”按钮。
![19c_RAC_install](./19c_RAC_install/Jietu20191119-233237@2x.jpg "Create a Database")
RAC数据库创建现已完成。

## 检查RAC的状态
有几种方法可以检查RAC的状态。 srvctl实用程序显示RAC数据库的当前配置和状态。
```console
[oracle@ol7-19c-rac1 cdbrac]$ grid_env
[oracle@ol7-19c-rac1 cdbrac]$ srvctl config database -d cdbrac
Database unique name: cdbrac
Database name: cdbrac
Oracle home: /u01/app/oracle/product/19.0.0/db_1
Oracle user: oracle
Spfile: +DATA/CDBRAC/PARAMETERFILE/spfile.275.1024741213
Password file: +DATA/CDBRAC/PASSWORD/pwdcdbrac.258.1024730137
Domain: 
Start options: open
Stop options: immediate
Database role: PRIMARY
Management policy: AUTOMATIC
Server pools: 
Disk Groups: DATA
Mount point paths: 
Services: 
Type: RAC
Start concurrency: 
Stop concurrency: 
OSDBA group: dba
OSOPER group: 
Database instances: cdbrac1,cdbrac2
Configured nodes: ol7-19c-rac1,ol7-19c-rac2
CSS critical: no
CPU count: 0
Memory target: 0
Maximum memory: 0
Default network number for database services: 
Database is administrator managed
[oracle@ol7-19c-rac1 cdbrac]$ srvctl status database -d cdbrac
Instance cdbrac1 is running on node ol7-19c-rac1
Instance cdbrac2 is running on node ol7-19c-rac2
[oracle@ol7-19c-rac1 cdbrac]$
```
V$ACTIVE_INSTANCES视图还可以显示实例的当前状态。
```console
[oracle@ol7-19c-rac1 cdbrac]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Tue Nov 19 10:34:25 2019
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> SELECT inst_name FROM v$active_instances;

INST_NAME
--------------------------------------------------------------------------------
ol7-19c-rac1:cdbrac1
ol7-19c-rac2:cdbrac2

SQL>
```

## 参考
有关更多信息，请参见：
- [Grid Infrastructure Installation and Upgrade Guide for Linux](https://docs.oracle.com/en/database/oracle/oracle-database/19/cwlin/index.html)  
- [Database Installation Guide for Linux](https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/index.html)
- [Oracle Database 12c Release 2 (12.2) RAC On Oracle Linux 7 Using VirtualBox](https://oracle-base.com/articles/12c/oracle-db-12cr2-rac-installation-on-oracle-linux-7-using-virtualbox)

### 结束

