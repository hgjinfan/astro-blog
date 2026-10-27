---
title: 一次 docker pull 把 1G 小鸡压到假死
description: 给 Uptime Kuma 升级前随手拉了一下新镜像，1 核 1G、无 swap 的甲骨文小鸡被几百 MB 的下载和解压顶满，负载冲到 27，SSH 和 HTTP 集体超时。服务没死，是被饿死的——记录定位过程和三个改法。
pubDatetime: 2026-10-27T09:40:00+08:00
tags:
  - docker
  - vps
---

## 背景

首尔的甲骨文免费小鸡 web-e2（VM.Standard.E2.1.Micro，1 OCPU、1 GB 内存、无 swap）上跑着两个容器：个人主页 Perch 和 Uptime Kuma。Kuma 在跑 2.5.6，GitHub 最新是 2.5.8，`:2` 是滚动标签，升级就是拉新镜像换容器。动手之前先 `docker pull` 验一下新镜像——事故就从这一步开始。

## 假死现场

pull 开始没几分钟，SSH 先断了，重连握手超时；公网访问主页和 Kuma 全部超时，一度以为机器没了。

从另一台机侧测才看清状态：到这台机的 22、3001 端口 TCP 都能连上，但 HTTP 始终不回话。内核还活着、还在应答连接，用户态进程却集体卡住——不是宕机，是 stall。

等 SSH 勉强能进，`uptime` 给出 load average 27（1 核机器），内存 available 只剩 200 多 MB，dmesg 里没有 OOM 记录。Perch 容器被 Docker 标成了 unhealthy，但本机直连 Node 端口 0.1 秒就返回 200：它没病，健康检查的探针只是在卡顿期间全部超过了 5 秒超时，第一条超时的时间戳正好落在 pull 开始的那一分钟。而 pull 进程本身已经随 SSH 断线死掉，新镜像根本没装上——纯粹白挨了一顿打。

## 原因

滚动标签的 pull 不是“检查更新”，是把几百 MB 的镜像层全量下载下来再逐层解压。在 1 核 CPU 上，下载、校验、解压全挤在一起，CPU 和 IO 同时饱和；1 GB 内存还没有 swap 兜底，整台机没有任何缓冲余量，所有进程一起被饿住。负载回落后服务自己就恢复了，全程没丢数据。

## 改法

1. 小机器上 `docker pull` 是重操作，当维护动作对待：先看负载，别在忙的时候拉大镜像。
2. 拉取要在机器上 detached 跑（nohup / systemd-run），别让 SSH 断线决定进程生死。
3. “有没有新版本”不需要 pull：拿 registry 上的 manifest digest 和本地镜像的 RepoDigests 对比就行，零下载、零解压。每周那个镜像更新检查也准备改成这个方式，免得它自己把机器压一遍。

Kuma 的升级改到负载平稳后，detached 拉完再换容器。小机器过日子，得精打细算。
