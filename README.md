## 说明

**本openwrt仅适用于x86_64的cpu.**

  |名称 |说明 |
  |:----|:----|
  |IP| 10.5.2.1|
  |用户| root|
  |密码||

> **说明**:构建本openwrt目的是自己使用,没有太多功能比较清爽,仅仅适用我个人使用.

### 构建openwrt或[releases](https://github.com/xYx-c/build-openwrt/releases)下载
- Fork本仓库-> Actions-> Build OpenWrt-> Run workflow

### pve8构建lxc openwrt容器
### pve7.4
# pct create 105 local:vztmpl/openwrt-x86-64-generic-rootfs.tar.gz --rootfs local-lvm:1 --ostype unmanaged --hostname openwrt-LXC --arch amd64 --cores 1 -memory 2048 --swap 2048 bridge=vmbr0,name=eth0
- 进入容器,执行命令:
```
pct create xxx \ # xxx容器序号
local:vztmpl/openwrt-xxx-rootfs.tar.gz \ # 镜像文件
--rootfs local:2 \ # 容器大小
--ostype unmanaged \
--hostname openwrt \
--arch amd64 \
--cores 4 \ # cpu
--memory 1024 \ # 内存
--swap 0  # 交换空间
```

- 配置文件路径
``` shell
vim /etc/pve/lxc/xxx.conf
```

- lxc追加配置
>   **注意**: 网卡配置的type, veth为虚拟网卡, phys为真实网卡
```
onboot: 0 # 是否开机启动 1是 0否
features: nesting=1
lxc.cgroup2.devices.allow: c 108:0 rwm
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.auto: proc:mixed sys:ro cgroup:mixed
lxc.mount.entry: /dev/net/tun dev/net/tun none rw,bind,create=file 0 0
lxc.mount.entry: /dev/ppp dev/ppp none rw,bind,optional,create=file 0 0
lxc.net.0.flags: up 
lxc.net.0.type: veth 
lxc.net.0.link: vmbr0 #虚拟网卡名
lxc.net.0.name: eth0
lxc.net.1.flags: up
lxc.net.1.type: phys
lxc.net.1.link: enp1s0 #真实网卡名
lxc.net.1.name: eth1
```

### 从lxc openwrt中dhcpv6服务获取ipv6
> pve启动后负责拨号的openwrt还未启动无法获取ipv6地址,添加定时任务系统启动3分钟后获取ipv6,每12小时重新尝试获取
- #### 创建dhcpv6.service
``` shell
cat >> /etc/systemd/system/dhcpv6.service << EOF
[Unit]
Description=OpenWrt DHCPv6 Server
After=network.target
[Service]
ExecStart=/usr/sbin/dhclient -6 vmbr0
[Install]
WantedBy=multi-user.target
EOF
```
- #### 创建dhcpv6.timer
``` shell
cat >> /etc/systemd/system/dhcpv6.timer << EOF
[Unit]
Description=OpenWrt DHCPv6 Server
After=network.target
[Timer]
OnBootSec=3min
OnUnitActiveSec=12h
[Install]
WantedBy=multi-user.target
EOF
```
- #### 运行定时任务
``` shell
systemctl daemon-reload
systemctl enable dhcpv6.timer
systemctl start dhcpv6.timer
```

### 网络配置
接口-> lan-> IPV6设置-> 本地IPV6-> **DNS服务器取消勾选**

### AdgHome DNS配置
- 重定向53端口到AdGuardHome
```
127.0.0.1:7874
# 防污染解析、
tls://dns.pub
https://dns.pub/dns-query
tls://dns.alidns.com
https://dns.alidns.com/dns-query
tls://dns.google.com
https://dns.google/dns-query
——————————————————
Bootstrap DNS
——————————————————
119.29.29.29
202.96.134.133
2402:4e00::
2400:3200::1
```

### OpenClash配置
插件设置-> DNS设置-> 停用*本地DNS劫持

#### 服务
  1. OpenClash
  2. AdguardHome
  3. DDNS
  4. ~~SmartDns~~

## 鸣谢

- 感谢[openwrt源码](https://github.com/openwrt/openwrt)
- 感谢[@P3TERX](https://github.com/P3TERX)

> 使用了
>   1. [P3TERX大佬的云编译](https://github.com/P3TERX/Actions-OpenWrt)
>   2. 最后感谢上面使用了但未提及的大佬们

