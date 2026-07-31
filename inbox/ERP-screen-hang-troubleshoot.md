# 故障排查报告:MySoft ERP - Good Receive Detail 界面卡死(Not Responding)

**文档类型:** Troubleshooting Article
**受影响模块:** MySoft ERP300 - Good Receive Note (GRN) Detail Screen
**问题状态:** Root Cause 已确认,业务流程验证中
**适用环境:** Windows Server(启用 IE ESC),SQL Server (SQLEXPRESS instance)

---

## 1. 问题描述(Issue Summary)

用户在这台特定 PC 上,每次登录 MySoft ERP300 并进入 **Good Receive Detail(GRN 详情)** 界面时,应用**必现卡死**,Windows 显示 **Not Responding**。该问题只在此 PC 上出现,其他 PC 打开相同界面均正常。

**关键现象:**
- Task Manager 显示 CPU 占用稳定在 **~25%**(该 PC 为 4 core / 4 logical processor,25% ≈ 单核心被占满)
- Memory 占用稳定在 **16180K** 左右,不再增长(排除 memory leak 持续增长的可能)
- 需要强制结束进程(kill process)才能退出

---

## 2. 排查过程(Diagnostic Steps)

### 2.1 资源占用分析(Task Manager / Resource Monitor)
使用 Task Manager 和 Resource Monitor 确认:
- CPU 固定占满 1 个核心 → 判断为**单线程死循环(single-thread infinite loop)**,而非等待外部资源(网络 / 锁)
- Memory 不再增长 → 排除数据加载阶段卡住的可能,问题出在**后续处理逻辑**

### 2.2 Handle 排查
在 Resource Monitor 的 **Associated Handles** 中发现:
- 一个 mutex:`ZonesLockedCacheCounterMutex`(与 WinINet / IE 安全区域相关)
- 该 PC 的 IE 启用了 **Enhanced Security Configuration (IE ESC)**

这一线索一度指向"内嵌 WebBrowser 控件被 IE ESC 拦截",但后续排查未能证实,**此方向最终被排除**。

### 2.3 Process Monitor(procmon)排查
使用 Sysinternals **Process Monitor** 抓取进程行为,发现:
- 短时间内产生 **130 万+ 次** `RegOpenKey` 操作
- 目标路径:`HKCU\Software\Classes\CLSID\{0000051A-0000-0010-8000-00AA006D2EA4}\ExtendedErrors`
- 结果均为 `NAME NOT FOUND`

通过 **Stack(调用栈)** 分析,确认调用来源为:
```
oledb32.dll → DllGetClassObject
msado15.dll (ADO 15.0)
```

**结论:** 该 CLSID 属于 **ADO / OLE DB 的 Rich Error Info(标准错误详情查找机制)**,并非缺失的 COM 组件(与另一台正常 PC 对比注册表结构一致)。该机制被触发上百万次,说明程序在**反复遇到同一个 ADO/OLE DB 错误**,每次报错都会触发一次这个查找动作,从而形成死循环。

### 2.4 SQL Server Profiler 抓包(Root Cause 确认)
使用 **SQL Server Profiler** 连接到 `APPSRV\SQLEXPRESS` 实例,勾选 **Errors and Warnings**(Exception / User Error Message)及 **SQL:BatchCompleted / RPC:Completed** 后重现问题,抓到关键报错:

```
EventClass: Exception / User Error Message
TextData: Invalid object name 'SSTImportK1'.

SQL 语句:
SELECT DISTINCT GRNItemNo FROM SSTImportK1 WHERE GRNVoucherNo='HQ03263'
```

**确认根因(Root Cause):**
`SSTImportK1` 是一张**不存在**的表(推测为流程中动态创建 / 用后即删的临时表,staging table)。程序在 GRN Detail 界面加载时,针对某个 GRN Voucher No.(如 `HQ03263`)执行了对该表的 SELECT 查询;由于该表不存在,SQL 层直接返回 `Invalid object name`。程序对该错误**没有做异常捕获(exception handling)或跳出循环的处理**,导致:

```
执行查询 → 报错(Invalid object name)
        → 触发 ADO 查错误详情(ExtendedErrors CLSID)
        → 仍然报错 → 再次执行查询
        → ... (死循环)
```

大量重复执行 → CPU 打满单核心 → UI 线程被阻塞 → 界面 Not Responding。

---

## 3. 为什么登录时不卡,只有进入这个界面才卡?

登录(login)阶段只验证数据库连接与账号权限,**不涉及 `SSTImportK1` 这张表**,所以登录能正常通过。只有进入 **GRN Detail** 这个具体界面、针对特定单据加载详情数据时,才会触发对 `SSTImportK1` 的查询,因此问题只在这一步出现。

---

## 4. 已排除的方向(Ruled Out)

| 排查方向 | 结果 |
|---|---|
| IE Enhanced Security Configuration (IE ESC) 拦截内嵌浏览器控件 | 未证实,推翻 |
| COM 组件 CLSID 缺失 / 注册表损坏 | 对比正常 PC 后确认注册表结构一致,非根因 |
| SQL Server 实例名不匹配 / 连接失败 | 登录能正常通过,排除连接层问题 |
| `regsvr32 oleaut32.dll` 重新注册系统组件 | 无效尝试(且注册失败,Error 0x80070005 Access Denied) |

---

## 5. 待验证 / 下一步(Next Steps)

1. **确认 `SSTImportK1` 表的设计用途:**
   - 是否为某个"导入(Import)/ 暂存(Staging)"流程动态创建的临时表
   - 该表应该在哪个前置步骤被创建(例如某个"GRN Import"按钮或菜单)

2. **验证是否为业务操作顺序问题:**
   - 换一个**已完整走过导入流程**的 GRN Voucher No. 测试该界面,确认是否正常
   - 若正常单据无问题、只有特定单据(如 `HQ03263`, `HQ03268`)报错 → 说明是这些单据的前置导入步骤未完成或中途失败
   - 若所有单据均报同样的错 → 说明该表的创建逻辑存在系统性问题,需要厂商(MySoft)介入检查对应程序模块或 stored procedure

3. **联系 MySoft 技术支持 / DBA:**
   - 提供本报告中的 SQL 报错信息(`Invalid object name 'SSTImportK1'`)及触发场景(进入 Good Receive Detail 界面)
   - 请求确认该表的创建时机及正常业务流程顺序

4. **短期缓解建议(Workaround,非根本修复):**
   - 若确认是"未跑导入步骤"导致,应指导用户在打开 GRN Detail 前先完成对应的导入 / 暂存操作
   - 长期应建议厂商在程序中为该 SELECT 查询增加**表存在性检查(existence check)**或 **try-catch 异常处理**,避免表缺失时陷入死循环

---

## 6. 使用到的诊断工具(Tools Used)

- **Task Manager** — 初步确认 CPU / Memory 占用情况
- **Resource Monitor** — 查看 Associated Handles(mutex、CLSID 引用)
- **Process Monitor (procmon)** — 抓取注册表 / 文件操作,定位死循环行为及调用栈(Stack)
- **SQL Server Profiler** — 抓取实际执行的 SQL 语句及数据库层报错信息,最终定位 Root Cause

---

*本文档基于实际排查过程整理,供后续同类问题(应用界面卡死 / Not Responding)排查参考。*
