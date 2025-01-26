---
title: 折腾tailscale
keywords: mikrotik,tailscale
author: devinaille
desc: 在mikrotik里折腾tailscale
date: 2025-01-24
---


## 启用容器
这个自己百度

## 创建docker网桥

```bash
/interface bridge
add comment=docker name=dockers
```

## 创建挂载目录
```
/container mounts
add dst=/var/lib/tailscale name=tailscale src=/tailscale
```

