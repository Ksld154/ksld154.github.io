---
title: "HyperLoop: Group-Based NIC-Offloading to Accelerate Replicated Transactions in Multi-Tenant Storage Systems"
tags: Paper SIGCOMM 2018 RDMA
show_edit_on_github: false
---

## 0 Abstract
* Hyperloop利用RDMA將memory operation offload到NIC，
* 透過避開CPU達到加速replicated transaction的目標

## 1 Introduction
* **Replicated Transaction**
    * DB為了確保availability跟durability，會將data複製成多份存在各處
    * Replicated Transaction就是將這些data同時update(同步)，確保consistancy
    * 會涉及CPU及locking, 故latency高且不穩定
* 現有解法(如kernel bypass, 傳統RDMA)，僅適用於**單一storage system**上的RT
    * 在多replica時，需要CPU I/O polling
    * CPU會參與transaction每個步驟
    * 故不適用 Multi-Tenant Storage Systems
* **HyperLoop 創新點**
    * **RDMA WAIT**
        * buildin cmd但少用 
        * 當接收到特定的op後，wait才會activate我們pre-post的RDMA ops
        * 可避免CPU polling
    * 可以主動update其他NIC的work queue entry的內容
        * 要修改NIC driver
## 2 Background & Issues 
* **簡介 Replicated Transaction**
    * 單一transaction update流程(atomic):
        * 1. 將X,Y的修改寫到log(memory)
        * 2. lock X, Y
        * 3. commit(flush) log to NVM
        * 4. Release lock
    * 確保data update到足夠多份的replica之後，才會通知user 
    * **Two-phase commit**: consensus protocol, 決定replicas何時要flush
    * **Chain Replication**
        * 一種Two-phase commit, **Hyperloop的主要優化目標**
        * 1. chain上的replica先照順序寫log
        * 2. Last replica寫完log後，往回傳ACK，代表各個replica都準備好要commit了
        * 3. 每個replica收到ACK後就可commit
        * 4. head收到ACK，代表Replicated Transaction作完
* Multi-tenancy Causes High Latency
    * Busy CPU
        * 傳統作法是每份replica per-process
        * process負責**LOCK**：從network stack撈log，再將log存到storage stack
        * 但在Multi-tenant中，一台server會切成不同用戶的replica，占用過多process
        * CPU heavd load, many context switches => **HIGH Latency**
    * 現有解法的不足
        * user-level TCP, 傳統RDMA只offload network stack到NIC
        * 關鍵的storage **replication, transaction還是卡在CPU**
        * 避免context switch而用的core-pinning, 更不適合用在multi-tenant
            * 因 replica數(partition數) >>> core數
## 3 HyperLoop Overview
### 3-1 Design Goals
1. No replica CPU involvement **on the critical path**
2. Provide **ACID** operation
3. End-host only implementation based on commodity hardware
    * 只要動client跟replica的硬體，不用動到switch 
### 3-2 HyperLoop Architecture
![](https://i.imgur.com/EjIIYM0.png)
* Net primitive lib: 實作四個group network primitives(by RDMA)
    * 提供Replicated Transactions的寫入/同步等
    * 不涉及Replica的CPU
* RDMA NIC:
    * 接收上個NIC來的封包後，執行對應的memory ops, 再forward封包到下一個replica

## 4 HyperLoop Design
### 4-1 Key Ideas
1. 傳統的RDMA**為什麼需要CPU Polling**
    * Client -> Replica#1: 用RDMA直接送，不用經CPU
    * **Replica#1 -> Replica#2:** **CPU Polling**
        * 因要forward給#2的東西，是根據client送來的而定
        * 故#1會pre-post recv request，再定時polling自己的complete queue
        * 確定client送完東西後，才能決定何時要forward哪些東西給#2 **(when and what)**
    * 照此道理，每個replica的CPU都必須做polling，等到上一個replica送完東西後，才知道要何時送哪些東西給下個replica
    * 故Hyperloop便讓**replica自己偵測**recv complition，**並自動forward**對應event到下一個replica，藉此避開polling 
        * => **RDMA WAIT**
2. **RDMA WAIT (When)**
    ![](https://i.imgur.com/EZnUhvv.png)
    * 每個replica都要pre-post RECV跟WAIT跟Operation  
    * Operation會被WAIT block住
    * WAIT被RECV trigger後，會activate blocked operation
    * Operatiun仍未被決定(fixed replication)   

3. **Remote Work Request Manipulation (What)**
    ![](https://i.imgur.com/Xx307ZX.png)
    * 根據收到的封包，決定forwarded operation的內容
    * 必須修改網卡driver，讓client(or replica)可以修改replica的WQ裡， 特定pre-posted work request的內容
        * 因為WQ也是一段memory，故client應該要可以透過RDMA修改其內容
        * 根據收到的pre-calc. metadata來改(包含各replica memory info)
    * 修改完後，wait會activate這些WR，並forward出去
    * 會占用多餘的WR送metadata跟WAIT，但影響不大    
4. Integration with other RDMA operations to support ACID
### 4-2 Detailed Primitives Design
* 利用以上概念，寫出四隻API提供Replicated Transactions所需操作
1. **Group write (gWRITE)**
    *  Allows client to write data to memory regions of **a group of remote nodes** without involving their CPUs
    *  提供寫入log **(replicated transaction log management)**
2. **Group compare-and-swap (gCAS):**
    ![](https://i.imgur.com/Y3EGOYa.png)
    * 讓各replica比較給定的數值，決定要不要swap數值
    * 提供group lock
3. **Group memory copy (gMEMCPY)**
    * Copying the data size of size from `src_offset` to `dest_offset` for all nodes
    *  NICs will copy the data from the **log region to the persistent data** region without involving the CPUs
    *  Remote log processing 
4. **Group RDMA flush (gFLUSH)**
    *  Supports the **durability** at the “**NIC-level**”
    *  把NIC cache裡的data, flush到NVM(SSD)
## 5 Case Study
Skip



## Architecture


## Figure


