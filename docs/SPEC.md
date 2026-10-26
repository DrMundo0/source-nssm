# SPEC：NSSM 技术规格说明

## 1. 文档目的

本文档定义 NSSM 项目的技术边界、总体架构、模块职责、数据模型、安全要求和兼容性要求，作为开发实现、代码评审和测试验证的技术依据。

## 2. 技术背景

NSSM 是一个用于将任意可执行程序作为 Windows 服务运行与管理的工具。它通过 Windows 服务控制管理器（SCM）与注册表配置，实现对后台程序的生命周期管理。

本仓库在开源 NSSM 的基础上进行维护增强，重点解决以下问题：
- 兼容现代 Windows 的服务行为
- 修复稳定性问题
- 抹平中文文案和术语不一致
- 提升文档和构建流程说明
- 维持兼容命令行接口

## 3. 目标平台与约束

### 3.1 平台
- Windows 7 及以上
- 32 位和 64 位 Windows 版本
- Visual Studio C++ 桌面开发工具链

### 3.2 编译规范
- 以 C++ 实现
- 使用 Unicode/UTF-16 相关代码路径
- 通过 Windows API 管理服务、注册表、进程和事件日志

### 3.3 兼容性策略
- 明确最低系统基线：_WIN32_WINNT=0x0601
- 不再使用 Win7+ 已失效的交互式服务选项
- 服务类型统一按 SERVICE_WIN32_OWN_PROCESS 执行

## 4. 总体架构

项目核心分为以下几大层面：

### 4.1 程序入口层
- nssm.cpp
- 功能：解析命令行参数、分发命令、进行管理员权限检测、启用 UTF-8 兼容、启动服务控制器

### 4.2 服务控制层
- service.cpp / service.h
- 功能：创建、打开、删除和编辑服务，处理生命周期控制逻辑

### 4.3 配置参数层
- settings.cpp / settings.h
- 功能：统一定义参数表，支持 get/set/reset/unset/dump

### 4.4 注册表层
- registry.cpp / registry.h
- 功能：读写服务参数、事件日志消息名称、注册表键值

### 4.5 进程控制层
- process.cpp / process.h
- 功能：优雅停止、进程树终止、权限提升、优先级与亲和性设置

### 4.6 I/O 与日志层
- io.cpp / io.h
- 功能：stdin/stdout/stderr 重定向、日志轮转、在线/离线切分

### 4.7 GUI 层
- gui.cpp / gui.h
- 功能：图形界面安装、编辑和卸载操作

### 4.8 事件与消息层
- event.cpp / event.h
- 功能：日志输出、控制台输出、弹出消息框

### 4.9 环境变量层
- env.cpp / env.h
- 功能：环境块处理、变量展开、合并与清理

### 4.10 账户与权限层
- account.cpp / account.h
- 功能：SID 互转、账号规范化、LocalSystem 判断及权限申请

### 4.11 事件钩子层
- hook.cpp / hook.h
- 功能：监听 start/stop/exit/power/rotate 等事件，并执行用户脚本

### 4.12 资源及消息层
- nssm.rc / resource.h / version.h / messages.mc
- 功能：程序图标、资源定义、版本信息和中英文消息表

## 5. 模块职责说明

### 5.1 nssm.cpp
职责：
- 程序入口 _tmain
- 解析命令，如 start/stop/restart/install/edit/get/set/reset/unset/list/processes/version/remove
- 处理管理员权限提升
- 设置 UTF-8 控制台输出
- 调用 SCM 启动服务控制器

关键行为：
- 若命令为 version，则直接输出版本信息
- 若命令为 install/remove，则在要求管理员权限时进行提升
- 若未带参数，则根据环境选择启动服务模式或展示 usage 文本

### 5.2 service.cpp
职责：
- 打开或创建服务
- 设置服务启动类型、依赖、描述、登录账号等
- 处理服务控制消息
- 处理服务生命周期与恢复动作

关键行为：
- service_main 作为 Windows 服务主线程入口
- 支持启动、停止、暂停、继续、轮转等控制指令

### 5.3 settings.cpp
职责：
- 统一管理所有可配置参数
- 根据 settings_t 定义的名称、类型和回调函数，执行 get/set/reset/dump

设计亮点：
- 参数定义集中在 settings_t
- 配置键使用统一命名方式
- 参数支持原生属性与扩展属性

### 5.4 registry.cpp
职责：
- 针对 Windows 注册表读写
- 维护服务独立配置项
- 保护环境变量、日志轮转、退出动作等参数

关键要求：
- 覆盖 String/ExpandString/DWORD/Multi-String 等注册表类型
- 参数校验必须严格，避免非法值写入配置项

### 5.5 process.cpp
职责：
- 执行进程优雅终止流程
- 处理终止顺序：Console -> Window -> Threads -> TerminateProcess
- 获取并设置权限
- 支持优先级与 CPU 亲和性

关键要求：
- 终止逻辑必须逐步降级，不得直接无序 kill
- 需要对进程树作完整遍历，防止残留子进程

### 5.6 io.cpp
职责：
- 构造服务子进程的标准输入输出流
- 支持日志写入文件
- 支持轮转触发
- 处理 BOM 和时间戳前缀

关键要求：
- 可配置在线/离线轮转
- 支持按时间与按大小阈值轮转
- 在直接写文件场景下，需避免和在线轮转冲突

### 5.7 gui.cpp
职责：
- 提供安装、卸载、编辑 GUI 流程
- 多 tab 配置界面：应用、详情、登录、依赖、进程、关闭、退出、I/O、日志轮转、环境、钩子

关键要求：
- 所有配置项须与 CLI 参数保持一致
- 不展示失效功能，例如 Win7+ 不支持的交互式服务选项

### 5.8 event.cpp
职责：
- 统一控制台消息、GUI 弹窗和事件日志输出
- 封装 print_message / popup_message / log_event

关键要求：
- 错误输出具备可诊断性
- 与消息资源表保持一致

### 5.9 env.cpp
职责：
- 处理环境块的创建、复制、合并与清理
- 支持用户变量与系统变量扩展
- 在创建子进程前验证环境是否合法

### 5.10 account.cpp
职责：
- 执行账号名与 SID 对应关系处理
- 检查用户是否具有服务登录权限
- 支持 LocalSystem/虚拟账户和自定义用户

### 5.11 hook.cpp
职责：
- 钩子事件定义与执行机制
- 支持 pre/post/change/resume 等阶段
- 对脚本/程序执行器进行超时与清理控制

## 6. 数据模型与参数设计

### 6.1 参数模型
项目通过 settings_t 实现统一参数模型，结构如下：
- name：参数名
- type：参数类型
- default_value：默认值
- native：是否为服务原生属性
- additional：附加行为
- set/get/dump：回调处理函数

### 6.2 关键参数
- DisplayName：显示名称
- Description：服务描述
- ImagePath：执行路径
- AppDirectory：工作目录
- Arguments：启动参数
- ObjectName：服务账号
- Start：启动类型
- Type：服务类型
- DependOnService：依赖服务
- DependOnGroup：依赖组
- AppExit：退出动作
- AppStdout / AppStderr：输出文件
- AppRotate / AppRotateOnline：轮转开关
- AppRotateSeconds / AppRotateBytes：轮转阈值
- AppEnvironment / AppEnvironmentExtra：环境变量
- AppPriority / AppAffinity：优先级和亲和性

## 7. 安全要求

### 7.1 权限要求
- 只有管理员可以执行服务安装、删除、编辑等高权限操作。
- 提升权限时应使用 ShellExecuteEx 的 runas 方式。

### 7.2 敏感信息处理
- 服务密码等敏感字段应在完成操作后用 SecureZeroMemory 清理。
- 不应将密码直接写入日志或事件文本。

### 7.3 注册表与参数合法性
- 所有参数必须先校验再写入.
- 对非法类型、超长值、错误枚举值等进行拒绝处理。

## 8. 兼容性设计

### 8.1 Windows 兼容
- 最低支持：Windows 7
- 交互式服务在 Win7+ 失效，需从 GUI 中移除
- 使用 SERVICE_WIN32_OWN_PROCESS 与现代 SCM 语义对齐

### 8.2 构建兼容
- 支持 VS 2019/2022 的 C++ 桌面开发环境
- 支持 GitHub Actions 自动构建 32/64 位 Release

### 8.3 命令行兼容
- 保持原始 NSSM 命令语义不破坏
- 官方命令功能应映射到现有参数与代码逻辑

## 9. 运行时行为规范

### 9.1 服务启动
- 若程序以服务模式运行，调用 SCM `StartServiceCtrlDispatcher`。
- 完成服务主线程注册后进入控制回调。

### 9.2 服务停止
- 不直接强制杀进程，而是按阶段执行：Console -> Window -> Threads -> TerminateProcess
- 失败时记录日志，并回调退出动作。

### 9.3 日志轮转
- 轮转触发条件可组合：按时间、按大小、按需手动
- 仅在线轮转启用时，允许运行中切分日志

## 10. 测试要求

### 10.1 功能测试
- 安装、启动、停止、删除服务
- 获取/设置/卸载参数
- 退出动作顺序
- 日志轮转
- 建立或更新环境变量
- 账号配置

### 10.2 稳定性测试
- 长时间运行验证
- 重启/异常退出验证
- 资源泄漏检查

### 10.3 回归测试
- 32 位与 64 位构建
- GUI 与 CLI 一致性检查
- 中英文消息资源一致性检查

## 11. 约束与风险

### 11.1 约束
- 依赖 Windows 平台的专有 APIs
- 管理员权限不可绕过
- 需兼容旧版工程配置和构建流程

### 11.2 风险
- 注册表配置错误可能导致服务不可启动
- 网络/用户权限问题会影响服务账号配置
- 日志轮转和程序直接写文件可能出现冲突
- 事件钩子脚本执行失败需妥善回退与日志记录

## 12. 结论

NSSM 的技术实现本质上是一个 Windows 服务管理引擎：它以服务控制管理器作为基础能力层，以注册表和环境变量为参数层，以清晰模块化设计承载功能扩展。

在本仓库中，技术规格的重点是：改进稳定性、修正兼容性问题、提升维护可读性、落实中文用户体验，并保持对原始 NSSM 的命令行兼容性。该体系为后续功能扩展和工程化迭代提供了清晰基础。
