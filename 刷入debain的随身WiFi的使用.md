# 刷入debain的随身WiFi的使用

相关包链接：通过网盘分享的文件：随身WiFi
链接: https://pan.baidu.com/s/14-llhO2B2FKsiO7PGF0tcg?pwd=yxb4 提取码: yxb4

板子上电后wlan0 是 **WiFi STA 客户端模式**（连外面路由器），不是 AP 热点模式。 如果想切换成热点（让手机连棒子的 WiFi），就需要用 nmcli 新建 AP 连接，将usb作为共享网口，板子的wlan0口作为ap结点，使其成为能用其他设备连接的路由器设备。

# 一.**添加ap热点配置**

## 方案 1：nmcli + NetworkManager（推荐，当前系统适用）

### 1. 创建 AP 热点配置

```
nmcli connection add type wifi ifname wlan0 con-name openstick-ap autoconnect no \
wifi.mode ap \
wifi.ssid "OpenStick-WiFi" \
wifi-sec.key-mgmt wpa-psk \
wifi-sec.psk "12345678" \
ipv4.method shared
```

参数说明

- `con-name openstick-ap`：连接名，随便写
- `wifi.ssid "OpenStick-WiFi"`：手机搜到的 WiFi 名字，自行修改
- `wifi-sec.psk "12345678"`：热点密码，至少 8 位
- `ipv4.method shared`：自带 DHCP，手机连上自动分配 IP，开启 NAT 转发

### 2. 启用热点

```
nmcli connection up openstick-ap
```

> ⚠️ 注意：如果 wlan0 当前还连着路由器（STA 模式），**要先断开原有 WiFi 连接**，一个无线网卡不能同时做 STA+AP：

```
# 查看现有wifi连接
nmcli connection show
# 断开原来的STA连接，替换成你的旧连接名称
nmcli connection down "旧WiFi名称"
```

### 3. 查看状态

```
nmcli dev status
iw dev wlan0 info
```

手机搜索 `OpenStick-WiFi`，输入密码连接即可。

### 4. 开机自动启动热点（可选）

```
nmcli connection modify openstick-ap autoconnect yes
```

### 停止热点

```
nmcli connection down openstick-ap
```



# 二.删除 AP 热点配置（openstick-ap）

## 1. 先停止热点

```
nmcli connection down openstick-ap
```

## 2. 删除这条连接配置（永久删掉）

```
nmcli connection delete openstick-ap
```

## 3. 验证是否删除成功

```
nmcli connection show
```

输出列表里看不到 `openstick-ap` 就代表删除完成。

## 4. 重置 wlan0 网卡（清理残留状态）

```
nmcli device disconnect wlan0
ip link set wlan0 down
```

> 现在 wlan0 回到空白状态，你后面二选一：
>
> - 方案 A：重新创建 NM 热点（ipv4.method shared）
> - 方案 B：hostapd+dnsmasq 独立热点

## 可选：查看 wlan0 当前模式

```
iw dev wlan0 info
```

删除后应该变成 `type managed`（STA 客户端模式），不再是 AP。

正常 USB 共享网络逻辑： 电脑 Windows 是网关（WindowsIP = 网关），OpenStick 棒子作为客户端。

# 三.Windows 开启 ICS（Internet 连接共享）完整步骤

> ⚠️重要前提：**先把电脑 WLAN 连上 WiFi，电脑本身能访问外网**，再做下面操作！

1. 打开网络适配器面板 按下 `Win+R`，输入 `ncpa.cpl`，回车，弹出【网络连接】窗口，就是你截图里这个页面。
2. **右键【能上网的网卡】（你的 WLAN 无线网卡）**

> ❗注意：**不要点以太网 2（远程 NDIS）！！** WLAN 是电脑用来上外网的网卡，我们把 WLAN 的网络共享给 OpenStick 的远程 NDIS（以太网 2）。 右键 WLAN → **属性**，切换到顶部【共享】选项卡。

1. 勾选共享选项

- ✅ 勾选：**允许其他网络用户通过此计算机的 Internet 连接来连接**
- 下方【家庭网络连接】下拉菜单：选择 **以太网 2（远程 NDIS 兼容设备）**
- 下方那个 “允许其他网络用户控制或禁用共享的 Internet 连接”**不要勾选**。

1. 点击【确定】保存

> 弹出提示框：开启 ICS 会把以太网 2（NDIS 网卡）自动改成`192.168.137.1`网段，直接点 “是” 确认。

1. 重新插拔 OpenStick 的 USB 线

> 开启 ICS 后网段发生变化，必须重插 USB，让 RNDIS 网卡重新协商，OpenStick 的 usb0 会自动获取 IP、网关、DNS。



硬件场景：OpenStick，USB RNDIS 虚拟网卡 usb0，**固件硬写死 usb0=192.168.68.1，没有 udhcpc，没有 ip 命令**。Windows 10/11（客户端系统，不是服务器）有两个硬限制：

1. ICS（Windows 自带连接共享）
   - 一旦开启，共享网卡会被强制锁定网段：固定`192.168.137.1/24`，**不能改**。
   - Windows 客户端不能直接添加 “跨网段回包路由”，想用 192.168.68.1 访问 192.168.137.1，`route add`写法会报参数错误，这个是 Windows 客户端系统本身的限制，不是命令写错。

## 方案 1：【最简单，最稳】修改板子的网段适配 ICS（板子改 IP，PC 不动）

> 思路：板子 usb0 不用 192.168.68.1，改成 192.168.137.x，直接匹配 ICS 默认网段，**完全不用 PC 添加任何静态路由、不用 NAT 工具**，避开 Windows 所有路由坑。 板子没有 udhcpc，那我们**手动静态配置 usb0 为 192.168.137.x**，不用 DHCP。

PC 侧：

1. WLAN 网卡开启 ICS 共享，共享给以太网 2，以太网 2 自动变成`192.168.137.1`，到此 PC 配置结束，**不需要任何 route 命令**。

板子 OpenStick（adb root shell，只用 ifconfig+route）

```
# 设置usb0静态IP，和ICS同网段
ifconfig usb0 192.168.137.100 netmask 255.255.255.0 up
# 默认网关就是ICS网关
route add default gw 192.168.137.1 usb0
# DNS
echo "nameserver 114.114.114.114" > /etc/resolv.conf
```

测试：

```
ping 192.168.137.1
ping 8.8.8.8
```

✅ 优点：完全避开 Windows 路由报错、不需要第三方软件，是 Windows ICS 设计的原生用法，成功率最高。 ⚠️ 唯一缺点：OpenStick 重启后，gt-usb-gadget 脚本会自动把 usb0 改回 192.168.68.1。每次重启板子后，**重新执行上面 3 行板子命令即可临时生效**。

**RNDIS USB gadget 虚拟网卡的特殊问题**： 虽然网卡配置了 IP，但是板子的内核 ARP 无法到达`192.168.137.1`（PC 的以太网 2），内核认为这个网关不可访问，拒绝添加默认路由。

> 本质：板子不知道怎么把 ARP 包发给 PC 的 192.168.137.1，二层不通。

## 第一步：先测试板子能不能 ping 到 PC 网关

```
ping 192.168.137.1
```

大概率直接 ping 不通。 原因：**Windows ICS 开启后，RNDIS 虚拟网卡的双向 ARP / 二层连通性有问题**。

## 临时解决办法：先添加一条主机路由，单独告诉板子怎么到达 192.168.137.1

在板子 adb shell 里依次执行：

```
# 增加主机路由：单独把192.168.137.1这条主机路由从usb0网卡走
route add 192.168.137.1 usb0

# 现在再添加默认网关
route add default gw 192.168.137.1 usb0
```

### 完整顺序

```
ifconfig usb0 192.168.137.100 netmask 255.255.255.0 up
route add 192.168.137.1 usb0
route add default gw 192.168.137.1 usb0
echo "nameserver 114.114.114.114" > /etc/resolv.conf
```

执行完之后：

```
route -n
ping 192.168.137.1
```

### 原理

对于一些虚拟网卡（RNDIS），Linux 内核不会自动生成同网段主机路由。 手动添加单条主机路由`192.168.137.1 dev usb0`，告诉内核：访问这个 IP 直接走 usb0 虚拟网卡，**不需要 ARP 自动探测**，之后才能添加默认网关。

> 重点：
>
> 1. 如果`route add 192.168.137.1 usb0`成功，再添加默认网关一般就不会报 Network unreachable。
> 2. 如果 ping 192.168.137.1 还是不通，问题就不在板子路由，而是 PC 端：Windows 防火墙拦截了 RNDIS 网卡的 ARP/ICMP，或者 ICS 共享没有正常启用。

你直接执行上面两行 route 命令，把结果贴给我。

> 补充： `route add 192.168.137.1 usb0` 是**主机路由**，不是网段路由，意思：访问 192.168.137.1，从 usb0 网卡出去。 这条命令不需要网关，只指定网卡。这是 RNDIS 虚拟网卡场景下非常经典的修复手段。



# 总结

### PC 侧：

WLAN 网卡开启 ICS 共享，共享给以太网 2，以太网 2 自动变成`192.168.137.1`，到此 PC 配置结束。

注意：只有当以太网 2 变成`192.168.137.1`时，才代表正确开启ICS 共享，本地ICS 共享开启的过程中遇到了以太网 2 没有自动变成`192.168.137.1`的情况。可能的原因是dhcp动态分配ip的过程中ip未被释放（猜测，未证实），解决办法是过段时间重新开启 ICS 共享。

### 板子

简单复盘下这套工作流程，方便后面重启板子快速恢复：

```
#启用热点
nmcli connection up openstick-ap
```

```
#每次重启OpenStick后，依次执行这4条
ifconfig usb0 192.168.137.100 netmask 255.255.255.0 up
route add 192.168.137.1 usb0
route add default gw 192.168.137.1 usb0
echo "nameserver 114.114.114.114" > /etc/resolv.conf
```

执行完，就可以：

```
ping 192.168.137.1
ping 8.8.8.8
ping www.baidu.com
```

> 小提醒：板子重启后，`gt-usb-gadget`会自动把 usb0 重置回 192.168.68.1，所以每次重启板子都要重新跑一遍上面命令。PC 这边只要保持 WLAN 网卡的 ICS 共享打开就不用反复配置。