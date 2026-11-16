# BB10 Dropbox 复用评估与交付计划

[English](REUSE-ASSESSMENT.md) | 简体中文

评估日期：2026-11-16。状态：可行性评估已完成，尚未开始实现。

## 结论

针对“手动传输＋自动双向文件夹同步”的完整需求，不建议以 Basket 作为应用代码基础。建议使用已有 BB10 工具链和许可证明确的平台组件，新建一个精简的 Qt/Cascades 原生客户端。在复用授权明确之前，将 Basket 作为功能行为参考。

Basket 的原生界面和 Dropbox API v2 封装有价值，但现代授权、可靠传输、持久化同步引擎和后台恢复，都需要大量新增或替换代码。修复旧行为的成本会叠加在新客户端同样需要完成的核心工作之上。按照你提出的“与从零开始工作量差不多就不借用”的标准，目前没有充分证据证明采用它能明显节省工程量。

这一建议针对完整产品。如果只恢复旧式文件浏览和手动传输，且复用授权明确，采用旧项目可能更划算。这并不表示现有代码全部没有价值。

## 范围与证据

主要上游快照：

| 仓库 | 确切提交 | 职责 |
| --- | --- | --- |
| [doctorrokter/basket](https://github.com/doctorrokter/basket/tree/3d20d69b8123397dcd5a0a1ceb4e8ebc54814333) | `3d20d69b8123397dcd5a0a1ceb4e8ebc54814333` | 原生界面、文件操作、元数据缓存 |
| [doctorrokter/qdropbox](https://github.com/doctorrokter/qdropbox/tree/6c35d03aec46838d003c07f7311ceda6fbdd67c9) | `6c35d03aec46838d003c07f7311ceda6fbdd67c9` | Qt 4.8 的 Dropbox API v2 封装 |
| [doctorrokter/basket_service](https://github.com/doctorrokter/basket_service/tree/69350174fa11238cfab9fba9486c1d903612b8ae) | `69350174fa11238cfab9fba9486c1d903612b8ae` | 无界面后台服务和相机上传 |

已检查仓库文件树、构建和打包声明、授权界面、应用与控制器流程、传输实现、元数据缓存与轮询、服务队列及相机索引。选取的源码快照位于 `/tmp/bb10-dropbox-assessment/`；这些临时副本不是完整可构建的检出。未修改上游原始源码。

Shell 克隆因无法解析 github.com 而失败，随后改用已连接的 GitHub 工具读取源码。BB10 开发技能的 `python3 scripts/doctor.py --profile native --probe` 返回退出码 2：`Image unavailable: bb10-native-dev:local`。没有进行目标编译、BAR 安装、真实账户授权或设备测试。编译与运行兼容性仍未验证。本次是采用决策与工作量评估，不是代码审查通过或安全认证。

## 影响决策的发现

| 方面 | 上游实际情况 | 必须完成的工作 |
| --- | --- | --- |
| 构建与打包 | Basket 及服务均引用仓库中不存在的 `config.pri`。打包依赖相邻目录里的 `BasketService` 二进制。服务源码单独位于 `basket_service`，并非完全缺失。 | 补齐构建配置、目录与输出布局及打包流程，验证 ARM 链接和启动。 |
| 授权 | 授权页面要求从隐式授权 URL 手动复制 token；通过 QSettings 保存。所查封装没有 PKCE 或刷新令牌生命周期。 | 注册新应用，实现 PKCE 授权码流程、过期刷新与撤销处理、凭据存储和日志脱敏。 |
| TLS | 使用 Qt 网络请求。仅凭源码不能确认兼容当前证书链和 TLS 协商。 | 在 Passport 上验证 API 与内容主机的证书及主机名检查，并验证无效证书会被拒绝；必要且可行时打包维护中的 TLS/CA 组件。 |
| 下载完整性 | `read()` 和 `readZip()` 向“下载目录＋文件名”追加内容，不检查打开和写入结果；网络成功便报告完成。 | 按完整映射区分目标，使用临时文件，检查写入和空间，验证大小/哈希后原子提交，避免同名冲突和重试损坏。 |
| 上传与恢复 | 已有 QFile 流式上传和分块上传会话。服务队列仅在内存中，上传或网络错误会调用 `dequeue()`。 | 持久化任务、有限重试与退避、会话恢复、取消和准确结果反馈；检查并测试分块边界。 |
| “同步”的含义 | 服务监听相机目录的新文件并上传到 `/Camera`；轮询器刷新云端元数据缓存。 | 新增文件夹配对同步引擎：递归状态、版本/哈希比较、冲突、删除历史和恢复。现有缓存轮询不是双向文件同步。 |
| 云端变化 | 轮询器处理目录继续查询时明确忽略 `hasMore`。 | 用于同步前，需要完整分页、游标失效恢复及游标/状态的持久化事务。 |
| 复用授权 | 三个主要仓库文件树未见项目级许可证文件，GitHub 也未识别出许可证。部分生成或模板文件带 Apache 声明，内置 QJson 文件带 LGPL 2.1 声明。 | 澄清实际应用与库代码的使用授权，保留各组件声明。模板声明不能证明整个项目的许可范围。 |

源码定位：

- 构建：[Basket.pro](https://github.com/doctorrokter/basket/blob/3d20d69b8123397dcd5a0a1ceb4e8ebc54814333/Basket.pro#L5)、[BAR 描述文件](https://github.com/doctorrokter/basket/blob/3d20d69b8123397dcd5a0a1ceb4e8ebc54814333/bar-descriptor.xml#L80)、[服务工程](https://github.com/doctorrokter/basket_service/blob/69350174fa11238cfab9fba9486c1d903612b8ae/BasketService.pro)。
- 授权：[授权页面](https://github.com/doctorrokter/basket/blob/3d20d69b8123397dcd5a0a1ceb4e8ebc54814333/assets/pages/AuthFlowPage.qml#L51)、[令牌保存](https://github.com/doctorrokter/basket/blob/3d20d69b8123397dcd5a0a1ceb4e8ebc54814333/src/applicationui.cpp#L238)。
- 传输：[下载写入](https://github.com/doctorrokter/qdropbox/blob/6c35d03aec46838d003c07f7311ceda6fbdd67c9/src/qdropbox/QDropbox.cpp#L534)、[上传和会话实现](https://github.com/doctorrokter/qdropbox/blob/6c35d03aec46838d003c07f7311ceda6fbdd67c9/src/qdropbox/QDropbox.cpp#L600)、[服务错误处理](https://github.com/doctorrokter/basket_service/blob/69350174fa11238cfab9fba9486c1d903612b8ae/src/service.cpp#L348)。
- 同步：[相机索引](https://github.com/doctorrokter/basket_service/blob/69350174fa11238cfab9fba9486c1d903612b8ae/src/service.cpp#L254)、[上传失败移出队列](https://github.com/doctorrokter/basket_service/blob/69350174fa11238cfab9fba9486c1d903612b8ae/src/service.cpp#L445)、[元数据轮询器](https://github.com/doctorrokter/basket/blob/3d20d69b8123397dcd5a0a1ceb4e8ebc54814333/src/cache/QDropboxPoller.cpp#L73)。

[Clouder](https://github.com/syedshaishad/Clouder) 只作为次要候选进行了筛选，没有开展同等深度的评估。它有根目录 Apache 许可证，但 [Pleaseread.txt](https://github.com/syedshaishad/Clouder/blob/master/Pleaseread.txt) 又限制复制和再制作；仓库最近推送在 2015 年。目前没有展示出能改变本次决策的优势。发现的四个 Basket fork 的推送日期也在 2018 年；这类元数据检查不能证明源码完全相同，也不能证明存在持续维护的替代品。

## 当前 Dropbox 要求

Dropbox 建议无法保密客户端密钥的原生应用使用 PKCE，需要离线访问时使用刷新令牌。第一版可考虑在另一台设备的现代浏览器完成授权，再把授权码交给 BB10 客户端；必须将 PKCE verifier 与对应授权过程关联，并避免写入日志。具体支持的流程需要在 M1 验证。参见[官方 OAuth 指南](https://docs.dropboxapi.com/dropbox-api/docs/oauth)。

Dropbox 公告 API 根证书变更从 2026 年 1 月 1 日或之后开始。这意味着需要验证实际证书链，不意味着所有 BB10 连接必定失败。参见[官方证书公告](https://dropbox.tech/developers/api-server-certificate-changes)。

限流时遵守 `Retry-After`，必要时使用上传会话，参见[官方性能指南](https://docs.dropboxapi.com/dropbox-api/docs/performance)。

## 工作量对比

以下是置信度较低的规划估算，以专注工作的工程师人日计算，不是已经测量的交付承诺。假设支持单账户、普通个人 Dropbox 文件夹、Passport 原生界面、手动传输、递归双向文件夹同步和后台调度。不包含等待设备/账户、恢复缺失工具链、澄清许可、SDK 再分发以及意外 TLS/平台限制带来的额外工作。

| 路线 | 粗估工作量 | 解释 |
| --- | --- | --- |
| 修复并采用 Basket，再补齐完整需求 | 16–28 人日 | 节省部分界面和 API 接线，但增加构建恢复、旧行为修复与整合成本，许可仍不明确。 |
| 使用成熟平台组件新建精简原生应用 | 15–25 人日 | 初始功能面更小，核心授权、传输和同步工作仍需完成。 |

两个区间高度重叠，没有证据表明采用 Basket 能明显节省工作。在验证新客户端核心可行性之前，不投入迁移整套旧界面、缓存和分享功能。M0 与 M2 完成后重新估算；下面的分阶段计划比总数更适合作为规划依据。

## 推荐路线的 Milestone

每个阶段交付带版本的 BAR 或探针产物、中英发布说明、检查记录及明确的设备证据。开始实现时，按选定构建方式初始化 project gate，提交前运行 fast，交接前运行 full。设备测试与主机检查、目标构建分开记录。

| Milestone | 工作量 | 交付物 | 验收条件 |
| --- | --- | --- | --- |
| M0 — 平台与连接验证 | 1–2 人日 | 可复现的最小 ARM BAR、本机存储和 HTTPS 探针 | 在 Passport 10.3.x 安装启动；读写用户选定的测试位置，有 SD 卡时包含 SD 卡；验证当前 Dropbox API/内容主机 TLS，并拒绝无效证书；记录准确系统、root 和安装环境。 |
| M1 — 账户授权 | 2–3 人日 | 原生账户连接、最小必要权限、刷新和断开连接 | 客户端不接收密码，完成 PKCE 授权；令牌过期和重启后正常恢复；撤销后停止访问；日志无秘密；建立受保护的凭据存储并说明 root 环境下的限制。 |
| M2 — 可靠手动传输 | 3–5 人日 | 文件夹浏览、文件选择、上传下载队列、进度/取消/重试 | 验证零字节、Unicode 文件名、嵌套路径和需要上传会话的大文件；校验字节完整性；不同目录同名文件不冲突；断网、磁盘满和重启不会以残缺文件替换有效文件。此时可交付实用的手动传输版本。 |
| M3 — 应用运行时双向同步 | 4–7 人日 | 文件夹配对、递归本地/云端状态、持久化任务、自动协调 | 两端新增/修改双向传播；同时修改保留两个版本；确认后的删除策略正确执行；离线/重启和游标失效不丢数据；SD 卡或存储缺失不被误判为大批删除；手动任务共用传输队列，并优先于待执行的自动任务。 |
| M4 — Passport 后台同步 | 3–5 人日 | 原生后台服务、调度、网络/电量控制和状态显示 | 在已测试系统条件下，关闭界面和熄屏后，变化在配置的间隔内完成同步；重启、断网、服务重启可恢复任务；重复运行不重复传输；记录电量和生命周期表现。 |
| M5 — 发布加固 | 2–3 人日 | 正式 BAR、可复现构建/检查、中英安装与恢复文档 | 在 Passport 完成冲突/删除/重试/存储故障矩阵、持续运行测试、令牌撤销和无效证书拒绝；升级保留账户与同步状态；检查最终权限及发布产物。 |

依赖关系：M0 → M1 → M2 → M3 → M4 → M5。M2 是可用的手动传输版本；M3 增加应用运行时的自动同步；完成 M4 后才能宣称支持后台同步。手动传输和自动双向同步均保留为最终要求。

## 停止条件与实现前决策

M0 是有投入上限的可行性验证。如果无法在计划的探针工作量内实现正确验证的直连 HTTPS 或必要的原生运行，应停止后续产品工作，报告具体障碍。原生客户端＋NAS 网关仍可作为备选，但切换架构需要重新估算并由用户决定。不得通过关闭证书校验使阶段验收通过。

M3 前确定删除传播与冲突呈现策略。建议初始策略：冲突保留两个版本；自动传播删除默认关闭，显式开启后保留删除/恢复记录。这只是建议，尚未经用户确认。写入应附带版本条件，避免扫描后、上传前的云端新修改被静默覆盖。

如果独立的 Dropbox 交换目录能满足需求，优先采用应用文件夹权限；访问任意已有 Dropbox 文件夹需要更广的内容访问权限。在应用注册前确认，不预先实现两套账户模式。整个账户镜像、团队管理、分享链接管理、媒体流播放和恢复旧系统文件管理器集成不包含在本估算内。

已经 root 且停止维护的系统，限制了应用对其他特权进程的防护能力。应用可以验证连接、限制权限并防止数据丢失，但无法承诺抵抗已失陷的系统。本次评估未连接手机，也未读取私人文件或账户。
