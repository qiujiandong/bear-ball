---
title: openEuler RISC-V QEMU网络配置
date: 2025-10-25 22:14:44
tags:
  - openEuler
  - QEMU
  - RISC-V
categories:
  - [RISC-V]
  - [openEuler]
---

openEuler的虚拟机镜像使用官网提供的即可, [openEuler RISC-V虚拟机安装文档](https://docs.openeuler.org/zh/docs/25.09/server/installation_upgrade/installation/risc_v_qemu.html)

关于如何连接外网, 网上已经有很多文章介绍了, 比如这些参考链接:

- [QEMU Networking](https://wiki.qemu.org/Documentation/Networking)
- [QEMU Networking NAT](https://wiki.qemu.org/Documentation/Networking/NAT)
- [QEMU Networking Wiki](https://en.wikibooks.org/wiki/QEMU/Networking)
- [openEuler Embedded QEMU Networking](https://embedded.pages.openeuler.org/master/developer_guide/debug/qemu/qemu_start.html#qemu-enable-net)

但这些文章看上去配置起来也都挺麻烦的, 所以我在这里记录一下我的配置方式.

## NAT模式

Network Address Translation, 官方提供的`start_vm.sh`中默认启用的是user mode network backend,
这个模式默认不能用`ping`来测试网络的连通性. 需要在宿主机上做一些额外的设置才行.
但用`curl`来测试网络的连通性是可以的.

这个NAT模式的额外开销比较多, 所以网络的性能也相对较差一些.

通过`qemu-system-riscv`的help帮助信息可以看到有如下的配置选项

```txt
-netdev user,id=str[,ipv4=on|off][,net=addr[/mask]][,host=addr]
         [,ipv6=on|off][,ipv6-net=addr[/int]][,ipv6-host=addr]
         [,restrict=on|off][,hostname=host][,dhcpstart=addr]
         [,dns=addr][,ipv6-dns=addr][,dnssearch=domain][,domainname=domain]
         [,tftp=dir][,tftp-server-name=name][,bootfile=f][,hostfwd=rule][,guestfwd=rule][,smb=dir[,smbserver=addr]]
                configure a user mode network backend with ID 'str',
                its DHCP server and optional services
```

## 桥接模式

桥接模式顾名思义就是用宿主机上的网桥来转发虚拟机上的网络数据包,
但这个需要在宿主机上创建网桥, 然后设置`iptables`的转发. 感觉是有点麻烦的,
不过好在qemu提供来一个`qemu-bridge-helper`来帮助我们做这个工作.

关于网桥, 我发现我之前安装过`libvirt-daemon`, 所以有一个`virbr0`网桥已经有了,

```bash
ip addr show virbr0 && brctl show virbr0
```

```txt
4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:18:ec:ef brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
bridge name     bridge id               STP enabled     interfaces
virbr0          8000.52540018ecef       yes
```

可能是因为我之前装过`gnome-boxes`吧, 所以创建网桥的步骤我这里可以省略,
只要安装过`libvirt-daemon`并启动来`libvirtd`服务应该就有这个网桥.
在启动qemu的时候, 直接指定`-netdev bridge`, `br=virbr0`,
QEMU就会启用`qemu-bridge-helper`来帮助我们做转发工作.

```txt
-netdev bridge,id=str[,br=bridge][,helper=helper]
                configure a host TAP network backend with ID 'str' that is
                connected to a bridge (default=br0)
                using the program 'helper (default=/usr/local/bin/qemu-riscv64/libexec/qemu-bridge-helper)
```

根据QEMU安装位置的不同, 还需要提供一个`bridge.conf`文件

```bash
cat /usr/local/bin/qemu-riscv64/etc/qemu/bridge.conf
allow virbr0
```

然后用的是时候还需要加上管理员权限

```bash
sudo qemu-system-riscv ...
```

用这种bridge模式的话, 网络性能相对更好, 而且也可以用`ping`来测试网络的连通性了.

## 更新软件包

更新软件包的时候可能会遇到这样的问题:

```txt
SSL peer certificate or SSH remote key was not OK ...
```

在更新软件包之前需要先同步一下时间. 一开始可以先手动设置一下时间:

```bash
sudo date -s "2025-10-25 20:30:00"
```

然后在可以连网更新软件包之后再同步一下时间:

```bash
sudo ntpdate ntp.aliyun.com
sudo timedatectl set-timezone Asia/Shanghai
```

也可以按照这里的文档来设置时间, [基础设置](https://docs.openeuler.org/zh/docs/21.03/docs/Administration/%E5%9F%BA%E7%A1%80%E9%85%8D%E7%BD%AE.html)
具体如何更新软件包可以参考[文档](https://docs.openeuler.org/zh/docs/21.03/docs/Administration/%E4%BD%BF%E7%94%A8DNF%E7%AE%A1%E7%90%86%E8%BD%AF%E4%BB%B6%E5%8C%85.html)
