---
title: "SoftRoCE Environment Setup"
tags: ["Tutorial", "RDMA", "Environment Setup"]
key: post7
show_edit_on_github: false
---

## 簡介
記錄一下如何在VM上配置SoftRoCE，希望能在沒有RDMA實體網卡的情況下，也能編譯與運行測試RDMA相關程式

## 步驟

### 1 Install VM

* 直接安裝kernel version >= 4.9的distro, 推薦使用**Ubuntu 18.04/20.04**
* 如果使用kernel version < 4.9的distro，則需要自己打開與RDMA相關的選項並重新編譯kernel
    * 請參考[這裡](https://zhengjingwei.github.io/2018/01/27/Soft-RoCE-setup/)

### 2 Install User Library
* 先裝這些庫

```bash
sudo apt install libibverbs-dev librdmacm-dev
```

* 安裝 [rdma-core](https://github.com/linux-rdma/rdma-core) ([librxe](https://github.com/SoftRoCE/librxe-dev)過時了，不建議使用)

```bash
git clone git@github.com:linux-rdma/rdma-core.git
sudo apt install build-essential cmake gcc libudev-dev libnl-3-dev libnl-route-3-dev 
sudo apt install ninja-build pkg-config valgrind python3-dev cython3 python3-docutils 
sudo apt install pandoc

cd rdma-core
bash build.sh
```

### 3 Add Virtual RDMA NIC
* 每次reboot後都要重新加一次

```bash
rdma link add <RDMA_NIC_NAME> type <TYPE> netdev <DEVICE>
```
* `RDMA_NIC_NAME`: 自己取
* `TYPE`: rxe *(for RoCE)*
* `DEVICE`: 實體網卡名稱


## 測試
簡單測試兩台VM之間能不能透過RDMA傳輸訊息

### rping
```bash
sudo apt install rdmacm-utils

## server side
rping -s -v

## client side
rping -c -v -a <SERVER_IP>
```

### ib_rc_pingpong
```bash
## server side
ibv_rc_pingpong -d <RDMA_NIC_NAME> -g <GID>

## client side
ibv_rc_pingpong -d <RDMA_NIC_NAME> -g <GID> <SERVER_IP>
```

### [perftest](https://github.com/linux-rdma/perftest)
```zsh
sudo apt install perftest

## server side
ib_send_bw -a

## client side
ib_send_bw -a <SERVER_IP>
```


## 編譯並運行RDMA程式
* 推薦使用此[Repo](https://github.com/tarickb/the-geek-in-the-corner) 
    * 包含基本client/server收發訊息、讀寫檔案等

## Ref.
* <https://thegeekinthecorner.wordpress.com/page/2>
* <https://kruztw.tw/%e5%88%a9%e7%94%a8-ibverbs-%e5%af%a6%e5%81%9a-rdma-2/>
* <https://zhuanlan.zhihu.com/p/55142568>