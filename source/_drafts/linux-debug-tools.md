---
title: linux-debug-tools
tags:
  - linux
  - tool
---

## reference:
[ProcDump](https://www.facebook.com/champ.yen/posts/10219757418979212)
先 bookmark
微軟開源的 debug tool
主要是部分的 Sysinternals 工具已移植到 Linux 上
本文中介紹的是在 Sysinternals 最受到歡迎的工具之一 ProcDump
能夠將 process 的記憶體狀態 dump 下來再透過 gdb / gcore 分析
[Debug Linux using ProcDump](https://opensource.com/article/20/7/procdump-linux?fbclid=IwAR2AmFzCBBCsZCakdSjTjlv8dolu7Fl04kPzy4AzeThUcAJRI0tIDG96AYE)

[ProcMon](https://www.facebook.com/champ.yen/posts/10219757775628128)
才分享完 ProcDump 接著 Microsoft 最新開源了同在 Sysinternal 中的 ProcMon
ProcMon 是以 C++ 撰寫的開源工具, 用來監控 Linux 上的 process, 並且能夠簡單的追蹤 system call 的活動. ProcMon 以 MIT License 釋出.
[Microsoft Releases Its Own Open-Source Process Monitor For Linux](https://www.phoronix.com/scan.php?page=news_item&px=Microsoft-ProcMon-For-Linux&fbclid=IwAR2UoneM2tsqxVNbH_uoPkHK4gpFIr7QyyYGPXxyr8Eiiq_Bsg62pISVvuY)





