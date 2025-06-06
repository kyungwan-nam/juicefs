---
title: 元数据引擎性能测试
sidebar_position: 6
slug: /metadata_engines_benchmark
description: 本文采用亚马逊云的真实环境，介绍如何对 JuiceFS 的各种元数据引擎性能进行测试和评估。
---

首先展示结论：

- 对于纯元数据操作，MySQL 耗时约为 Redis 的 2～4 倍；TiKV 性能与 MySQL 接近，大部分场景下略优于 MySQL；etcd 的耗时约为 TiKV 的 1.5 倍
- 对于小 IO（～100 KiB）压力，使用 MySQL 引擎的操作总耗时大约是使用 Redis 引擎总耗时的 1～3 倍；TiKV 和 etcd 的耗时与 MySQL 接近
- 对于大 IO（～4 MiB）压力，使用不同元数据引擎的总耗时未见明显差异（此时对象存储成为瓶颈）

:::note 注意

1. Redis 可以通过将 `appendfsync` 配置项由 `always` 改为 `everysec`，牺牲少量可靠性来换取一定的性能提升。更多信息可参见[这里](https://redis.io/docs/manual/persistence)。
2. 测试中 Redis 和 MySQL 数据均仅在本地存储单副本，TiKV 和 etcd 数据会在三个节点间通过 Raft 协议存储三副本。

:::

以下提供了测试的具体细节。这些测试都运行在相同的对象存储（用来存放数据）、客户端和元数据节点上，只有元数据引擎不同。

## 测试环境

### JuiceFS 版本

1.1.0-beta1+2023-06-08.5ef17ba0

### 对象存储

Amazon S3

### 客户端节点

- Amazon c5.xlarge：4 vCPUs，8 GiB 内存，最高 10 Gigabit 网络
- Ubuntu 20.04.1 LTS

### 元数据节点

- Amazon c5d.xlarge：4 vCPUs，8 GiB 内存，最高 10 Gigabit 网络，100 GB SSD（为元数据引擎提供本地存储）
- Ubuntu 20.04.1 LTS
- SSD 数据盘被格式化为 ext4 文件系统并挂载到 `/data` 目录

### 元数据引擎

#### Redis

- 版本：[7.0.9](https://download.redis.io/releases/redis-7.0.9.tar.gz)
- 配置：
  - `appendonly`：`yes`
  - `appendfsync`：分别测试了 `always` 和 `everysec`
  - `dir`：`/data/redis`

#### MySQL

- 版本：8.0.25
- `/var/lib/mysql` 目录被绑定挂载到 `/data/mysql`

#### PostgreSQL

- 版本：15.3
- 数据目录被更改到 `/data/pgdata`

#### TiKV

- 版本：6.5.3
- 配置：
  - `deploy_dir`：`/data/tikv-deploy`
  - `data_dir`：`/data/tikv-data`

#### etcd

- 版本：3.3.25
- 配置：
  - `data-dir`：`/data/etcd`

#### FoundationDB

- 版本：6.3.23
- 配置：
  - `data-dir`：`/data/fdb`

## 测试工具

每种元数据引擎都会运行以下所有测试。

### Golang Benchmark

在源码中提供了简单的元数据基准测试：[`pkg/meta/benchmarks_test.go`](https://github.com/juicedata/juicefs/blob/main/pkg/meta/benchmarks_test.go)

### JuiceFS Bench

JuiceFS 提供了一个基础的性能测试命令：

```bash
juicefs bench /mnt/jfs -p 4
```

### mdtest

- 版本：mdtest-3.3.0

在 3 个客户端节点上并发执行测试：

```bash
$ cat myhost
client1 slots=4
client2 slots=4
client3 slots=4
```

测试命令：

meta only

```shell
mpirun --use-hwthread-cpus --allow-run-as-root -np 12 --hostfile myhost --map-by slot /root/mdtest -b 3 -z 1 -I 100 -u -d /mnt/jfs
```

12000 * 100KiB files

```shell
mpirun --use-hwthread-cpus --allow-run-as-root -np 12 --hostfile myhost --map-by slot /root/mdtest -F -w 102400 -I 1000 -z 0 -u -d /mnt/jfs
```

### fio

- 版本：fio-3.28

```bash
fio --name=big-write --directory=/mnt/jfs --rw=write --refill_buffers --bs=4M --size=4G --numjobs=4 --end_fsync=1 --group_reporting
```

## 测试结果

### Golang Benchmark

- 展示了操作耗时（单位为 微秒/op），数值越小越好
- 括号内数字是该指标对比 Redis-Always 的倍数（`always` 和 `everysec` 均是 Redis 配置项 `appendfsync` 的可选值）
- 由于元数据缓存缘故，目前 `Read` 接口测试数据均小于 1 微秒，暂无对比意义

  |              | Redis-Always | Redis-Everysec | MySQL        | PostgreSQL   | TiKV       | etcd         | FoundationDB |
  |--------------|--------------|----------------|--------------|--------------|------------|--------------|--------------|
  | mkdir        | 558          | 468 (0.8)      | 2042 (3.7)   | 1076 (1.9)   | 1237 (2.2) | 1916 (3.4)   | 1842 (3.3)   |
  | mvdir        | 693          | 621 (0.9)      | 2693 (3.9)   | 1459 (2.1)   | 1414 (2.0) | 2486 (3.6)   | 1895 (2.7)   |
  | rmdir        | 717          | 648 (0.9)      | 3050 (4.3)   | 1697 (2.4)   | 1641 (2.3) | 2980 (4.2)   | 2088 (2.9)   |
  | readdir_10   | 280          | 288 (1.0)      | 1350 (4.8)   | 1098 (3.9)   | 995 (3.6)  | 1757 (6.3)   | 1744 (6.2)   |
  | readdir_1k   | 1490         | 1547 (1.0)     | 18779 (12.6) | 18414 (12.4) | 5834 (3.9) | 15809 (10.6) | 15276 (10.3) |
  | mknod        | 562          | 464 (0.8)      | 1547 (2.8)   | 849 (1.5)    | 1211 (2.2) | 1838 (3.3)   | 1763 (3.1)   |
  | create       | 570          | 455 (0.8)      | 1570 (2.8)   | 844 (1.5)    | 1209 (2.1) | 1849 (3.2)   | 1761 (3.1)   |
  | rename       | 728          | 627 (0.9)      | 2735 (3.8)   | 1478 (2.0)   | 1419 (1.9) | 2445 (3.4)   | 1911 (2.6)   |
  | unlink       | 658          | 567 (0.9)      | 2365 (3.6)   | 1280 (1.9)   | 1443 (2.2) | 2461 (3.7)   | 1940 (2.9)   |
  | lookup       | 173          | 178 (1.0)      | 557 (3.2)    | 375 (2.2)    | 608 (3.5)  | 1054 (6.1)   | 1029 (5.9)   |
  | getattr      | 87           | 86 (1.0)       | 530 (6.1)    | 350 (4.0)    | 306 (3.5)  | 536 (6.2)    | 504 (5.8)    |
  | setattr      | 471          | 345 (0.7)      | 1029 (2.2)   | 571 (1.2)    | 1001 (2.1) | 1279 (2.7)   | 1596 (3.4)   |
  | access       | 87           | 89 (1.0)       | 518 (6.0)    | 356 (4.1)    | 307 (3.5)  | 534 (6.1)    | 526 (6.0)    |
  | setxattr     | 393          | 262 (0.7)      | 992 (2.5)    | 534 (1.4)    | 800 (2.0)  | 717 (1.8)    | 1300 (3.3)   |
  | getxattr     | 84           | 87 (1.0)       | 494 (5.9)    | 333 (4.0)    | 303 (3.6)  | 529 (6.3)    | 511 (6.1)    |
  | removexattr  | 215          | 96 (0.4)       | 697 (3.2)    | 385 (1.8)    | 1007 (4.7) | 1336 (6.2)   | 1597 (7.4)   |
  | listxattr_1  | 85           | 87 (1.0)       | 516 (6.1)    | 342 (4.0)    | 303 (3.6)  | 531 (6.2)    | 515 (6.1)    |
  | listxattr_10 | 87           | 91 (1.0)       | 561 (6.4)    | 383 (4.4)    | 322 (3.7)  | 565 (6.5)    | 529 (6.1)    |
  | link         | 680          | 545 (0.8)      | 2435 (3.6)   | 1375 (2.0)   | 1732 (2.5) | 3058 (4.5)   | 2402 (3.5)   |
  | symlink      | 580          | 448 (0.8)      | 1785 (3.1)   | 954 (1.6)    | 1224 (2.1) | 1897 (3.3)   | 1764 (3.0)   |
  | newchunk     | 0            | 0 (0.0)        | 1 (0.0)      | 1 (0.0)      | 1 (0.0)    | 1 (0.0)      | 2 (0.0)      |
  | write        | 553          | 369 (0.7)      | 2352 (4.3)   | 1183 (2.1)   | 1573 (2.8) | 1788 (3.2)   | 1747 (3.2)   |
  | read_1       | 0            | 0 (0.0)        | 0 (0.0)      | 0 (0.0)      | 0 (0.0)    | 0 (0.0)      | 0 (0.0)      |
  | read_10      | 0            | 0 (0.0)        | 0 (0.0)      | 0 (0.0)      | 0 (0.0)    | 0 (0.0)      | 0 (0.0)      |

### JuiceFS Bench

|                  | Redis-Always     | Redis-Everysec   | MySQL           | PostgreSQL      | TiKV            | etcd            | FoundationDB    |
|------------------|------------------|------------------|-----------------|-----------------|-----------------|-----------------|-----------------|
| Write big file   | 730.84 MiB/s     | 731.93 MiB/s     | 729.00 MiB/s    | 744.47 MiB/s    | 730.01 MiB/s    | 746.07 MiB/s    | 744.70 MiB/s    |
| Read big file    | 923.98 MiB/s     | 892.99 MiB/s     | 905.93 MiB/s    | 895.88 MiB/s    | 918.19 MiB/s    | 939.63 MiB/s    | 948.81 MiB/s    |
| Write small file | 95.20 files/s    | 109.10 files/s   | 82.30 files/s   | 86.40 files/s   | 101.20 files/s  | 95.80 files/s   | 94.60 files/s   |
| Read small file  | 1242.80 files/s  | 937.30 files/s   | 752.40 files/s  | 1857.90 files/s | 681.50 files/s  | 1229.10 files/s | 1301.40 files/s |
| Stat file        | 12313.80 files/s | 11989.50 files/s | 3583.10 files/s | 7845.80 files/s | 4211.20 files/s | 2836.60 files/s | 3400.00 files/s |
| FUSE operation   | 0.41 ms/op       | 0.40 ms/op       | 0.46 ms/op      | 0.44 ms/op      | 0.41 ms/op      | 0.41 ms/op      | 0.44 ms/op      |
| Update meta      | 2.45 ms/op       | 1.76 ms/op       | 2.46 ms/op      | 1.78 ms/op      | 3.76 ms/op      | 3.40 ms/op      | 2.87 ms/op      |

### mdtest

 展示了操作速率（每秒 OPS 数），数值越大越好

|                    | Redis-Always | Redis-Everysec | MySQL    | PostgreSQL | TiKV      | etcd     | FoundationDB |
|--------------------|--------------|----------------|----------|------------|-----------|----------|--------------|
| **EMPTY FILES**    |              |                |          |            |           |          |              |
| Directory creation | 4901.342     | 9990.029       | 1252.421 | 4091.934   | 4041.304  | 1910.768 | 3065.578     |
| Directory stat     | 289992.466   | 379692.576     | 9359.278 | 69384.097  | 49465.223 | 6500.178 | 17746.670    |
| Directory removal  | 5131.614     | 10356.293      | 902.077  | 1254.890   | 3210.518  | 1450.842 | 2460.604     |
| File creation      | 5472.628     | 9984.824       | 1326.613 | 4726.582   | 4053.610  | 1801.956 | 2908.526     |
| File stat          | 288951.216   | 253218.558     | 9135.571 | 233148.252 | 50432.658 | 6276.787 | 14939.411    |
| File read          | 64560.148    | 60861.397      | 8445.953 | 20013.027  | 18411.280 | 9094.627 | 11087.931    |
| File removal       | 6084.791     | 12221.083      | 1073.063 | 3961.855   | 3742.269  | 1648.734 | 2214.311     |
| Tree creation      | 80.121       | 83.546         | 34.420   | 61.937     | 77.875    | 56.299   | 74.982       |
| Tree removal       | 218.535      | 95.599         | 42.330   | 44.696     | 114.414   | 76.002   | 64.036       |
| **SMALL FILES**    |              |                |          |            |           |          |              |
| File creation      | 295.067      | 312.182        | 275.588  | 289.627    | 307.121   | 275.578  | 263.487      |
| File stat          | 54069.827    | 52800.108      | 8760.709 | 19841.728  | 14076.214 | 8214.318 | 10009.670    |
| File read          | 62341.568    | 57998.398      | 4639.571 | 19244.678  | 23376.733 | 5477.754 | 6533.787     |
| File removal       | 5615.018     | 11573.415      | 1061.600 | 3907.740   | 3411.663  | 1024.421 | 1750.613     |
| Tree creation      | 57.860       | 57.080         | 23.723   | 52.621     | 44.590    | 19.998   | 11.243       |
| Tree removal       | 96.756       | 65.279         | 23.227   | 19.511     | 27.616    | 17.868   | 10.571       |

### fio

|                 | Redis-Always | Redis-Everysec | MySQL     | PostgreSQL | TiKV      | etcd      | FoundationDB |
|-----------------|--------------|----------------|-----------|------------|-----------|-----------|--------------|
| Write bandwidth | 729 MiB/s    | 737 MiB/s      | 736 MiB/s | 768 MiB/s  | 731 MiB/s | 738 MiB/s | 745 MiB/s    |
