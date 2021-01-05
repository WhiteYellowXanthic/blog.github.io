---
layout: post
title:  "Avoid the VPN client update the primary route record."
date:   2019-07-03 00:00:00
categories: Batch
---

In the company, we have to access the remote servers through the VPN, but the VPN client will update the primary route record.

```bash
IPv4 路由表
===========================================================================
活动路由:
网络目标         网络掩码          网关            接口               跃点数
0.0.0.0          0.0.0.0           192.168.30.1    192.168.30.11      2
```

I haven't found any options to disable it, so I wrote the bash script to monitor the route table, then rewrite it.

```bash
@ECHO OFF

SET VPN_GATEWAY=192.168.30.1
SET WAN_GATEWAY=192.168.2.1

:LOOP
    ROUTE PRINT -4 0.0.0.0 | FINDSTR %VPN_GATEWAY% > NUL && (
        ROUTE DELETE 0.0.0.0 MASK 0.0.0.0 %VPN_GATEWAY% > NUL
        ROUTE PRINT -4 0.0.0.0 | FINDSTR %WAN_GATEWAY% > NUL || ROUTE ADD 0.0.0.0 MASK 0.0.0.0 %WAN_GATEWAY% > NUL
    )

    TIMEOUT /T 3 /NOBREAK > NUL

GOTO LOOP
```

Then, create a schedule for it. Start it when the computer power on.