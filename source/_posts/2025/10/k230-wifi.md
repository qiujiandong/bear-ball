---
title: K230设置自动连接wifi
date: 2025-10-19 20:43:42
tags:
  - wifi
  - K230
  - Embedded
categories:
  - Embedded
---

今天按照官方的[教程](https://www.kendryte.com/k230_linux/dev/zh/01_software/K230_linux_WiFi%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97.html)
设置了一下K230开发板的wifi密码. 感觉流程可以改进一些, 比如开机自动连wifi.

---

第一步, 启用无线网卡. `ifconfig` 或者 `ip` 命令都可以用, `ifconfig` 是相对较老的命令

```bash
# ifconfig wlan0 up
ip link set wlan0 up
```

第二步, 启动`wpa_supplicant`:

```bash
wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant.conf
```

`-B`的作用是后台运行, `-i`指定无线网卡, `-c`指定配置文件.
和wifi相关的配置都可以在配置文件中写好.

所以在第二步前, 需要先准备一下配置文件. 嘉楠提供了默认的配置文件:

```txt
ctrl_interface=/var/run/wpa_supplicant
ap_scan=1

network={
  key_mgmt=NONE
}
```

这里没有设置wifi的名称和密码, 只能用来连开放的无线网络.
如果要设置wifi的名称和密码的话, 可以先:

```bash
wpa_passphrase wifi_test 12345678
network={
        ssid="wifi_test"
        #psk="12345678"
        psk=5c86769b524f416c47ece4465c526cf24e20ce3e565618f4a081130ae22402cc
}
```

用`wpa_passphrase`生成加密的密码, 这样可以避免wifi密码泄漏.

接着写配置文件的时候可以写多个wifi名称和密码, 并且可以指定优先级,
以应多多种场景:

``` txt
ctrl_interface=/var/run/wpa_supplicant
ap_scan=1

network={
    ssid="HomeWiFi"
    psk="home1234"
    priority=10
}

network={
    ssid="OfficeWiFi"
    psk="work5678"
    priority=5
}

network={
    ssid="MyPhone"
    psk="87654321"
    priority=1
}
```

有了`wpa_supplicant`的配置之后, 还需要启动`udhcpc`来获取ip地址:

```bash
udhcpc -i wlan0 -q
```

---

最后, 这整个流程也都可以写在一个脚本里, 比如`/etc/init.d/wifi_connect.sh`,
然后在系统启动的过程中调用这个脚本, 以实现开机自动启动:

```bash
#!/bin/sh
WLAN_IFACE="wlan0"
CONF_FILE="/etc/wpa_supplicant.conf"
ip link set $WLAN_IFACE up
wpa_supplicant -B -i $WLAN_IFACE -c $CONF_FILE
sleep 3
udhcpc -i $WLAN_IFACE -q
```

然后在`/etc/init.d/rcS` 中的最后调用这个脚本:

```bash
/etc/init.d/wifi_connect.sh &
```

这样系统启动的时候就会自动连wifi啦.
