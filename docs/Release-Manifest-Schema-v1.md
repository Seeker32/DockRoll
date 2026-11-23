---
schema_version: 1
id: ota-release-manifest-v1
type: technical
title: 发布清单 Schema v1.0 与校验规则
summary: 冻结 manifest.json v1.0 的字段、必填/可选规则、digest 与哈希校验、密钥引用、资源归属、存量接管和版本演进策略。
status: active
lang: zh-CN
owners: [Seeker32]
tags: [ota, m0, release, schema, contract]
updated_at: 2026-10-24
order: 30
area: ota
---

# 发布清单 Schema v1.0 与校验规则

## 状态与证据

2026-10-24 完成 `manifest.json` v1.0 的 schema、字段规则、机器校验器、合法/非法样例和评审记录。本页是**规则说明和实现契约**；可执行的 JSON Schema、命令和样例位于证据目录，错误输出与哈希见[发布清单 Schema 验证记录（2026-10-24）](../evidence/Release-Manifest-Schema-2026-10-24.md)。

| 交付物 | 位置 |
| --- | --- |
| JSON Schema（Draft 2020-12） | [manifest.schema.json](../../verification/release-schema-2026-10-24/manifest.schema.json) |
| 校验器（结构 + 语义规则） | [validate_manifest.py](../../verification/release-schema-2026-10-24/validate_manifest.py) |
| Python 依赖 | [requirements.txt](../../verification/release-schema-2026-10-24/requirements.txt)（jsonschema 4.26.0、cryptography 50.0.2） |
| 一键回归命令 | [run-validation.sh](../../verification/release-schema-2026-10-24/run-validation.sh) |
| 合法样例（三类） | [samples/valid](../../verification/release-schema-2026-10-24/samples/valid/) |
| 非法样例（32 类） | [samples/invalid](../../verification/release-schema-2026-10-24/samples/invalid/) |
| 完整校验输出 | [validation-2026-10-24.txt](../../verification/release-schema-2026-10-24/output/validation-2026-10-24.txt) |
| 评审记录 | [发布清单 Schema v1.0 评审记录](../evidence/Release-Manifest-Schema-Review-2026-10-24.md) |
| Story | [发布清单 schema v1 与校验](../stories/Story-Release-Manifest-Schema.md) |

本页冻结的是**发布目标和执行契约**，不包含业务数据迁移或数据库变更承诺；也不实现 M1 的下载、Docker/Compose 执行器或签名信任根。签名信任、公钥预置/轮换和回滚授权由[状态机、恢复与安全设计](../stories/Story-Recovery-Security-Design.md)定义。

## 命名约定

- JSON 字段名统一使用 **camelCase**（如 `schemaVersion`、`appId`、`sharedResources`、`readOnly`）。
- 内部 `$defs` 名称同样使用 camelCase（如 `bindSource`、`healthHttp`）。
- 枚举值保持不变（如 `single_container`、`bind_path`、`adopt_existing`、`previous_release`），避免把字段命名风格变化扩散为值语义变化。
- 错误路径、文档字段表和样例使用同一套 camelCase 名称。

## 范围与边界

- 一个发布版本对应一个不可变的 `manifest.json`；修改必须产生新的 `releaseId`，不允许原地改写。
- 发布类型只有三种：`single_container`、`compose`、`config`。每个 manifest 必须且只能携带与 `releaseType` 对应的一个载荷块（`container`、`compose` 或 `config`）。
- manifest 只描述目标状态和校验/执行参数，不在清单内内联配置内容、镜像层或密钥值；镜像由 registry 分发，配置/Compose 文件由制品地址分发。
- 未知字段一律拒绝；字段白名单是 OTA 预检查的一部分，不依赖 Docker/Compose 的错误文本。
- Compose 文件本身的字段子集（禁止 `build`、远程 `include`、`profiles` 的默认激活等）由[三类参考发布包与 Compose 规则](../stories/Story-Release-Packages.md)定义；本页只冻结 manifest 如何引用 Compose 文件、服务集合、镜像 digest 和共享资源。
- 校验器不拉取镜像、不启动容器、不修改宿主机；M1 在下载阶段按 digest 拉取，并按本页规则做预检查。

## 文件布局与命令

参考发布包布局：

```text
release/
├── manifest.json
├── compose.yaml          # releaseType=compose
├── configs/              # releaseType=config 的源文件
└── payload/              # 其他受哈希约束的制品
```

`manifest.json` 中的 `path` 一律相对 manifest 所在目录，禁止绝对路径和 `..`；校验器解析后必须仍位于发布目录内。

```bash
cd verification/release-schema-2026-10-24
python3 -m pip install -r requirements.txt   # 首次运行
python3 validate_manifest.py samples/valid/single-container/manifest.json
python3 validate_manifest.py samples/valid/compose/manifest.json
python3 validate_manifest.py samples/valid/config/manifest.json
python3 validate_manifest.py samples/invalid/inline-secret/manifest.json   # 退出码 1
./run-validation.sh                                                        # 全部样例回归
```

校验器退出码：`0` 全部通过；`1` 至少一个 manifest 被拒绝；`2` 参数、schema 文件或可信公钥无法加载。每条错误格式为 `$.字段路径: 原因 [规则 ID]`，便于日志定位和回归断言；同一文件可同时输出 schema 和语义错误。

样例中的镜像 digest 是固定测试值，只用于结构和签名流程校验，不对应可拉取镜像；生产 manifest 必须填写 registry 中的真实 digest。

## 顶层字段

| 字段 | 必填 | 类型 | 规则与默认值 |
| --- | --- | --- | --- |
| `schemaVersion` | 是 | string | `major.minor`；v1.0 只接受 `1.0`；更高 minor 在校验字段前拒绝 |
| `releaseId` | 是 | string | `^[a-z0-9][a-z0-9._-]{2,63}$`；同一 `appId` 内唯一且不可变 |
| `appId` | 是 | string | `^[a-z][a-z0-9-]{2,31}$`；必须等于 `ownership.composeProjectName` |
| `releaseType` | 是 | enum | `single_container` / `compose` / `config` |
| `createdAt` | 是 | string | UTC 时间，格式 `YYYY-MM-DDTHH:MM:SSZ` |
| `metadata` | 否 | object | 可选；`appVersion`（semver）、`description`（≤256） |
| `compatibility` | 是 | object | 见[兼容性条件](#兼容性条件) |
| `ownership` | 是 | object | 见[资源归属与映射](#资源归属与映射) |
| `takeover` | 否 | object | 存量接管；缺省等价 `mode: none` |
| `sharedResources` | 否 | array | 受保护的共享卷/网络/宿主机路径；默认空数组；最多 32 项，见[共享资源声明](#共享资源声明) |
| `container` | 条件必填 | object | `releaseType=single_container` 时必填 |
| `compose` | 条件必填 | object | `releaseType=compose` 时必填 |
| `config` | 条件必填 | object | `releaseType=config` 时必填 |
| `secrets` | 否 | array | 受控密钥引用；默认空数组；禁止任何内联值 |
| `applyPolicy` | 是 | object | 配置生效策略；见[配置生效](#配置生效策略) |
| `health` | 是 | object | 健康检查与观察窗口 |
| `rollback` | 是 | object | 回滚策略 |
| `signature` | 是 | object | 签名信封；见[签名](#签名) |

`container`、`compose`、`config` 三者互斥；同时出现两个或一个都没有都会被拒绝。`releaseId` 在 `appId` 内的唯一性和历史版本引用由发布服务维护；本校验器只校验单个 manifest 的格式和内部引用关系。

### 兼容性条件

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `os` | 是 | 非空数组，当前只允许 `linux` |
| `arch` | 是 | 非空数组，枚举 `amd64`、`arm64`（与设备基线一致） |
| `minAgentVersion` | 是 | semver，Agent 最低版本 |
| `minDockerVersion` | 是 | semver，Docker Engine 最低版本 |
| `minComposeVersion` | 是 | semver，且不得低于已冻结的 Compose 最低边界 `5.0.1`；低于该值直接拒绝 |

这些字段声明**发布要求**，不代表已在设备上验证。设备端是否满足由 M1 预检查对照[运行时兼容性支持矩阵](Compatibility-Matrix.md)判定；未验证组合不自动准入。

### 资源归属与映射

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `ownership.owner` | 是 | 小写团队/责任标识，`^[a-z0-9][a-z0-9-]{1,31}$` |
| `ownership.composeProjectName` | 是 | 必须等于 `appId`；Compose 项目名、容器 label、配置根目录共用该映射 |
| `ownership.labels` | 是 | 必须包含 `io.dockroll.managed=true`、`io.dockroll.app-id=<appId>`、`io.dockroll.release-id=<releaseId>`；最多 16 个键；所有键必须匹配 `^[a-z0-9][a-z0-9._-]{0,127}$`（ASCII，避免 Docker 和 JCS 差异） |

映射规则：**一个 `appId` 固定对应一个 Compose project name，且两者字符串相同**。配置文件写到 `/etc/dockroll/apps/<appId>/`；容器/Compose 资源必须带上述标签。这样回滚、orphan 清理和诊断都能按 `appId` 过滤，不需要猜测项目名。

### 存量接管与服务增删

`takeover` 只在 `compose` 发布中允许，用于把已经存在的 Compose 栈纳入管理：

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `mode` | 是 | `none` / `adopt_existing` / `replace_existing` |
| `existingProjectName` | 条件必填 | `adopt_existing`、`replace_existing` 时必填，且必须等于 `appId`；项目名不一致时须先走单独授权的改名操作 |
| `existingServices` | 条件必填 | 接管时已观察到的完整服务集合；用于计算新增/删除服务 |
| `preserveSharedResources` | 否 | 引用的名称必须出现在 `sharedResources[]` 中 |

`compose.services[]` 是**目标完整服务集合**，不是增量。服务增删规则：

- 新增服务 = `compose.services[]` − `takeover.existingServices[]`。
- 删除服务 = `takeover.existingServices[]` − `compose.services[]`；删除是有意的目标状态差异，不需要额外的 `remove_services` 字段。
- 回滚按回滚目标的 manifest 重新部署**完整集合**：恢复目标有、当前没有的服务，并删除目标没有、但属于同一 `appId` 的受管服务；不得留下占用端口的残留服务。
- `sharedResources[]`（`kind` 为 `volume`、`network` 或 `bind_path`，`protection` 固定 `preserve`）永远不因升级或回滚被删除；`takeover.preserveSharedResources` 必须与之一致。
- 存量接管前必须记录现有项目名、服务集合、共享资源和数据卷；当前盘点见[现有部署盘点](Existing-Deployment-Inventory.md)。

### 共享资源声明

`sharedResources[]` 声明不能由 OTA 删除的资源。`name` 是逻辑标识，`path` 只用于 `bind_path` 的真实宿主机路径：

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `kind` | 是 | `volume` / `network` / `bind_path` |
| `name` | 是 | 逻辑名，`^[a-zA-Z0-9][a-zA-Z0-9_.-]{0,63}$`；`takeover.preserveSharedResources` 引用这个名称 |
| `path` | `bind_path` 必填 | 宿主机绝对路径，规范化、不能是 `/`、不能含 `..`；`volume`/`network` 禁止该字段 |
| `protection` | 是 | 固定 `preserve`；升级、回滚和垃圾回收都不得删除 |

数组最多 32 项；`name` 必须在所有 kind 之间**全局唯一**，否则 `takeover.preserveSharedResources` 按 name 引用会有歧义。`bind_path` 的 `name` 不再承担路径含义，执行器使用 `path` 映射真实路径。示例：

```json
"sharedResources": [
  {"kind": "volume", "name": "shared-data", "protection": "preserve"},
  {"kind": "bind_path", "name": "demo-single-config", "path": "/srv/demo-single/config", "protection": "preserve"}
]
```

## 单容器载荷

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `image` | 是 | `repository` 不含 tag，`digest` 为 `sha256:<64 位小写十六进制>`；可选 `platform`，其架构必须属于 `compatibility.arch` |
| `command` | 否 | 非空字符串数组；覆盖镜像 ENTRYPOINT |
| `args` | 否 | 字符串数组；覆盖镜像 CMD |
| `env` | 否 | 数组；每项 `name` + 恰好一个 `value` 或 `secretRef`；`secretRef` 必须在 `secrets[]` 声明 |
| `ports` | 否 | `hostIp`、`hostPort`、`containerPort`、`protocol` 必填；`hostIp` 必须是合法 IPv4/IPv6，不再有隐式默认值 |
| `mounts` | 否 | `type` 为 `bind`/`volume`，`source`、`target`、`readOnly` 必填；bind 源必须命中[bind mount 安全边界](#bind-mount-安全边界)；容器目标必须是绝对路径、规范化、不含 `..` 且不能是 `/` |
| `network` | 是 | `mode` 为 `bridge`/`host`/`none`；`bridge` 可带 `name`；`host`/`none` 禁止 `name`，且 `container.ports` 必须为空 |
| `restartPolicy` | 是 | `name` 为 `no`/`on-failure`/`always`/`unless-stopped`；`on-failure` 必须显式给出 `maxRetries`（1～10），其他策略禁止该字段，避免“无限重试”歧义 |
| `resources` | 是 | 整数单位：`cpuMillicores`、`memoryBytes`、`pidsLimit`，不允许浮点 |
| `stopGracePeriodSeconds` | 是 | 显式给出 1～300 秒；不再使用 schema default |

`volume` 的 source 使用 `$defs.volumeName`，`network.name` 使用 `$defs.networkName`；两者不再复用同一个定义，避免“volume source 叫 networkName”的误导。

数组/对象上限（防止预认证阶段解析无界输入）：

| 字段 | 上限 | 字段 | 上限 |
| --- | --- | --- | --- |
| `container.command` | 32 | `container.args` | 128 |
| `container.env` | 128 | `container.ports` | 64 |
| `container.mounts` | 64 | `compose.services` | 32 |
| `compose.envFiles` | 32 | `compose.profiles` | 8 |
| `config.files` | 128 | `secrets` | 64 |
| `sharedResources` | 32 | `ownership.labels` | 16 |
| `takeover.existingServices` | 64 | `takeover.preserveSharedResources` | 32 |
| `health.check.command` | 32 | `applyPolicy.reload.command` | 32 |

单份 `manifest.json` 的预解析大小上限为 **1 MiB（1,048,576 字节）**；校验器在读入 JSON 前检查文件大小，超过即拒绝，避免在验签前完整解析任意大输入。

校验器现在会**同时报告 schema 错误和语义错误**：即使缺少必填字段，仍会继续检查哈希、密钥、挂载、时间戳等独立规则，避免一次只暴露一类问题。

### Bind mount 安全边界

Bind mount 源使用固定 allowlist，发布清单不能自行扩大：

| 源路径 | 读写 | 规则 |
| --- | --- | --- |
| `/etc/dockroll/apps/<appId>/...` | 只读 | `readOnly` 必须为 `true`；用于配置挂载 |
| `/var/lib/dockroll/apps/<appId>/...` | 可读写 | 应用数据目录 |
| `/srv/<appId>/...` | 可读写 | 应用数据目录 |
| `/opt/<appId>/...` | 可读写 | 应用数据目录 |

`<appId>` 必须是当前 manifest 的 `appId`，不能借用其他应用目录。以下路径不在 allowlist 中，必须拒绝：

- `/var/run/docker.sock`（Docker socket，禁止挂载；执行器应使用 Docker SDK）
- `/etc`、`/var/run`、`/proc`、`/sys`、`/dev`、`/boot`
- `/usr`、`/bin`、`/sbin`、`/lib`、`/lib64`、`/root`、`/home`
- `/var/lib/docker`、`/var/lib/containerd`

schema 用 `$defs.bindSource` 表达字符串层 allowlist；校验器再检查 `<appId>` 归属、配置挂载只读、Docker socket 和敏感路径。M1 还必须在宿主机上对路径做 `realpath` 后再校验，防止符号链接把允许路径指向 `/etc` 或其他应用目录；仅做字符串前缀判断不够。

Compose 文件中的 bind mount 不会被当前 manifest 校验器解析；M1 预检查必须对 `compose config` 输出的每个 bind source 套用同一策略。`sharedResources.bind_path` 是保护声明，不是挂载声明；若它同时作为 bind source 使用，也受本策略约束。

## Compose 载荷

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `file` | 是 | 制品对象：相对 `path`、`sha256`、`sizeBytes`；校验器检查文件存在、大小和哈希 |
| `envFiles` | 否 | 制品数组；默认空数组 |
| `services` | 是 | 非空数组，每项 `name`（Compose service 名）、`image`（digest 固定）和 `required`（布尔，显式必填）；不再使用 schema default |
| `profiles` | 否 | 字符串数组；默认空数组；若使用 profile 必须在这里显式声明，不能假设默认激活 |

`compose.services[]` 与 `compose.yaml` 必须表达同一个服务集合和镜像 digest；每个 service 的 `image.platform` 若声明，也必须属于 `compatibility.arch`；本页的校验器只验证 manifest 侧和文件哈希，**Compose 文件语义解析、字段子集和 `compose config` 校验属于 Story 4/M1 预检查**。Compose 文件中的镜像也必须按 digest 固定，不能用 tag。

## 配置载荷

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `files` | 是 | 非空数组；每项包含 `source` 制品、`targetPath`、`mode`、`uid`、`gid` |
| `source` | 是 | 相对路径 + `sha256` + `sizeBytes`；文件内容不内联 |
| `targetPath` | 是 | 绝对路径，必须位于 `/etc/dockroll/apps/<appId>/` 下，规范化且不含 `..`；任意两个 `targetPath` 不能互为文件前缀 |
| `mode` | 是 | 四位八进制字符串（如 `0640`）；拒绝 world-writable（其他用户可写） |
| `uid` / `gid` | 是 | 非负整数；落盘时按此设置属主 |

配置目录通过 bind mount 暴露给容器；单文件挂载/重建行为由 `applyPolicy` 指定。配置源文件按哈希和大小校验通过后才允许落盘。

## 密钥与内联禁止

`secrets[]` 只能声明**引用**：

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `name` | 是 | 逻辑名称；供 `env[].secretRef` 引用；同名重复被拒绝 |
| `provider` | 是 | `file` / `env` / `docker_secret` |
| `ref` | 是 | `file` 必须位于 `/run/dockroll-secrets/`；`env` 必须是大写环境变量名；`docker_secret` 必须是合法 Docker secret 名 |

违规示例（均被拒绝）：

```json
{"secrets": [{"name": "DB_PASSWORD", "provider": "file", "ref": "/run/dockroll-secrets/db_password", "value": "s3cr3t"}]}
```

错误：`$.secrets[0].value: inline secret value is forbidden; declare a controlled ref instead [secret.inline]`。

```json
{"container": {"env": [{"name": "DB_TOKEN", "secretRef": "DB_TOKEN"}]}}
```

若 `secrets[]` 没有声明 `DB_TOKEN`，错误：`$.container.env[0].secretRef: secretRef 'DB_TOKEN' is not declared in secrets[] [secret.ref-unknown]`。

密钥值不得出现在 manifest、Compose 文件、普通日志或诊断输出中；发布样例只使用占位引用。实际密钥注入和本地权限由 M1/安全设计落实。

## 镜像、制品与哈希规则

- 镜像必须 `repository` + `digest` 分离表达，digest 形如 `sha256:<64 位小写十六进制>`；`repository` 中不得出现 tag，`@` 也不允许。校验失败示例：`$.container.image.repository: image repository must not contain a tag; pin the image with digest [image.tag-forbidden]`。
- 制品 `path` 必须是相对路径、字符集受限、不能是绝对路径或含 `..`；解析后的真实路径必须仍在发布目录内（防路径穿越）。
- 制品必须存在，且实际 `sizeBytes` 和 `sha256` 与声明一致。失败示例：`$.config.files[0].source.sha256: sha256 mismatch: declared 0000…, actual … [artifact.hash-mismatch]`；`$.config.files[0].source.sizeBytes: size mismatch: declared 97, actual 96 [artifact.size-mismatch]`。
- 本 Story 的校验器不拉取镜像，也不验证 registry 端 digest 是否存在；M1 下载时按 digest 拉取并校验，镜像内容以 registry digest 为准。
- 配置/Compose 文件的实际内容哈希在发布构建时生成并签名；设备端在激活前重新计算，任何不一致都拒绝。

## 健康检查与观察窗口

`health.check` 三选一：

| 类型 | 必填字段 | 说明 |
| --- | --- | --- |
| `http` | `path`、`port`、`scheme`、`expectedStatus` | `expectedStatus` 可填 100～599 的整数或 `2xx`/`3xx`/`4xx`/`5xx` 类；可选 `service`，仅 Compose 使用 |
| `tcp` | `port` | 可选 `service` |
| `command` | `command`（非空数组） | 可选 `service` |

公共字段：`intervalSeconds`（1～300）、`timeoutSeconds`（1～120，且必须小于 `intervalSeconds`）、`retries`（1～10）、`startupGraceSeconds`（0～600）、`observationWindowSeconds`（10～3600）、`successCriteria`（`all_services_healthy` / `primary_service_healthy`）。Compose 多服务使用 `primary_service_healthy` 时必须显式给出 `check.service`。

`health.check.port` 表示**容器内端口**，从应用网络内部探测，不要求该端口已发布到宿主机；`expectedStatus` 支持 `204` 这类精确值和 `2xx` 类匹配。`observationWindowSeconds` 是健康观察时间；`rollback.deadlineSeconds` 从观察判定失败后开始计时，不包含观察窗口，因此两者不强制大小关系，但总恢复时间应按设备 RTO 单独核算。

只有健康检查在观察窗口内持续满足条件，才允许提交成功；失败进入回滚。检查命令由应用方提供，校验器只校验结构和引用关系。

## 时间戳语义

`createdAt` 和 `signature.signedAt` 的 schema pattern 只保证形状；校验器还会：

- 用 UTC 日历解析，拒绝 `2026-13-45T99:99:99Z` 这类“格式对但日期非法”的值；
- 要求 `signedAt >= createdAt`，防止签名时间早于清单创建时间。

重放防护（签名时效、nonce、设备时钟）仍由安全设计定义，不在本层实现。

## 配置生效策略

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `applyPolicy.strategy` | 是 | `reload` / `restart_container` / `recreate_container` |
| `applyPolicy.reload.command` | 条件必填 | `strategy=reload` 时必填，非空字符串数组 |
| `applyPolicy.reload.timeoutSeconds` | 条件必填 | 1～300 |
| `applyPolicy.reload.service` | 否 | 仅 Compose；必须在 `compose.services[]` 中 |

`strategy` 非 `reload` 时禁止出现 `reload` 块，避免声明与实际执行不一致。目录挂载 + reload 与单文件挂载/重建两类行为的完整样例由 Story 4 演示；本 Story 冻结枚举和条件约束。

## 回滚策略

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `strategy` | 是 | `previous_release` / `explicit_release` |
| `targetReleaseId` | 条件必填 | `explicit_release` 时必填，且不能等于当前 `releaseId`；`previous_release` 时禁止 |
| `maxAttempts` | 是 | 1～3 |
| `failureAction` | 是 | `escalate`（停止自动切换并转人工）/ `keep_failed` |
| `preserveVolumes` | 是 | 固定 `true`；禁止删除业务数据卷 |
| `restoreServiceSet` | 是 | 固定 `true`；回滚必须恢复目标的完整服务集合 |
| `deadlineSeconds` | 是 | 30～1800；超时进入人工处置 |

回滚目标是完整 manifest，不是一个增量补丁。回滚后必须核对：服务集合、端口占用、共享资源、数据卷和健康状态；失败时保留诊断，不自动再次切换。

## 签名

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `algorithm` | 是 | `ed25519` / `ecdsa-p256-sha256` |
| `keyId` | 是 | 稳定密钥标识 |
| `signedAt` | 是 | UTC 时间 |
| `canonicalization` | 是 | 固定 `RFC8785` |
| `signature` | 是 | Base64 编码签名值；Ed25519 固定 64 字节 |

签名对象是**删除 `signature.signature` 后的 manifest 规范化字节**；`algorithm`、`keyId`、`signedAt`、`canonicalization` 都在签名保护范围内。规范化采用 RFC 8785 JCS 的受限子集：只允许对象、数组、字符串、整数、布尔和 null，禁止浮点数（本 schema 的所有数值字段都是整数），对象键按 UTF-16 code unit 排序（与 RFC 8785 一致）。`ownership.labels` 的键进一步限制为 ASCII，避免 Docker label 格式和跨语言排序差异。签名者对删除 `signature.signature` 后的规范化字节签名；校验器输出的 `manifest_digest=sha256:<…>` 是这些字节的 SHA-256，用于引用和比对，不是另行签名的对象。

本 Story 只冻结信封和规范化规则；签名信任根、公钥预置/轮换/过期、重放防护由安全设计定义。`--trusted-key` 仅用于测试样例验证，样例公钥是 **test fixture key，不是生产信任根**。生产发布不得使用样例密钥。

## 未知字段、默认值与版本演进

未知字段处理：所有已定义对象都设置 `additionalProperties: false`；`ownership.labels` 是唯一允许扩展标签的映射。错误示例：`$.unexpected_release_note: unknown field 'unexpected_release_note' is not allowed [schema.additionalProperties]`。

默认值汇总：

Schema **不再声明 `default` 注解**，避免 Agent 各自实现默认值导致分叉：

| 字段 | 缺省语义 |
| --- | --- |
| `takeover` | 整个块缺省等价于 `mode: none`；出现块时 `mode` 必填 |
| `compose.services[].required` | 必填布尔，不再有隐式 `true` |
| `container.ports[].hostIp` | 必填，必须是合法 IPv4/IPv6 |
| `container.stopGracePeriodSeconds` | 必填整数 |
| `compose.envFiles` / `compose.profiles` / `secrets` | 缺省按空数组处理；数组本身不写 `default` |
| `sharedResources` / `metadata` | 缺省不存在 |

版本演进规则：

1. **Major（如 1.0 → 2.0）**：新增必填字段、删除/重命名字段、收窄类型或枚举、改变字段语义、改变签名规范化方式。设备必须拒绝不支持的 major；不承诺向后兼容。
2. **Minor（如 1.0 → 1.1）**：只允许新增可选字段或新增不影响 v1.0 语义的校验规则；设备只接受显式支持列表中的版本。旧设备遇到更高 minor 必须在字段校验前拒绝，并报告“不支持的 schema 版本”，而不是把它当作未知字段误报。
3. 在受支持的 major.minor 内，未知字段始终拒绝；不能以“向前兼容”为由放宽字段白名单。
4. 文档澄清、拼写修正且不改变 schema 字节和校验结果时不升级版本；一旦改变可观察校验行为，至少升 minor。
5. 已发布 manifest 不可原地修改；任何目标变化都必须生成新 `releaseId` 并重新签名。回滚引用旧 manifest，不修改旧文件。

## 与后续 stories 的交接

- [Story 4：三类参考发布包与 Compose 规则](../stories/Story-Release-Packages.md) 复用本页 schema 和 `samples/`，补齐单容器/Compose/配置三类完整包、Compose 字段子集、资源归属与接管步骤；本页的合法样例可作为其 manifest 基线，非法样例可共同维护。
- M1 实现时应把 `manifest.schema.json`、校验器和样例**复制/固定到 DockRoll 代码仓库**（或作为构建依赖），锁定 schema SHA-256 与 Python 依赖版本；设备端预检查不得依赖运行时下载 schema。
- 安全设计负责签名信任根和公钥生命周期；M1 负责下载、按 digest 拉取、按哈希/大小校验和激活前拒绝。
- 本页不实现 Compose 执行器、配置 reload 或回滚执行；这些从 M1 开始。

## 评审与未决事项

技术评审记录见[发布清单 Schema v1.0 评审记录](../evidence/Release-Manifest-Schema-Review-2026-10-24.md)，样例和校验输出已登记到[M0 证据索引](../evidence/M0-Evidence-Index.md)。未决事项：

| 事项 | Owner | 期限/门禁 |
| --- | --- | --- |
| 生产签名信任根、公钥分发/轮换/过期、重放防护 | 安全/平台 owner（Seeker32 暂代） | M0 安全设计评审 / M1 签名校验实现前 |
| Compose 文件字段子集、`compose config` 解析、与 manifest 服务集合的一致性检查 | Platform Engineer | Story 4 交付 / M1 预检查 |
| schema 与校验器进入 DockRoll 仓库并固定依赖/SHA-256 | Agent Engineer | M1 开始前 |
| registry 拉取按 digest 的实际校验与失败分类 | Agent Engineer | M1 下载/校验实现 |
| 真实设备上的配置 reload/重建行为验证 | Platform Engineer | M1/M3 真实设备验证 |
