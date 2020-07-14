---
title: Find Device by Mac
tags:
  - network
  - tool
date: 2020-07-14 21:55:01
---


## The way to find Raspberry pi in LAN.

### 1. Update local arp table by ICMP echo reply (ping)
```
nmap -sP 192.168.1.0/24
```
or
```
ping 192.168.1.255
```

### 2. List ip and mac mapping by tools
```
apt install net-tools
arp -an
```
or
```
ip neighbor
```

## One line:
```
$ nmap -sP 192.168.1.0/24 >/dev/null && arp -an | grep <mac address here> | awk '{print $2}' | sed 's/[()]//g'
```

## reference:
[Nmap-掃瞄主機所開啟的 Port](http://wiki.weithenn.org/cgi-bin/wiki.pl?Nmap-%E6%8E%83%E7%9E%84%E4%B8%BB%E6%A9%9F%E6%89%80%E9%96%8B%E5%95%9F%E7%9A%84_Port)
[Nmap 網路診斷工具基本使用技巧與教學](https://blog.gtwang.org/linux/nmap-command-examples-tutorials/)
[Can I determine the current IP from a known MAC Address?](https://stackoverflow.com/questions/13552881/can-i-determine-the-current-ip-from-a-known-mac-address)

## tools:
ip
nmap
arp


