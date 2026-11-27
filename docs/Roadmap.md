# Roadmap

记录已从当前规范中移除、留待后续版本开发的功能。每项保留决策背景与设计基线，重新引入时以此为起点。

## 存量接管（takeover）

状态：**计划中，未排期**。已从 [Release-Manifest-Schema v1.0](Release-Manifest-Schema-v1.md) 中删除；v1.0 只覆盖全新部署和已受管应用的发版/回滚。

### 要解决的问题

设备上往往已经存在手工 `docker compose up` 部署的 Compose 栈。没有接管能力时，DockRoll 无法把这类栈纳入管理：要么放弃存量应用，要么要求先手动拆除再用 DockRoll 重新部署（服务中断、存在丢失状态的风险）。

### 删除前的设计基线

以下要点来自已删除的 schema 章节，重新实现时作为需求起点：

- `takeover` 是顶层可选对象，只允许出现在 `releaseType=compose` 的发布中；整个块缺省等价 `mode: none`。
- `mode` 三种取值：
  - `none`：不接管，全新部署或已受管应用正常发版。
  - `adopt_existing`：收编现有栈——容器原地保留，按目标服务集合增量收敛。
  - `replace_existing`：替换现有栈——按本清单目标状态重建。
- `existingProjectName`：`adopt_existing` / `replace_existing` 时必填，必须等于 `appId`；项目名不一致须先走单独授权的改名操作。
- `existingServices`：接管时必填，为接管前已观察到的完整服务集合；服务增删按 `compose.services[]` 与该集合的差集计算，删除是有意的目标状态差异，不需要单独的 `remove_services` 字段。
- `preserveSharedResources`（可选）：引用本清单 `sharedResources[]` 的逻辑名；未列入的资源仍按设备上已有和本次新增的保护记录处理。
- 上限：`existingServices` ≤ 64，`preserveSharedResources` ≤ 32。
- 接管前必须完成并持久化现有部署盘点（项目名、服务集合、共享资源、数据卷）；对应文档 Existing-Deployment-Inventory 尚未建立，需一并补齐。

v1.0 已改用[固定的资源保留规则](Release-Manifest-Schema-v1.md#资源保留规则)，不再接受上述保护名单。重新设计接管时，以固定保留规则和设备盘点为起点，不直接恢复旧字段引用。

### 重新引入时的待办

- [ ] schema：恢复顶层 `takeover` 字段表、"存量接管与服务增删"章节、数组上限与默认值条目
- [ ] 校验器：compose 专用条件校验
- [ ] 设备端：持久保存接管前的盘点，并按资源保留规则执行接管
- [ ] 执行器：定义 `adopt_existing`（原地保留）与 `replace_existing`（重建）的执行语义差异——schema 层从未定义，属开放设计点
- [ ] 流程：决定接管发布的服务删除（目标集合不含现有服务）是否需要单独的确认/授权步骤
