---
layout: post
title: "怎么设置pxe安装时使用的cobbler的默认ks？"
date: 2017-03-17 00:11:12
author: 伊布
categories: tech
tags: kickstart
cover:  "/assets/instacode.png"
---


在之前的[如何为linux集群批量装机？](http://www.datastart.cn/tech/2016/08/20/kickstart.html)中，提到了可以使用cobbler来安装pxe装机环境，最近在写cobbler安装的脚本，遇到了一个问题：

怎么设置pxe安装时使用的cobbler的默认ks？

之前解决过这个问题，但是时过境迁已经完全忘记了，这里记录下过程。

---

```
cobbler import --arch=x86_64 --path=/mnt/iso --name=CentOS-7
cobbler signature update
cobbler profile edit --name=CentOS-7-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-7.ks
```

脚本里做的主要步骤如上，但是pxe测试的时候，在下面这个界面（别人的图，我的是centos-7）：

![示意图](http://www.unixmen.com/wp-content/uploads/2014/07/CentOS-6.5-PXE-Client-Running-Oracle-VM-VirtualBox_010.png)

如果在这个阶段不用键盘选择的话，会自动进入local流程。

为什么呢？我们重新理一下kickstart的流程。

![pxe boot](http://7xir15.com1.z0.glb.clouddn.com/pxe-boot.png)

简单来说，pxe client先发送dhcp请求，dhcp server除了会offer地址外，还会指定next server，即tftp server的ip地址；pxe client去tftp server先拉取pxelinux.0（一个bootloader），启动pxelinux.0后，再去tftp server拉这次kickstart的配置文件。默认是取tftp server的根目录下pxelinux.cfg目录；具体以cobbler配置来说，配置文件的目录是 /var/lib/tftpboot/pxelinux.cfg。

pxelinux.0是取这个目录的哪个文件呢？有一个顺序：

1. 取pxe client网卡MAC地址同名的文件(加前缀01-)，若无则2；
2. 取dhcp获得的地址对应16进制同名的文件，注意支持掩码，即先按最长匹配，若无则依次忽略最后一位；若全部不匹配，则3；
3. 获取default文件

举例如下：

```
/var/lib/tftpboot/pxelinux.cfg/01-88-99-aa-bb-cc-dd
/var/lib/tftpboot/pxelinux.cfg/C000025B
/var/lib/tftpboot/pxelinux.cfg/C000025
/var/lib/tftpboot/pxelinux.cfg/C00002
/var/lib/tftpboot/pxelinux.cfg/C0000
/var/lib/tftpboot/pxelinux.cfg/C000
/var/lib/tftpboot/pxelinux.cfg/C00
/var/lib/tftpboot/pxelinux.cfg/C0
/var/lib/tftpboot/pxelinux.cfg/C
/var/lib/tftpboot/pxelinux.cfg/default
```

更详细的可以参考 [pxespec.pdf](http://www.pix.net/software/pxeboot/archive/pxespec.pdf.)第二章。

我们这里没有配置按MAC地址或者按IP地址来给特定服务器装机，所以pxelinux.cfg目录下只有一个default文件。来看看里面有啥。

```
DEFAULT menu
PROMPT 0
MENU TITLE Cobbler | http://cobbler.github.io/
TIMEOUT 30
TOTALTIMEOUT 60
ONTIMEOUT local

LABEL local
        MENU LABEL (local)
        MENU DEFAULT
        LOCALBOOT -1

LABEL CentOS-7-x86_64
        kernel /images/CentOS-7-x86_64/vmlinuz
        MENU LABEL CentOS-7-x86_64
        append initrd=/images/CentOS-7-x86_64/initrd.img ksdevice=bootif lang=  kssendmac text  ks=http://192.168.182.99/cblr/svc/op/ks/profile/CentOS-7-x86_64
        ipappend 2
```

显然这就是我们在kickstart上看到的装机界面。控制超时选哪个LABEL的选项是`ONTIMEOUT local`，显然超时后会走local，即本地硬盘启动。

如何来控制这个选项的值呢？我们知道这些配置其实都是cobbler sync的时候生成的，不妨看看/etc/cobbler/pxe/pxedefault.template。

```
DEFAULT menu
PROMPT 0
MENU TITLE Cobbler | http://cobbler.github.io/
TIMEOUT 30
TOTALTIMEOUT 60
ONTIMEOUT $pxe_timeout_profile

LABEL local
        MENU LABEL (local)
        MENU DEFAULT
        LOCALBOOT -1

$pxe_menu_items

MENU end
```

哦，cobbler会替换 `$pxe_timeout_profile`。它是怎么做的呢？不妨看看cobbler源码。

```python
def make_actual_pxe_menu(self):
    """
    Generates both pxe and grub boot menus.
    """
    # only do this if there is NOT a system named default.
    default = self.systems.find(name="default")

    if default is None:
        timeout_action = "local"
    else:
        timeout_action = default.profile

    menu_items = self.get_menu_items()

    # Write the PXE menu:
    metadata = { "pxe_menu_items" : menu_items['pxe'], "pxe_timeout_profile" : timeout_action}
    outfile = os.path.join(self.bootloc, "pxelinux.cfg", "default")
    template_src = open(os.path.join(self.settings.pxe_template_dir,"pxedefault.template"))
    template_data = template_src.read()
    self.templar.render(template_data, metadata, outfile, None)
    template_src.close()
```

pxe_timeout_profile是由timeout_action来控制，而timeout_action是看有没有default system来决定的，若无则timeout_action就是local了。不妨`cobbler system list`看看当前cobbler有哪些system，显然没有，因为前面只创建了distro和profile。

补充一下，cobbler的三级概念：

- distro， 即“操作系统”；如上我们用cobbler import导入iso时，会自动生成一个distro。当然也可以用cobler distro add，不推荐就是了。
- profile，即“操作系统” + “具体的系统安装参数”；系统安装参数，通常就是指ks文件
- system，即具体的实例，除了default外，还可以具体指定到某一台服务器

回到我们的问题，~~没有枪没有炮~~没有default system，创建一个好了：

```
cobbler system add --name=default --profile=CentOS-7-x86_64
```

重新cobbler sync下刷配置，再来看看tftp下的pxelinux.cfg/default:

```
# cat /var/lib/tftpboot/pxelinux.cfg/default
DEFAULT menu
PROMPT 0
MENU TITLE Cobbler | http://cobbler.github.io/
TIMEOUT 30
TOTALTIMEOUT 60
ONTIMEOUT CentOS-7-x86_64

LABEL local
        MENU LABEL (local)
        MENU DEFAULT
        LOCALBOOT -1

LABEL CentOS-7-x86_64
        kernel /images/CentOS-7-x86_64/vmlinuz
        MENU LABEL CentOS-7-x86_64
        append initrd=/images/CentOS-7-x86_64/initrd.img ksdevice=bootif lang=  kssendmac text  ks=http://192.168.182.99/cblr/svc/op/ks/profile/CentOS-7-x86_64
        ipappend 2



MENU end
```

ONTIMEOUT已经是CentOS-7-x86-64这个LABEL，而这个label指定的ks内容就是我们前面加上去的ks。

完事，收工。

---
