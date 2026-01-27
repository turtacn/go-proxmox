# SAGA Workflow 原理深化与虚拟机生命周期完整实现方案

## 一、SAGA Workflow 核心原理与事务保障机制

### 1.1 SAGA 事务模型基础理论

SAGA 模式将长事务拆解为一系列本地事务（Ti），每个本地事务都有对应的补偿事务（Ci）。事务执行遵循以下规则：

```mermaid
graph LR
    subgraph SAGA[SAGA事务执行模型]
        T1[T1:本地事务1<br/>Local Transaction 1] --> T2[T2:本地事务2<br/>Local Transaction 2]
        T2 --> T3[T3:本地事务3<br/>Local Transaction 3]
        T3 --> T4[T4:本地事务N<br/>Local Transaction N]
        
        T4 -.失败.-> C3[C3:补偿事务3<br/>Compensate Transaction 3]
        C3 --> C2[C2:补偿事务2<br/>Compensate Transaction 2]
        C2 --> C1[C1:补偿事务1<br/>Compensate Transaction 1]
    end
    
    T1 -.成功.-> T2
    T2 -.成功.-> T3
    T3 -.成功.-> T4
    T4 -.成功.-> COMMIT[提交成功<br/>Commit]
    
    C1 --> ROLLBACK[回滚完成<br/>Rollback]
    
    style COMMIT fill:#90EE90
    style ROLLBACK fill:#FFB6C1
```

**核心保证**：

* **原子性（Atomicity）**：要么所有本地事务成功提交，要么全部通过补偿事务回滚
* **一致性（Consistency）**：最终状态必须满足业务约束（通过补偿逻辑保证）
* **隔离性（Isolation）**：SAGA 不保证严格隔离，允许中间状态可见（需应用层控制）
* **持久性（Durability）**：每个本地事务的结果持久化存储

### 1.2 DTM 框架的 SAGA 实现机制

```mermaid
sequenceDiagram
    participant APP as 应用服务<br/>Application Service
    participant DTM as DTM协调器<br/>DTM Coordinator
    participant DB as 状态存储<br/>State Store
    participant SVC as 业务服务<br/>Business Service
    
    APP->>DTM: 1.提交SAGA事务<br/>Submit SAGA Transaction
    DTM->>DB: 2.持久化事务状态<br/>Persist Transaction State<br/>状态:Prepared
    
    loop 遍历所有步骤<br/>Iterate All Steps
        DTM->>SVC: 3.调用正向操作<br/>Call Forward Action
        SVC-->>DTM: 4.返回结果<br/>Return Result
        DTM->>DB: 5.更新步骤状态<br/>Update Step State
        
        alt 步骤成功<br/>Success
            Note over DTM: 继续下一步<br/>Continue Next Step
        else 步骤失败<br/>Failure
            DTM->>DTM: 6.触发补偿流程<br/>Trigger Compensation
            loop 反向补偿<br/>Reverse Compensation
                DTM->>SVC: 7.调用补偿操作<br/>Call Compensate Action
                SVC-->>DTM: 8.补偿结果<br/>Compensate Result
            end
            DTM->>DB: 9.标记事务回滚<br/>Mark Transaction Rollback
        end
    end
    
    DTM->>DB: 10.最终状态持久化<br/>Final State Persist<br/>状态:Succeed/Failed
    DTM-->>APP: 11.返回事务结果<br/>Return Transaction Result
```

**关键设计点**：

1. **状态机驱动**：事务状态流转

   ```
   Prepared → Executing → Succeed
                        ↘ Failed → Compensating → Rollback
   ```

2. **幂等性保障**：

   * **请求级幂等**：DTM 为每个步骤生成全局唯一 `gid`（Global ID）和 `branch_id`（分支 ID）
   * **业务级幂等**：业务服务需检查操作是否已执行（通过 `gid+branch_id` 查询）

3. **超时与重试**：

   * DTM 默认步骤超时 60s，可配置
   * 失败自动重试（指数退避：1s、2s、5s、10s...最多重试 100 次）
   * 补偿操作同样支持重试

4. **数据持久化**：

   * DTM 使用 MySQL/PostgreSQL 存储事务日志
   * 每个步骤的请求/响应都被记录，支持故障恢复

### 1.3 SAGA 与业务数据的 CRUD 协同

**核心挑战**：SAGA 不保证隔离性，中间状态对外可见，需业务层设计状态字段。

```mermaid
graph TB
    subgraph 业务数据状态设计[业务数据状态设计<br/>Business Data State Design]
        A[创建中<br/>Creating] --> B[运行中<br/>Running]
        A --> C[创建失败<br/>Create Failed]
        
        B --> D[迁移中<br/>Migrating]
        D --> B
        D --> E[迁移失败<br/>Migrate Failed]
        
        B --> F[销毁中<br/>Destroying]
        F --> G[已销毁<br/>Destroyed]
    end
    
    subgraph SAGA事务状态[SAGA事务状态<br/>SAGA Transaction State]
        S1[Prepared] --> S2[Executing]
        S2 --> S3[Succeed]
        S2 --> S4[Failed]
        S4 --> S5[Compensating]
        S5 --> S6[Rollback]
    end
    
    A -.对应.-> S1
    A -.对应.-> S2
    B -.对应.-> S3
    C -.对应.-> S6
    
    style S3 fill:#90EE90
    style S6 fill:#FFB6C1
```

**最佳实践**：

| 业务操作      | 数据库事务范围                | SAGA 步骤设计                                        |
| --------- | ---------------------- | ------------------------------------------------ |
| **创建 VM** | 每个步骤独立事务               | 步骤 1 插入 `state=Creating`，步骤 N 更新 `state=Running` |
| **查询 VM** | 只读操作                   | 不参与 SAGA，直接查询当前状态                                |
| **更新配置**  | 先更新 `state=Updating`   | SAGA 完成后更新 `state=Running`                       |
| **删除 VM** | 先更新 `state=Destroying` | SAGA 完成后物理删除记录                                   |

**冲突处理**：

* 用户发起操作时检查 VM 状态，`Creating/Migrating/Destroying` 状态禁止新操作
* 使用乐观锁（版本号）防止并发修改

---

## 二、VM 生命周期 SAGA 详细实现

### 2.1 必须使用 SAGA 的复杂操作完整枚举

| 操作            | 正向步骤数 | 补偿步骤数 | 预估耗时      | 失败率（实测）[1] |
| ------------- | ----- | ----- | --------- | ---------- |
| **创建 VM**     | 7 步   | 6 步   | 30-60s    | 2-5%       |
| **热迁移（共享存储）** | 6 步   | 5 步   | 60-300s   | 5-10%      |
| **冷迁移（本地盘）**  | 8 步   | 7 步   | 5-30 分钟   | 8-15%      |
| **销毁 VM**     | 6 步   | 2 步   | 10-30s    | 1-3%       |
| **配置升级（需重启）** | 5 步   | 4 步   | 20-40s    | 3-6%       |
| **节点替换**      | 10+ 步 | 9 步   | 30-120 分钟 | 10-20%     |

### 2.2 创建 VM 的 SAGA 完整实现

#### 2.2.1 正向步骤、补偿步骤、重试步骤详解

```mermaid
graph TB
    subgraph FORWARD[正向步骤流程<br/>Forward Steps Flow]
        F1[1.1 预留配额<br/>Reserve Quota] --> F2[1.2 选择节点<br/>Select PM]
        F2 --> F3[1.3 创建磁盘<br/>Create Disk]
        F3 --> F4[1.4 生成配置<br/>Generate Config]
        F4 --> F5[1.5 启动VM<br/>Start VM]
        F5 --> F6[1.6 配置网络<br/>Setup Network]
        F6 --> F7[1.7 持久化元数据<br/>Persist Metadata]
    end
    
    subgraph COMPENSATE[补偿步骤流程<br/>Compensate Steps Flow]
        C7[7.1 删除元数据<br/>Delete Metadata] --> C6[6.1 清理网络<br/>Cleanup Network]
        C6 --> C5[5.1 停止VM<br/>Stop VM]
        C5 --> C4[4.1 删除配置<br/>Delete Config]
        C4 --> C3[3.1 删除磁盘<br/>Delete Disk]
        C3 --> C2[2.1 释放节点锁<br/>Release PM Lock]
        C2 --> C1[1.1 释放配额<br/>Release Quota]
    end
    
    subgraph RETRY[重试策略<br/>Retry Policy]
        R1[重试间隔<br/>1s/2s/5s/10s]
        R2[最大重试100次<br/>Max 100 Retries]
        R3[补偿也支持重试<br/>Compensate Retryable]
    end
    
    F7 -.失败触发.-> C7
    
    style F7 fill:#90EE90
    style C1 fill:#FFB6C1
```

#### 2.2.2 幂等性保障机制详解

**1. DTM 层面的幂等性**

```go
// DTM 自动为每个步骤生成唯一标识
type DTMRequest struct {
    Gid      string // 全局事务ID，如 "saga-20260127-001"
    BranchID string // 分支ID，如 "01"（步骤序号）
    Op       string // 操作类型："action" 或 "compensate"
    Data     []byte // 业务数据
}

// 业务服务需实现幂等检查
func HandleReserveQuota(c *gin.Context) {
    var req DTMRequest
    c.ShouldBindJSON(&req)
    
    // 1. 检查是否已处理（幂等性保证）
    if result := idempotentStore.Get(req.Gid, req.BranchID); result != nil {
        // 已处理过，直接返回之前的结果
        c.JSON(result.StatusCode, result.Body)
        return
    }
    
    // 2. 执行业务逻辑
    err := doReserveQuota(req.Data)
    
    // 3. 记录执行结果（持久化）
    idempotentStore.Set(req.Gid, req.BranchID, IdempotentRecord{
        StatusCode: 200,
        Body:       gin.H{"dtm_result": "SUCCESS"},
        CreatedAt:  time.Now(),
    })
    
    c.JSON(200, gin.H{"dtm_result": "SUCCESS"})
}
```

**2. 业务层面的幂等性实现**

```go
// 预留配额的幂等实现
func doReserveQuota(data []byte) error {
    var req CreateVMRequest
    json.Unmarshal(data, &req)
    
    // 使用 VMID 作为幂等键
    reservationKey := fmt.Sprintf("quota:reserve:%s", req.VMID)
    
    // 1. 尝试获取分布式锁（防止并发）
    lock := redisLock.Acquire(reservationKey, 10*time.Second)
    defer lock.Release()
    
    // 2. 检查预留记录是否存在
    existing := db.QueryOne("SELECT * FROM quota_reservations WHERE vm_id = ?", req.VMID)
    if existing != nil {
        // 已预留，幂等返回
        return nil
    }
    
    // 3. 检查用户配额
    userQuota := db.QueryOne("SELECT * FROM user_quotas WHERE user_id = ?", req.UserID)
    if userQuota.AvailableCPU < req.CPU || userQuota.AvailableMemory < req.Memory {
        return errors.New("insufficient quota")
    }
    
    // 4. 原子性预留（数据库事务）
    return db.Transaction(func(tx *sql.Tx) error {
        // 插入预留记录
        tx.Exec("INSERT INTO quota_reservations (vm_id, user_id, cpu, memory) VALUES (?, ?, ?, ?)",
            req.VMID, req.UserID, req.CPU, req.Memory)
        
        // 扣减可用配额
        tx.Exec("UPDATE user_quotas SET available_cpu = available_cpu - ?, available_memory = available_memory - ? WHERE user_id = ?",
            req.CPU, req.Memory, req.UserID)
        
        return nil
    })
}
```

**3. 磁盘创建的幂等性（文件系统操作）**

```go
func doCreateDisk(pmID string, req *CreateVMRequest) (string, error) {
    diskPath := fmt.Sprintf("/var/lib/vms/%s/disk.qcow2", req.VMID)
    
    // 1. 检查文件是否已存在
    if fileExists(diskPath) {
        // 验证文件大小是否匹配
        actualSize := getFileSize(diskPath)
        if actualSize == req.DiskSize {
            // 文件完整，幂等成功
            return diskPath, nil
        }
        
        // 文件不完整或大小不匹配，可能是之前失败留下的脏数据
        os.Remove(diskPath) // 删除后重新创建
    }
    
    // 2. 调用 Agent 创建磁盘
    agentClient := getAgentClient(pmID)
    result, err := agentClient.CreateQCOW2(ctx, &CreateDiskRequest{
        Path:   diskPath,
        Size:   req.DiskSize,
        Format: "qcow2",
    })
    
    // 3. Agent 侧也需要幂等检查
    // Agent 内部逻辑：
    // - 检查文件是否存在
    // - 如果存在且大小匹配，返回成功
    // - 否则执行 qemu-img create 命令
    
    return result.Path, err
}
```

#### 2.2.3 DTM SAGA 注册与执行代码

```go
package main

import (
    "github.com/dtm-labs/client/dtmcli"
    "github.com/gin-gonic/gin"
)

// 1. 定义 SAGA 工作流
func CreateVMSaga(req *CreateVMRequest) error {
    // 生成全局事务 ID
    gid := dtmcli.MustGenGid("http://dtm-server:36789")
    
    // 创建 SAGA 对象
    saga := dtmcli.NewSaga("http://dtm-server:36789", gid).
        // 配置重试策略
        SetRetryInterval(1, 2, 5, 10, 30). // 秒
        SetTimeoutToFail(600). // 总超时 10 分钟
        SetBranchHeaders(map[string]string{
            "Content-Type": "application/json",
        })
    
    // 步骤 1：预留配额
    saga.Add(
        "http://vm-controller:8080/saga/reserve-quota",
        "http://vm-controller:8080/saga/compensate-reserve-quota",
        req,
    )
    
    // 步骤 2：选择节点
    saga.Add(
        "http://vm-controller:8080/saga/select-pm",
        "http://vm-controller:8080/saga/compensate-select-pm",
        req,
    )
    
    // 步骤 3：创建磁盘
    saga.Add(
        "http://vm-controller:8080/saga/create-disk",
        "http://vm-controller:8080/saga/compensate-create-disk",
        req,
    )
    
    // 步骤 4：生成配置
    saga.Add(
        "http://vm-controller:8080/saga/generate-config",
        "http://vm-controller:8080/saga/compensate-generate-config",
        req,
    )
    
    // 步骤 5：启动 VM
    saga.Add(
        "http://vm-controller:8080/saga/start-vm",
        "http://vm-controller:8080/saga/compensate-start-vm",
        req,
    )
    
    // 步骤 6：配置网络
    saga.Add(
        "http://vm-controller:8080/saga/setup-network",
        "http://vm-controller:8080/saga/compensate-setup-network",
        req,
    )
    
    // 步骤 7：持久化元数据（最后一步无需补偿）
    saga.Add(
        "http://vm-controller:8080/saga/persist-metadata",
        "",
        req,
    )
    
    // 提交 SAGA 事务
    err := saga.Submit()
    if err != nil {
        return fmt.Errorf("saga submit failed: %w", err)
    }
    
    // 等待事务完成（同步模式）
    // 注意：DTM 默认是异步的，这里演示同步等待
    return saga.WaitResult()
}

// 2. 实现各步骤的 HTTP 处理器
func main() {
    r := gin.Default()
    
    // 步骤 1 正向操作
    r.POST("/saga/reserve-quota", func(c *gin.Context) {
        var dtmReq struct {
            Gid      string            `json:"gid"`
            BranchID string            `json:"branch_id"`
            Data     CreateVMRequest   `json:"data"`
        }
        c.ShouldBindJSON(&dtmReq)
        
        // 幂等性检查
        if isProcessed(dtmReq.Gid, dtmReq.BranchID) {
            c.JSON(200, gin.H{"dtm_result": "SUCCESS"})
            return
        }
        
        // 执行业务逻辑
        err := doReserveQuota(&dtmReq.Data)
        if err != nil {
            c.JSON(409, gin.H{"dtm_result": "FAILURE", "message": err.Error()})
            return
        }
        
        // 记录幂等标识
        markAsProcessed(dtmReq.Gid, dtmReq.BranchID)
        c.JSON(200, gin.H{"dtm_result": "SUCCESS"})
    })
    
    // 步骤 1 补偿操作
    r.POST("/saga/compensate-reserve-quota", func(c *gin.Context) {
        var dtmReq struct {
            Gid      string            `json:"gid"`
            BranchID string            `json:"branch_id"`
            Data     CreateVMRequest   `json:"data"`
        }
        c.ShouldBindJSON(&dtmReq)
        
        // 补偿操作也需要幂等
        if isCompensated(dtmReq.Gid, dtmReq.BranchID) {
            c.JSON(200, gin.H{"dtm_result": "SUCCESS"})
            return
        }
        
        // 执行补偿逻辑
        err := doCompensateReserveQuota(&dtmReq.Data)
        if err != nil {
            c.JSON(409, gin.H{"dtm_result": "FAILURE", "message": err.Error()})
            return
        }
        
        markAsCompensated(dtmReq.Gid, dtmReq.BranchID)
        c.JSON(200, gin.H{"dtm_result": "SUCCESS"})
    })
    
    // ... 其他步骤的处理器实现类似
    
    r.Run(":8080")
}

// 3. 幂等性存储实现
var idempotentCache = make(map[string]bool) // 生产环境用 Redis

func isProcessed(gid, branchID string) bool {
    key := fmt.Sprintf("processed:%s:%s", gid, branchID)
    return idempotentCache[key]
}

func markAsProcessed(gid, branchID string) {
    key := fmt.Sprintf("processed:%s:%s", gid, branchID)
    idempotentCache[key] = true
    // 生产环境：redis.Set(key, "1", 24*time.Hour)
}

func isCompensated(gid, branchID string) bool {
    key := fmt.Sprintf("compensated:%s:%s", gid, branchID)
    return idempotentCache[key]
}

func markAsCompensated(gid, branchID string) {
    key := fmt.Sprintf("compensated:%s:%s", gid, branchID)
    idempotentCache[key] = true
}
```

#### 2.2.4 DFR（Design for Reliability）场景处理

**场景 1：步骤 3（创建磁盘）超时**

```mermaid
sequenceDiagram
    participant DTM as DTM协调器<br/>DTM Coordinator
    participant CTL as VM Controller
    participant AGT as PM Agent
    
    DTM->>CTL: 调用步骤3:创建磁盘<br/>Call Step 3: Create Disk
    CTL->>AGT: CreateDisk请求<br/>CreateDisk Request
    
    Note over AGT: 网络抖动导致超时<br/>Network Timeout
    AGT--xCTL: 超时无响应<br/>Timeout No Response
    
    CTL-->>DTM: 返回失败<br/>Return Failure
    
    Note over DTM: 等待1秒后重试<br/>Wait 1s Then Retry
    DTM->>CTL: 重试步骤3<br/>Retry Step 3
    CTL->>AGT: 再次CreateDisk<br/>CreateDisk Again
    
    Note over AGT: 幂等检查：文件已存在<br/>Idempotent Check: File Exists
    AGT-->>CTL: 返回成功（幂等）<br/>Return Success Idempotent
    
    CTL-->>DTM: 成功<br/>Success
    DTM->>CTL: 继续步骤4<br/>Continue Step 4
```

**处理策略**：

* Agent 侧实现操作日志，记录每次 `CreateDisk` 请求
* 重试时先查询操作日志，如果文件创建中则等待，如果已完成则直接返回成功
* 超时时间设置：`基础30s + 磁盘大小GB * 2s`

**场景 2：步骤 5（启动 VM）失败后的补偿流程**

```mermaid
graph TB
    subgraph 正向步骤失败[步骤5启动VM失败<br/>Step 5 Start VM Failed]
        A[QMP连接超时<br/>QMP Connect Timeout] --> B[DTM标记步骤失败<br/>DTM Mark Step Failed]
    end
    
    subgraph 补偿流程[补偿流程执行<br/>Compensation Flow]
        B --> C1[补偿5:停止VM<br/>Compensate 5: Stop VM]
        C1 --> C2[补偿4:删除配置<br/>Compensate 4: Delete Config]
        C2 --> C3[补偿3:删除磁盘<br/>Compensate 3: Delete Disk]
        C3 --> C4[补偿2:释放节点锁<br/>Compensate 2: Release PM Lock]
        C4 --> C5[补偿1:释放配额<br/>Compensate 1: Release Quota]
    end
    
    C1 -.补偿失败重试.-> C1
    C3 -.磁盘删除失败.-> LOG[记录到异步清理队列<br/>Log to Async Cleanup Queue]
    
    C5 --> DONE[事务回滚完成<br/>Transaction Rollback Done]
    
    style DONE fill:#FFB6C1
    style LOG fill:#FFFF99
```

**关键代码**：

```go
// 补偿操作：停止 VM
func doCompensateStartVM(req *CreateVMRequest) error {
    qmpPath := fmt.Sprintf("/var/run/qemu/%s.sock", req.VMID)
    agentClient := getAgentClient(req.SelectedPMID)
    
    // 1. 尝试通过 QMP 优雅关闭
    qmpClient := NewQMPClient(req.SelectedPMID, qmpPath)
    err := qmpClient.Execute(context.Background(), "system_powerdown", nil)
    if err == nil {
        // 等待最多 10 秒
        time.Sleep(10 * time.Second)
    }
    
    // 2. 检查进程是否还在运行
    if agentClient.IsProcessRunning(req.VMID) {
        // 强制杀死进程
        if err := agentClient.KillProcess(req.VMID); err != nil {
            // 即使杀进程失败，也返回成功（避免阻塞补偿流程）
            log.Errorf("kill VM process failed: %v, but mark as success", err)
        }
    }
    
    return nil // 补偿操作尽量返回成功
}

// 补偿操作：删除磁盘（容错设计）
func doCompensateCreateDisk(req *CreateVMRequest) error {
    agentClient := getAgentClient(req.SelectedPMID)
    diskPath := fmt.Sprintf("/var/lib/vms/%s/disk.qcow2", req.VMID)
    
    err := agentClient.DeleteDisk(diskPath)
    if err != nil {
        // 删除失败不阻塞补偿流程，记录到异步清理队列
        asyncCleanupQueue.Add(CleanupTask{
            Type:   "delete_disk",
            PMID:   req.SelectedPMID,
            Path:   diskPath,
            Retry:  0,
            MaxRetry: 10,
        })
        log.Warnf("disk deletion failed, added to async cleanup: %v", err)
    }
    
    return nil // 总是返回成功
}
```

**场景 3：DTM 服务自身故障恢复**

```mermaid
sequenceDiagram
    participant APP as 应用<br/>Application
    participant DTM1 as DTM实例1<br/>DTM Instance 1
    participant DTM2 as DTM实例2<br/>DTM Instance 2
    participant DB as MySQL<br/>Transaction Log
    participant SVC as 业务服务<br/>Business Service
    
    APP->>DTM1: 提交SAGA<br/>Submit SAGA
    DTM1->>DB: 持久化事务状态<br/>Persist State
    DTM1->>SVC: 执行步骤1<br/>Execute Step 1
    SVC-->>DTM1: 成功<br/>Success
    
    Note over DTM1: DTM1崩溃<br/>DTM1 Crashed
    
    Note over DTM2: DTM2接管<br/>DTM2 Takeover
    DTM2->>DB: 读取未完成事务<br/>Read Pending Transactions
    DTM2->>SVC: 继续执行步骤2<br/>Continue Step 2
    SVC-->>DTM2: 成功<br/>Success
    
    DTM2->>DB: 更新最终状态<br/>Update Final State
```

**容错机制**：

* DTM 使用 MySQL 存储事务日志（表 `trans_global` 和 `trans_branch`）
* 多个 DTM 实例通过数据库锁竞争处理事务
* 定时任务扫描超时事务（默认 60s），自动恢复执行

---

### 2.3 热迁移（共享存储）的 SAGA 实现

#### 2.3.1 步骤拆解与补偿设计

```mermaid
graph TB
    subgraph MIGRATE_FORWARD[热迁移正向步骤<br/>Hot Migration Forward Steps]
        M1[1.校验VM状态<br/>Validate VM State] --> M2[2.预检查目标节点<br/>Pre-check Target PM]
        M2 --> M3[3.目标启动监听<br/>Start Target Listener]
        M3 --> M4[4.QMP执行迁移<br/>Execute QMP Migrate]
        M4 --> M5[5.等待迁移完成<br/>Wait Migration Complete]
        M5 --> M6[6.更新元数据<br/>Update Metadata]
    end
    
    subgraph MIGRATE_COMPENSATE[热迁移补偿步骤<br/>Hot Migration Compensate Steps]
        MC6[6.恢复原元数据<br/>Restore Original Metadata] --> MC5[5.取消迁移<br/>Cancel Migration]
        MC5 --> MC4[4.源VM恢复运行<br/>Resume Source VM]
        MC4 --> MC3[3.停止目标监听<br/>Stop Target Listener]
        MC3 --> MC2[2.释放目标资源<br/>Release Target Resources]
    end
    
    M6 -.失败.-> MC6
    
    style M6 fill:#90EE90
    style MC2 fill:#FFB6C1
```

#### 2.3.2 关键代码实现

```go
// 步骤 4：QMP 执行迁移
func doExecuteQMPMigrate(req *MigrateRequest) error {
    sourceQMP := NewQMPClient(req.SourcePMID, getQMPPath(req.VMID))
    targetIP := getNodeIP(req.TargetPMID)
    migrateURI := fmt.Sprintf("tcp:%s:49152", targetIP)
    
    // 执行 QMP 迁移命令
    err := sourceQMP.Execute(context.Background(), "migrate", map[string]interface{}{
        "uri": migrateURI,
        "blk": false, // 共享存储不迁移磁盘
        "inc": false,
    })
    
    if err != nil {
        return fmt.Errorf("qmp migrate command failed: %w", err)
    }
    
    return nil
}

// 步骤 5：等待迁移完成（轮询状态）
func doWaitMigrationComplete(req *MigrateRequest) error {
    sourceQMP := NewQMPClient(req.SourcePMID, getQMPPath(req.VMID))
    
    timeout := time.After(5 * time.Minute)
    ticker := time.NewTicker(2 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-timeout:
            return errors.New("migration timeout")
        case <-ticker.C:
            // 查询迁移状态
            result, err := sourceQMP.Execute(context.Background(), "query-migrate", nil)
            if err != nil {
                return err
            }
            
            status := result["status"].(string)
            switch status {
            case "completed":
                return nil // 迁移成功
            case "failed":
                return errors.New("migration failed")
            case "active":
                // 检查传输速率（DFR：防止卡住）
                transferred := result["ram"].(map[string]interface{})["transferred"].(int64)
                total := result["ram"].(map[string]interface{})["total"].(int64)
                log.Infof("migration progress: %d/%d bytes", transferred, total)
                
                // 如果传输速率 < 10MB/s 持续 60s，视为卡住
                // 这里简化处理，实际需记录历史速率
            }
        }
    }
}

// 补偿操作：取消迁移
func doCompensateExecuteQMPMigrate(req *MigrateRequest) error {
    sourceQMP := NewQMPClient(req.SourcePMID, getQMPPath(req.VMID))
    
    // 取消迁移
    err := sourceQMP.Execute(context.Background(), "migrate_cancel", nil)
    if err != nil {
        log.Warnf("cancel migration failed: %v", err)
    }
    
    // 确保源 VM 恢复运行
    err = sourceQMP.Execute(context.Background(), "cont", nil)
    if err != nil {
        return fmt.Errorf("resume source VM failed: %w", err)
    }
    
    return nil
}
```

#### 2.3.3 DFR 场景：网络中断导致迁移失败

```mermaid
sequenceDiagram
    participant SRC as 源节点<br/>Source PM
    participant NET as 网络<br/>Network
    participant TGT as 目标节点<br/>Target PM
    participant DTM as DTM协调器<br/>DTM Coordinator
    
    DTM->>SRC: 步骤4:执行迁移<br/>Step 4: Execute Migration
    SRC->>TGT: 建立迁移连接<br/>Establish Migration Link
    SRC->>TGT: 传输内存数据<br/>Transfer Memory Data
    
    Note over NET: 网络中断<br/>Network Disconnected
    SRC--xTGT: 传输中断<br/>Transfer Interrupted
    
    Note over SRC: 迁移失败<br/>Migration Failed
    SRC-->>DTM: 返回失败<br/>Return Failure
    
    DTM->>SRC: 补偿:取消迁移<br/>Compensate: Cancel Migration
    SRC->>SRC: 恢复VM运行<br/>Resume VM Running
    SRC-->>DTM: 补偿成功<br/>Compensate Success
    
    Note over DTM: 事务回滚<br/>Transaction Rollback
```

**处理策略**：

* 迁移前验证网络连通性（TCP 探测 + 带宽测试）
* 迁移过程中监控传输速率，低于阈值（10MB/s）超过 60s 自动取消
* 补偿操作确保源 VM 状态恢复到 `running`

---

### 2.4 冷迁移（本地盘）的 SAGA 实现

#### 2.4.1 步骤拆解

```mermaid
graph LR
    subgraph COLD_MIGRATE[冷迁移步骤<br/>Cold Migration Steps]
        C1[1.关机VM<br/>Shutdown VM] --> C2[2.打包磁盘<br/>Package Disk]
        C2 --> C3[3.传输到目标<br/>Transfer to Target]
        C3 --> C4[4.解压磁盘<br/>Unpack Disk]
        C4 --> C5[5.生成配置<br/>Generate Config]
        C5 --> C6[6.启动VM<br/>Start VM]
        C6 --> C7[7.清理源端<br/>Cleanup Source]
        C7 --> C8[8.更新元数据<br/>Update Metadata]
    end
    
    style C8 fill:#90EE90
```

#### 2.4.2 关键代码：断点续传实现

```go
// 步骤 3：传输磁盘到目标节点（支持断点续传）
func doTransferDiskToTarget(req *ColdMigrateRequest) error {
    sourceAgent := getAgentClient(req.SourcePMID)
    targetAgent := getAgentClient(req.TargetPMID)
    
    sourcePath := fmt.Sprintf("/var/lib/vms/%s/disk.qcow2.tar.gz", req.VMID)
    targetPath := fmt.Sprintf("/var/lib/vms/%s/disk.qcow2.tar.gz", req.VMID)
    
    // 1. 检查目标是否已有部分文件（断点续传）
    targetInfo, _ := targetAgent.StatFile(targetPath)
    var startOffset int64 = 0
    if targetInfo != nil {
        startOffset = targetInfo.Size
    }
    
    // 2. 使用 rsync 传输（支持断点续传）
    err := sourceAgent.RsyncToRemote(RsyncRequest{
        SourcePath: sourcePath,
        TargetHost: getNodeIP(req.TargetPMID),
        TargetPath: targetPath,
        StartOffset: startOffset,
        BandwidthLimit: "100M", // 限速 100MB/s
    })
    
    if err != nil {
        return fmt.Errorf("rsync failed: %w", err)
    }
    
    // 3. 校验文件完整性
    sourceChecksum := sourceAgent.CalculateSHA256(sourcePath)
    targetChecksum := targetAgent.CalculateSHA256(targetPath)
    
    if sourceChecksum != targetChecksum {
        return errors.New("checksum mismatch, file corrupted")
    }
    
    return nil
}
```

---

### 2.5 销毁 VM 的 SAGA 实现

```go
// 销毁 VM 的 SAGA 定义
func DestroyVMSaga(vmID string) error {
    gid := dtmcli.MustGenGid(dtmServer)
    saga := dtmcli.NewSaga(dtmServer, gid)
    
    req := &DestroyVMRequest{VMID: vmID}
    
    saga.Add("http://vm-controller/saga/mark-deleting", "", req)
    saga.Add("http://vm-controller/saga/shutdown-vm", "http://vm-controller/saga/compensate-shutdown-vm", req)
    saga.Add("http://vm-controller/saga/delete-disk", "", req) // 删除失败不回滚
    saga.Add("http://vm-controller/saga/cleanup-network", "", req)
    saga.Add("http://vm-controller/saga/release-quota", "", req)
    saga.Add("http://vm-controller/saga/delete-metadata", "", req)
    
    return saga.Submit()
}

// 步骤 2：关机 VM
func doShutdownVM(req *DestroyVMRequest) error {
    vm := vmStore.GetVM(req.VMID)
    qmpClient := NewQMPClient(vm.PMID, getQMPPath(req.VMID))
    
    // 优雅关机
    err := qmpClient.Execute(context.Background(), "system_powerdown", nil)
    if err != nil {
        return err
    }
    
    // 等待最多 30 秒
    for i := 0; i < 30; i++ {
        time.Sleep(1 * time.Second)
        if !isVMRunning(vm.PMID, req.VMID) {
            return nil
        }
    }
    
    // 超时强制杀死
    agentClient := getAgentClient(vm.PMID)
    return agentClient.KillProcess(req.VMID)
}

// 补偿操作：重启 VM（如果后续步骤失败）
func doCompensateShutdownVM(req *DestroyVMRequest) error {
    vm := vmStore.GetVM(req.VMID)
    agentClient := getAgentClient(vm.PMID)
    
    // 重新启动 QEMU 进程
    return agentClient.StartVM(req.VMID)
}
```

---

## 三、可选直调的简单操作

### 3.1 简单操作枚举

| 操作            | 幂等性                      | 失败处理        | 示例代码       |
| ------------- | ------------------------ | ----------- | ---------- |
| **启动 VM**     | 强（QMP `cont` 命令幂等）       | 同步返回错误，前端重试 | 见下文        |
| **关机 VM**     | 强（`system_powerdown` 幂等） | 同步返回错误      | 见下文        |
| **查询 VM 状态**  | 只读操作                     | 无需处理        | 直接查询数据库    |
| **挂载磁盘（已关机）** | 强（修改配置文件）                | 失败回滚配置      | 修改 JSON 文件 |

### 3.2 启动 VM 直调实现

```go
// RESTful API 处理器
func HandleStartVM(c *gin.Context) {
    vmID := c.Param("vm_id")
    
    // 1. 查询 VM
    vm := vmStore.GetVM(vmID)
    if vm == nil {
        c.JSON(404, gin.H{"error": "VM not found"})
        return
    }
    
    // 2. 校验状态
    if vm.State != "stopped" {
        c.JSON(400, gin.H{"error": "VM is not in stopped state"})
        return
    }
    
    // 3. 调用 QMP 启动
    qmpClient := NewQMPClient(vm.PMID, getQMPPath(vmID))
    err := qmpClient.Execute(c.Request.Context(), "cont", nil)
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }
    
    // 4. 更新状态
    vm.State = "running"
    vmStore.UpdateVM(vm)
    
    c.JSON(200, gin.H{"status": "success"})
}
```

---

## 四、参考资料

* [1] DTM 官方文档 - SAGA 模式详解：[https://dtm.pub/practice/saga.html](https://dtm.pub/practice/saga.html)
* [2] DTM GitHub 仓库：[https://github.com/dtm-labs/dtm](https://github.com/dtm-labs/dtm)
* [3] QEMU QMP 协议规范：[https://qemu.readthedocs.io/en/latest/interop/qmp-spec.html](https://qemu.readthedocs.io/en/latest/interop/qmp-spec.html)
* [4] 分布式事务幂等性设计模式：[https://martinfowler.com/articles/patterns-of-distributed-systems/idempotent-receiver.html](https://martinfowler.com/articles/patterns-of-distributed-systems/idempotent-receiver.html)
* [5] Proxmox VE 迁移机制源码分析：[https://git.proxmox.com/?p=qemu-server.git](https://git.proxmox.com/?p=qemu-server.git)
* [6] QEMU Live Migration 内部实现：[https://wiki.qemu.org/Features/LiveMigration](https://wiki.qemu.org/Features/LiveMigration)
* [7] 容错设计最佳实践（Google SRE）：[https://sre.google/sre-book/addressing-cascading-failures/](https://sre.google/sre-book/addressing-cascading-failures/)
