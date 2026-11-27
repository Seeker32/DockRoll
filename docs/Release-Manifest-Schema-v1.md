# 发布清单 Schema v1.0 与校验规则

状态：v1.0 候选规范，尚未冻结。

## 命名约定

- JSON 字段名统一使用 **camelCase**（如 `schemaVersion`、`appId`、`targetPath`、`readOnly`）。
- 内部类型与规则定义名称同样使用 camelCase（如 `bindSource`、`volumeName`、`networkName`、`healthHttp`）。
- 枚举值保持不变（如 `single_container`、`restart_container`、`previous_release`），避免把字段命名风格变化扩散为值语义变化。
- 错误路径、文档字段表和样例使用同一套 camelCase 名称。

## 范围与边界

- 一个发布版本对应一个不可变的 `manifest.json`；修改必须产生新的 `releaseId`，不允许原地改写。
- 发布类型只有三种：`single_container`、`compose`、`config`。`single_container` 必须携带 `container`、`compose` 必须携带 `compose`，两者禁止同时出现，但都可**可选**携带 `config`（用于初次部署或随应用版本一并下发受管配置文件）；`config` 为纯配置发布，必须携带 `config` 且禁止携带 `container` 和 `compose`。
- manifest 只描述目标状态和校验/执行参数，不在清单内内联配置内容、镜像层或密钥值；镜像由 registry 分发，配置/Compose 文件由制品地址分发。
- 未知字段一律拒绝；字段白名单是 OTA 预检查的一部分，不依赖 Docker/Compose 的错误文本。
- Compose 文件自身的字段子集（禁止 `build`、远程 `include`、`profiles` 默认激活等）由独立的 Compose 字段子集规范（M1 阶段补充）定义；[Compose 载荷](#compose-载荷)规定设备如何锁定输入、服务集合和镜像 digest。
- 校验器不拉取镜像、不启动容器、不修改宿主机；M1 在下载阶段按 digest 拉取，并按本页规则做预检查。
- 存量接管（把设备上已存在的 Compose 栈纳入管理）不在 v1.0 范围内，作为后续功能开发，见 [Roadmap](Roadmap.md)。

## 文件布局与校验流程

参考发布包布局：

```text
release/
├── manifest.json
├── compose.yaml          # releaseType=compose
├── configs/              # config 载荷的配置文件源文件
└── payload/              # Compose envFiles 等辅助制品
```

`manifest.json` 中的 `path` 一律相对 manifest 所在目录，禁止绝对路径和 `..`；校验器解析后必须仍位于发布目录内。

本仓库以 Go 校验包与 CLI 作为规范的权威实现，校验执行顺序如下：

1. **预解析大小检查**：读入 JSON 前检查文件大小不超过 **1 MiB（1,048,576 字节）**，超限立即拒绝。
2. **严格 JSON 解析**：拒绝重复对象键、非法 UTF-8、不成对 UTF-16 代理项、小数/指数记法、`-0` 及超出安全整数范围 `-9007199254740991`～`9007199254740991` 的数值。
3. **版本检查**：校验 `schemaVersion == "1.0"`；不支持的版本在字段校验前直接拒绝。
4. **验签与后续校验**：
   - **设备端（M1）**：必须**先完成 Ed25519 验签**，通过后再访问本地磁盘制品或解析 Compose 文件，防止对未认证输入做文件 I/O 或模型解析。
   - **离线校验 CLI**：提供 `--trusted-key` 时执行验签，并尽可能一次收集全部结构错误、语义错误与制品校验错误输出，便于发布流水线一次定位。

CLI 退出码：`0` 全部通过；`1` 至少一个 manifest 被拒绝；`2` 命令行参数或可信公钥无法加载。每条错误格式为 `$.字段路径: 原因 [规则 ID]`，规则 ID 采用 `<域>.<规则>` 命名（域前缀包括 `parse`、`version`、`signature`、`schema`、`image`、`artifact`、`mount`、`health`、`apply`、`rollback`），便于日志定位和回归断言。

字段示例：[单容器 manifest](examples/manifest-single-container.json)、[Compose manifest](examples/manifest-compose.json)、[纯配置 manifest](examples/manifest-config.json)。示例中的镜像 digest、制品元数据和签名是占位值；这些文件不能直接部署或通过验签。生成发布包时应填入实际制品和 registry digest，再对最终 manifest 签名。

## 顶层字段

| 字段 | 必填 | 类型 | 规则与默认值 |
| --- | --- | --- | --- |
| `schemaVersion` | 是 | string | `major.minor`；v1.0 只接受 `1.0`；更高 minor 在校验字段前拒绝 |
| `releaseId` | 是 | string | `^[a-z0-9][a-z0-9._-]{2,63}$`；同一 `appId` 内唯一且不可变 |
| `appId` | 是 | string | `^[a-z][a-z0-9-]{2,31}$`；同时作为 Compose project name 和配置根目录名，见[命名映射](#命名映射) |
| `releaseType` | 是 | enum | `single_container` / `compose` / `config` |
| `createdAt` | 是 | string | UTC 时间，格式 `YYYY-MM-DDTHH:MM:SSZ` |
| `metadata` | 否 | object | 可选；`appVersion`（严格 SemVer 2.0.0，禁止 `v` 前缀）、`description`（≤256 个 Unicode 字符） |
| `compatibility` | 是 | object | 见[兼容性条件](#兼容性条件) |
| `container` | 条件必填 | object | `releaseType=single_container` 时必填，其他类型禁止 |
| `compose` | 条件必填 | object | `releaseType=compose` 时必填，其他类型禁止 |
| `config` | 条件必填 | object | `releaseType=config` 时必填；`single_container` / `compose` 时可选，用于随应用部署一并下发受管配置文件 |
| `applyPolicy` | 是 | object | 生效策略；`single_container` / `compose` 固定为 `recreate_container`，纯配置发布作用于同一 `appId` 的现有部署，见[配置生效策略](#配置生效策略) |
| `health` | 是 | object | 健康检查与观察窗口 |
| `rollback` | 是 | object | 回滚策略 |
| `signature` | 是 | string | 标准 Base64 编码的 Ed25519 签名，解码后为 64 字节；见[签名](#签名) |

`container` 与 `compose` 互斥且必须与 `releaseType` 匹配：`releaseType=single_container` 必填 `container`、禁止 `compose`、可选 `config`；`releaseType=compose` 必填 `compose`、禁止 `container`、可选 `config`；`releaseType=config` 必填 `config`、禁止 `container` 和 `compose`。缺少必填载荷或出现不允许的载荷都会被拒绝。`releaseId` 在 `appId` 内的唯一性和历史版本引用由发布服务维护；本校验器只校验单个 manifest 的格式和内部引用关系。

### 兼容性条件

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `os` | 是 | 非空去重数组，当前只允许 `linux` |
| `arch` | 是 | 非空去重数组，枚举 `amd64`、`arm64`（与设备基线一致） |
| `minAgentVersion` | 是 | 严格 SemVer 2.0.0（禁止 `v` 前缀），Agent 最低版本 |
| `minDockerVersion` | 是 | 严格 SemVer 2.0.0（禁止 `v` 前缀），Docker Engine 最低版本 |
| `minComposeVersion` | 条件必填 | `releaseType=compose` 时必填，`single_container` / `config` 时可选；若提供须为严格 SemVer 2.0.0（禁止 `v` 前缀）且不得低于 Compose 最低边界 `5.0.1` |

这些字段声明**发布要求**，不代表已在设备上验证。设备端是否满足由 M1 预检查对照运行时兼容性支持矩阵（M1 阶段定义）判定；未验证组合不自动准入。

### 命名映射

**一个 `appId` 固定对应一个 Compose project name，且两者字符串相同**；配置文件写到 `/etc/dockroll/apps/<appId>/`。这样回滚、orphan 清理和诊断都能按 `appId` 过滤，不需要猜测项目名。

### 服务集合语义

`compose.services[]` 是**目标完整服务集合**，不是增量。部署回滚按目标部署状态恢复**完整集合**：恢复目标有、当前没有的服务，并删除目标没有、但属于同一 `appId` 的受管服务；不得留下占用端口的残留服务。纯配置发布的文件恢复见[回滚策略](#回滚策略)。

存量接管（把设备上已存在的 Compose 栈纳入管理）不在 v1.0 范围内，作为后续功能开发，见 [Roadmap](Roadmap.md)。

### 资源保留规则

升级、回滚和自动清理时，执行器只清理已确认由 DockRoll 管理、属于同一 `appId` 且目标状态不再需要的容器。设备不得自动删除卷（包括匿名卷）、网络、宿主机 bind mount 源目录及其中的持久数据。清理容器时不得连带删除卷或网络，也不得使用会删除项目网络的 [`docker compose down`](https://docs.docker.com/reference/cli/docker/compose/down/) 代替容器清理。

Compose 应用在 `compose.yaml` 中用 `external: true` 引用由运维预先创建的共享卷和网络；不再由发布清单声明保护名单。卷、网络和挂载目录的清理由运维独立执行。纯配置回滚仍按[回滚策略](#回滚策略)恢复受管配置文件，不清理挂载目录或其他数据。

## 单容器载荷

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `image` | 是 | 字符串：完整镜像引用，digest 必填、tag 可选，见[镜像引用](#镜像引用) |
| `platform` | 否 | 镜像运行平台，格式 `os/arch` 或 `os/arch/variant`（如 `linux/amd64`、`linux/arm64/v8`）；OS 必须属于 `compatibility.os`，架构必须属于 `compatibility.arch` |
| `command` | 否 | 非空字符串数组，每项须非空；覆盖镜像 ENTRYPOINT |
| `args` | 否 | 字符串数组（允许空数组 `[]` 清空 CMD）；覆盖镜像 CMD |
| `env` | 否 | 非敏感环境变量数组；每项必填 `name`（匹配 `^[A-Za-z_][A-Za-z0-9_]*$`，数组内不可重名）和 `value`（字符串，允许空串） |
| `ports` | 否 | `hostIp`、`hostPort`、`containerPort`、`protocol` 全必填；`hostIp` 必须是合法 IPv4/IPv6（不含 zone ID）；`hostPort` 与 `containerPort` 为 `1～65535`；`protocol` 为 `tcp`/`udp`；禁止重复 `(hostIp, hostPort, protocol)` 绑定 |
| `mounts` | 否 | `type` 为 `bind`/`volume`，`source`、`target`、`readOnly` 必填；bind 源必须命中[bind mount 安全边界](#bind-mount-安全边界)；volume 源匹配 `volumeName`；`target` 必须是规范化绝对路径、不含 `..`/`//`、无尾斜杠、不能是 `/` 且数组内不可重复 |
| `network` | 是 | `mode` 为 `bridge`/`host`/`none`；`bridge` 可带 `name`（匹配 `networkName`）；`host`/`none` 禁止 `name`，且 `container.ports` 必须为空；`none` 模式下禁止 `health.check.type` 为 `http` 或 `tcp` |
| `restartPolicy` | 是 | `name` 为 `no`/`on-failure`/`always`/`unless-stopped`；`on-failure` 必须显式给出 `maxRetries`（1～10），其他策略禁止该字段，避免“无限重试”歧义 |
| `resources` | 是 | 三个整数字段全部必填，不允许浮点：`cpuMillicores`（`1～64000`）、`memoryBytes`（`>= 6291456`，即 Docker 6 MiB 下限）、`pidsLimit`（`1～65535`） |
| `stopGracePeriodSeconds` | 是 | 显式给出 1～300 秒；不再使用默认值 |

`volumeName`（用于 `volume` 类型挂载的 `source`）与 `networkName`（用于 `network.name`）均独立定义，正则为 `^[a-zA-Z0-9][a-zA-Z0-9_.-]{0,63}$`，避免语义混用。

数组/对象上限（防止预认证阶段解析无界输入）：

| 字段 | 上限 | 字段 | 上限 |
| --- | --- | --- | --- |
| `container.command` | 32 | `container.args` | 128 |
| `container.env` | 128 | `container.ports` | 64 |
| `container.mounts` | 64 | `compose.services` | 32 |
| `compose.envFiles` | 32 | `compose.profiles` | 8 |
| `config.files` | 128 | `health.check.command` | 32 |
| `applyPolicy.reload.command` | 32 | | |

单份 `manifest.json` 的预解析大小上限为 **1 MiB（1,048,576 字节）**；校验器在读入 JSON 前检查文件大小，超过即拒绝，避免在验签前完整解析任意大输入。

### Bind mount 安全边界

Bind mount 源必须为规范化绝对路径（禁止 `..`、`//` 和尾部 `/`），并使用固定 allowlist，发布清单不能自行扩大：

| 源路径 | 读写 | 规则 |
| --- | --- | --- |
| `/etc/dockroll/apps/<appId>` 或其子路径 | 只读 | `readOnly` 必须为 `true`；用于配置挂载 |
| `/run/dockroll-secrets/<appId>/<file>` | 只读 | `readOnly` 必须为 `true`；仅允许单层文件名（禁止子目录，禁止挂载目录本身），见[应用凭据](#应用凭据) |
| `/var/lib/dockroll/apps/<appId>` 或其子路径 | 可读写 | 应用数据目录 |
| `/srv/<appId>` 或其子路径 | 可读写 | 应用数据目录 |
| `/opt/<appId>` 或其子路径 | 可读写 | 应用数据目录 |

`<appId>` 必须是当前 manifest 的 `appId`，不能借用其他应用目录。以下路径不在 allowlist 中，必须拒绝：

- `/var/run/docker.sock`（Docker socket，禁止挂载；执行器应使用 Docker SDK）
- `/etc`、`/var/run`、`/proc`、`/sys`、`/dev`、`/boot`
- `/usr`、`/bin`、`/sbin`、`/lib`、`/lib64`、`/root`、`/home`
- `/var/lib/docker`、`/var/lib/containerd`

清单校验器按 `bindSource` 规则检查字符串层 allowlist、`<appId>` 归属、配置与凭据只读、Docker socket 和敏感路径。M1 还必须在宿主机上对路径做 `realpath` 后再校验，防止符号链接把允许路径指向 `/etc` 或其他应用目录；仅做字符串前缀判断不够。

Compose 文件中的 bind mount 不会被当前 manifest 校验器解析；M1 预检查必须对 `compose config` 输出的每个 bind source 套用同一策略。

## Compose 载荷

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `file` | 是 | 制品对象：相对 `path`、`sha256`、`sizeBytes`（见[文件制品](#文件制品)）；校验器检查文件存在、大小和哈希 |
| `envFiles` | 否 | 制品数组，缺省为空数组；数组内 `path` 不可重复；按声明顺序提供插值变量，同名变量以后一个文件为准 |
| `services` | 是 | 启用的完整服务集合，非空且 `name`（匹配 `^[a-zA-Z0-9._-]{1,64}$`）不重复。每项必填 `name`、`image`（[完整镜像引用](#镜像引用)）和 `required`（布尔，参与[健康判定](#健康检查与观察窗口)）；可选 `platform`（格式与 OS/arch 校验同 `container.platform`） |
| `profiles` | 否 | 字符串数组，缺省为空数组；每项匹配 `^[a-zA-Z0-9._-]{1,64}$` 且不可重复；只启用显式声明的 profile |

本页的校验器只验证 manifest 侧和文件哈希；Compose 字段子集和有效模型由设备在 M1 预检查：

1. 使用 `compose.file`、已校验的 `compose.envFiles`、`compose.profiles` 和 `appId`（作为 Compose project name）显式解析有效模型。禁止继承设备环境中的插值变量或 Compose 控制变量，禁止自动读取 `.env` 或发现追加的 Compose 文件；发布的环境文件不得覆盖 Compose 控制参数，也不得包含真实凭据。未解析且无显式默认值的变量导致预检查失败。模型引用的发布包文件必须由 manifest 声明并校验；服务的 `env_file` 也只能引用 `compose.envFiles` 中的制品。设备上预置的应用凭据只允许通过受限的只读 bind mount 提供。
2. 按有效模型比对服务集合及每个服务的镜像：将 Compose 中的 `services.<name>.image` 与 manifest 中该服务的 `image` 解析为完整仓库名和 digest，两者都须一致，tag 不参与比较。任一侧缺少 digest 则拒绝。未绑定 profile 的服务及命中显式 profile 的服务纳入比对；未启用的服务不纳入。启用的依赖不完整则拒绝；不得靠指定单个服务隐式启用额外 profile。若 manifest 中该服务声明了 `platform`，其架构必须属于 `compatibility.arch`。
3. 部署和回滚必须使用对应版本预检查通过的有效模型，或用同一份已校验的输入和同样的解析参数得到相同模型后再执行。不得在执行时重新读取设备环境、隐式文件或未经校验的内容；[bind mount 安全边界](#bind-mount-安全边界)也以该模型为准。

Compose 的默认行为见 Docker 的[变量插值](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/)和 [profile](https://docs.docker.com/compose/how-tos/profiles/) 文档；DockRoll 用上述显式输入替代默认发现行为。

## 配置载荷

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `files` | 是 | 非空数组（上限 128）；每项包含 `source` 制品、`targetPath`、`mode`、`uid`、`gid` |
| `source` | 是 | 制品对象（见[文件制品](#文件制品)）；文件内容不内联 |
| `targetPath` | 是 | 绝对路径，必须位于 `/etc/dockroll/apps/<appId>/` 下（禁止等于根目录 `/etc/dockroll/apps/<appId>` 本身），规范化，不含 `..`/`//` 与尾斜杠；数组内不可重复，且任意两个 `targetPath` 不能互为目录/文件前缀 |
| `mode` | 是 | 四位八进制字符串，匹配 `^0[0-7][0-7][0145]$`（首位固定 `0` 禁止 SUID/SGID/Sticky 位，拒绝 world-writable） |
| `uid` / `gid` | 是 | 整数，`0～4294967295`（Linux `uint32` 范围）；落盘时按此设置属主 |

`config.files[]` 只列出本次要写入的文件；省略的文件保持原状。配置源文件按哈希和大小校验通过后，设备必须先持久保存每个目标文件的原内容、权限、UID/GID，或记录该文件原本不存在，再开始写入。设备须在任何写入前拒绝无法安全备份或恢复的目标，并保留这些记录直到发布提交或回滚完成。当 `single_container` 或 `compose` 携带 `config` 时，设备先完成配置文件备份与落盘，再创建/重建容器使其通过 bind mount 读取新配置；纯 `config` 发布的生效范围见[配置生效策略](#配置生效策略)。

## 应用凭据

数据库密码、API Token 等真实凭据不得写入 manifest、`compose.yaml`、环境文件、其他发布制品、日志或诊断输出。`container.env[].value` 只放非敏感值，例如应用读取凭据的容器内文件路径。

运维按 `appId` 在设备上预置凭据文件并设置访问权限。单容器通过 `container.mounts[]`，Compose 通过 `compose.yaml` 声明只读挂载；设备在预检查中确认真实路径仍位于该应用的凭据目录、文件已存在且为普通文件。不符合条件时拒绝发布，不自动创建文件或读取其他应用的凭据。应用直接读取挂载后的文件；OTA 不写入或回滚设备上的凭据。

## 镜像、制品与哈希规则

### 镜像引用

`container.image` 和 `compose.services[].image` 都使用同一种字符串：`<registry>/<repository>[:<tag>]@sha256:<64 位小写十六进制>`。仓库名必须包含 registry（可含端口）和完整仓库路径；不接受 URL scheme 或隐式 registry。

tag 可省略，只用于标识人类可读的版本。发布流水线应使用同一次发布的 tag 和 registry 返回的 digest。镜像拉取、版本比较和回滚按解析后的 `repository@digest` 执行；只改 tag 不算镜像变更，也不能在 digest 缺失或拉取失败时回退到 tag。保留清单中的原始 `image` 字符串用于展示和验签。

旧的 `{repository, digest}` 对象不是 v1.0 候选格式；平台仍可在 `image` 的同级 `platform` 字段指定。只有 tag 没有 digest 的拒绝示例：`$.container.image: image reference must include a sha256 digest [image.digest-required]`。

清单校验器不拉取镜像，也不验证 registry 端 digest 是否存在；M1 下载时按 digest 拉取并校验。

### 文件制品

- `path`：相对路径，匹配 `^[A-Za-z0-9._/-]+$`；禁止以 `/` 或 `./` 开头、禁止 `//`、禁止尾斜杠 `/`、禁止任何路径段为 `.` 或 `..`、禁止指向 `manifest.json` 自身；解析后的真实路径必须仍在发布目录内（防路径穿越）。
- `sha256`：必填字符串，统一带前缀，匹配 `^sha256:[0-9a-f]{64}$`。
- `sizeBytes`：整数，`0～10485760`（允许 0 字节文件，单制品上限 10 MiB）。
- 制品必须存在，且实际 `sizeBytes` 和 `sha256` 与声明一致。失败示例：`$.config.files[0].source.sha256: sha256 mismatch: declared sha256:0000…, actual sha256:… [artifact.hash-mismatch]`；`$.config.files[0].source.sizeBytes: size mismatch: declared 97, actual 96 [artifact.size-mismatch]`。
- 配置/Compose 文件的实际内容哈希在发布构建时生成并签名；设备端在激活前重新计算，任何不一致都拒绝。

## 健康检查与观察窗口

`health.check` 通过必填字段 `type` 三选一：

| `type` | 必填字段 | 说明 |
| --- | --- | --- |
| `http` | `path`、`port`、`scheme`、`expectedStatus` | `path` 须以 `/` 开头；`port` 为 `1～65535`；`scheme` 为 `http`/`https`；`expectedStatus` 为 `100～599` 整数或 `2xx`/`3xx`/`4xx`/`5xx`。固定发送 `GET` 请求、不跟随重定向；`https` 使用系统可信 CA 校验 |
| `tcp` | `port` | 探测容器内端口，`port` 为 `1～65535` |
| `command` | `command`（非空数组） | 在受检容器内执行，上限 32 项且首元素非空，退出码 `0` 为通过 |

公共字段：`intervalSeconds`（1～300）、`timeoutSeconds`（1～120，必须小于 `intervalSeconds` 且 `<= observationWindowSeconds`）、`retries`（1～10）、`startupGraceSeconds`（0～600）、`observationWindowSeconds`（10～3600）、`successCriteria`（`all_services_healthy` / `primary_service_healthy`）。三种探针都支持同样的 `check.service` 规则（匹配 `^[a-zA-Z0-9._-]{1,64}$`）：`single_container` 禁止携带；`compose` 必须引用启用的服务；`config` 发布在离线校验器中允许可选携带（仅校验格式），由设备端预检查按绑定的现有部署及其服务集合校验。

| 部署与判定条件 | 每轮通过条件 |
| --- | --- |
| 单容器，两种 `successCriteria` | 容器在运行且显式探针通过；禁止 `check.service` |
| Compose，`primary_service_healthy` | 只检查主服务；多服务必须显式指定 `check.service`，只有一个启用服务时可省略。受检服务的每个预期实例都必须在运行且探针通过 |
| Compose，`all_services_healthy`，未指定 `check.service` | 必须至少有一个 `required: true` 的启用服务；对每个必需服务的全部预期实例运行同一个探针，每个实例都必须在运行且通过探针 |
| Compose，`all_services_healthy`，指定了 `check.service` | 指定服务必须是启用的必需服务，其全部预期实例都必须在运行且通过显式探针；其他必需服务的全部预期实例都必须在运行且 Docker 容器健康状态为 `healthy`。其他必需服务缺少有效 `HEALTHCHECK` 时在预检查拒绝，不能用「正在运行」代替健康 |

`all_services_healthy` 下，`required: false` 不参与成功判定，但仍属于部署集合。预期实例缺失、退出或探针超时均视为该轮未通过。例如，`api` 的 HTTP 探针通过但必需的 `worker` 已退出，整轮仍然失败。

`health.check.port` 表示**容器内端口**，从应用网络内部探测，不要求该端口已发布到宿主机；`expectedStatus` 支持 `204` 这类精确值和 `2xx` 类匹配。`rollback.deadlineSeconds` 从发布判定失败时开始计时（包括配置生效失败或观察判定失败），不包含观察时间；总恢复时间应按设备 RTO 单独核算。

启动宽限期结束后立即开始观察并执行首轮检查，首轮超时不得超过剩余观察时间。之后按固定周期（fixed-rate）每隔 `intervalSeconds` 发起一轮，剩余观察时间不足 `timeoutSeconds` 时不再发起新一轮。观察期间允许非连续的偶发失败：任意一轮通过即重置连续失败计数；若连续 `retries` 轮失败，则立即提前判定失败并触发回滚。观察窗口结束时，若至少完成一轮、期间未达到连续 `retries` 轮失败且最后一轮检查通过，则提交发布；否则回滚。清单校验器仅检查结构和清单内可判定的引用，已部署对象及其健康检查能力由设备预检查确认。

## 时间戳语义

`createdAt` 表示清单创建时间，受签名保护。校验器须按 UTC 日历解析，拒绝 `2026-13-45T99:99:99Z` 这类格式匹配但日期非法的值。设备端另行执行重放防护，不能只凭 `createdAt` 判定发布是否可接受。

## 配置生效策略

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `applyPolicy.strategy` | 是 | `reload` / `restart_container` / `recreate_container` |
| `applyPolicy.reload.command` | 条件必填 | `strategy=reload` 时必填，非空字符串数组 |
| `applyPolicy.reload.timeoutSeconds` | 条件必填 | `strategy=reload` 时必填，1～300 秒 |
| `applyPolicy.reload.service` | 条件必填 | 匹配 `^[a-zA-Z0-9._-]{1,64}$`；仅绑定 Compose 部署时允许使用（多启用服务必填且须引用启用服务，单启用服务可省略）；离线校验器对 `releaseType=config` 允许可选携带并仅校验格式，设备端按绑定部署校验 |

`strategy` 非 `reload` 时禁止出现 `reload` 块。对于 `single_container` 和 `compose` 发布，`applyPolicy.strategy` **必须为 `recreate_container`**（禁止 `reload` 和 `restart_container`）。`reload` 和 `restart_container` 仅允许用于 `releaseType=config` 的纯配置发布。

对于纯 `config` 发布，预检查必须按 `appId` 找到并绑定设备当前已提交的单容器或 Compose 部署，记录部署版本、启用服务及其配置挂载。找不到唯一目标、配置未挂载到该部署、策略不能使修改生效，或目标在应用前已变化时，拒绝写入。设备按 `appId` 串行执行配置更新和部署切换。

- `reload`：在绑定的单容器或指定 Compose 服务的全部运行实例中执行命令。若受影响的配置还被其他服务使用，不得用一次定向 reload 作为整个发布的生效策略。只有选定实例能读到新配置且命令全部成功时才进入观察。
- `restart_container`：重启绑定部署中挂载受影响配置的容器，不改镜像或服务集合。
- `recreate_container`：按绑定的部署版本重建挂载受影响配置的容器，不改镜像或服务集合。若原单文件 bind mount 不能看到替换后的文件，必须使用此策略。

## 回滚策略

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `strategy` | 是 | `previous_release`：回到本次发布开始前的快照状态（含首次部署时的“未部署”空快照）；`explicit_release`：恢复指定发布的状态 |
| `targetReleaseId` | 条件必填 | `explicit_release` 时必填；必须属于同一 `appId`，且不能等于当前 `releaseId`。`previous_release` 时禁止 |
| `maxAttempts` | 是 | 1～3 |
| `preserveVolumes` | 是 | 固定 `true`；回滚不得删除卷，见[资源保留规则](#资源保留规则) |
| `restoreServiceSet` | 是 | 固定 `true`；回滚必须恢复目标的完整服务集合，纯配置回滚不得把配置清单当作部署清单 |
| `deadlineSeconds` | 是 | 30～1800；回滚执行与回滚后验证均须在此截止时间内完成，超时即判定回滚失败 |

每次发布开始变更前，设备须持久记录该 `appId` 的**发布前快照**（包含当前已提交的部署版本、完整服务集合及受管配置文件状态；首次部署时为空快照，即无受管容器、无受管配置文件）。部署回滚使用目标版本的完整部署状态，包括部署清单及与之对应的受管配置；配置清单本身不能重建服务。采用 `explicit_release` 时，设备须在变更前确认指定目标可完整恢复。若该目标是 `config`，必须能取得其提交时记录的部署版本和完整受管配置状态，仅有 `config.files[]` 不够。

若本次发布包含配置文件写入（无论是纯 `config` 发布，还是携带了 `config` 载荷的 `single_container` / `compose` 发布），回滚时设备须撤销本次写入：恢复原文件内容和权限、属主，删除本次创建且原先不存在的配置文件。此步骤只处理本次记录的受管配置文件，不触碰其他文件或挂载目录。然后按回滚策略恢复运行状态：

- `previous_release`：恢复至发布开始前的快照状态。
  - 若快照存在已提交部署：对 `single_container` / `compose` 发布，恢复快照的完整服务集合与受管配置，并清理本次新建但快照中不存在的受管容器；对纯 `config` 发布，保留快照绑定的部署，若已经开始使新配置生效，则用恢复后的配置和原部署定义重建受影响的容器（不重新执行本次失败的 reload 命令）。
  - 若为首次部署（空快照）：清理本次新建的受管容器并撤销本次写入/新建的受管配置文件，回到“未部署”状态（仍遵守[资源保留规则](#资源保留规则)，不自动删除卷、网络及宿主机持久数据目录）。
- `explicit_release`：在撤销本次写入后恢复已确认的目标部署和配置状态。按目标状态处理受管文件，不能只重放目标配置清单。

例如，Compose 版本 A 已在运行，纯配置版本 B 新建了 `extra.conf` 后探针失败；`previous_release` 必须撤销该文件。即使重新应用 A 的 Compose 文件，也不能代替这一步。

回滚执行与回滚后核对均须在 `rollback.deadlineSeconds` 内完成。当恢复目标包含已提交部署时，回滚后必须按恢复目标核对服务集合、端口占用、所需的卷和网络、bind mount，并在剩余 `deadlineSeconds` 预算内通过恢复目标的健康检查（按目标的 `health.check` 与 `successCriteria` 至少完成一轮通过，不再等待目标的完整 `observationWindowSeconds`）；当回滚至首次部署的空快照时，核对本次新建的受管容器与受管配置文件已清理干净即可。回滚达到 `maxAttempts` 上限或超过 `deadlineSeconds` 仍未成功时，停止自动切换（不自动切到其他版本），保留现场与诊断信息，并将回滚失败结果直接上报给 Server。

## 签名

`schemaVersion: "1.0"` 固定使用 [Ed25519](https://www.rfc-editor.org/rfc/rfc8032.html#section-5.1) 和 [RFC 8785 JCS](https://www.rfc-editor.org/rfc/rfc8785.html)。`signature` 是唯一的签名字段，必须是字符串；旧的签名对象或单列的算法、密钥标识、签名时间和规范化字段均不接受。

解析 manifest 时，在对象归并或数值转换前拒绝重复的对象键（按解码后的键名比较）、非法 UTF-8 和不成对的 UTF-16 代理项。所有数值字段只接受规范十进制整数记法（禁止小数点、指数和 `-0`），先精确检查范围 `-9007199254740991`～`9007199254740991`（含端点），再应用具体字段的更窄约束。

签名和验签时，仅移除顶层 `signature`，不补默认值或改写其他字段。将剩余对象按 JCS 规范化为 UTF-8 字节：对象键按 UTF-16 code unit 排序，保持数组顺序和 Unicode 字符串内容不变。直接对这些字节使用 Ed25519，不预哈希，也不使用 `Ed25519ph`。

签名的 64 字节按 [RFC 4648 §4](https://www.rfc-editor.org/rfc/rfc4648.html#section-4) 编码为标准 Base64，保留末尾 `==`，拒绝空白、Base64URL 和非规范编码。

校验器输出的 `manifest_digest=sha256:<…>` 是同一份 JCS 字节的 SHA-256，用于引用和比对；不签名这段文本。

设备只使用本地信任配置中已为该 `appId` 授权且仍有效的有限公钥集合验签；没有任何公钥通过时拒绝发布。轮换时先通过可信渠道给设备下发新公钥，再切换用于签名的私钥。撤销、过期、重放防护和旧版回滚的授权由设备端安全策略定义。`--trusted-key` 仅用于测试样例验证，生产发布不得使用测试密钥。

## 未知字段、默认值与版本演进

未知字段处理：所有已定义对象都设置 `additionalProperties: false`。错误示例：`$.unexpected_release_note: unknown field 'unexpected_release_note' is not allowed [schema.additionalProperties]`。

默认值汇总：

Schema **不再声明 `default` 注解**，避免 Agent 各自实现默认值导致分叉：

| 字段 | 缺省语义 |
| --- | --- |
| `compose.services[].required` | 必填布尔，不再有隐式 `true` |
| `container.ports[].hostIp` | 必填，必须是合法 IPv4/IPv6 |
| `container.stopGracePeriodSeconds` | 必填整数 |
| `compose.envFiles` / `compose.profiles` | 缺省按空数组处理；数组本身不写 `default` |
| `compatibility.minComposeVersion`（`single_container` / `config` 中） | 缺省不存在，不要求设备满足 Compose 最低版本 |
| `config`（`single_container` / `compose` 中） | 缺省不存在，本次发布不写入受管配置文件 |
| `metadata` | 缺省不存在 |

以下版本演进规则适用于首次冻结后：

1. **Major（如 1.0 → 2.0）**：新增必填字段、删除/重命名字段、收窄类型或枚举、改变字段语义、改变签名规范化方式。设备必须拒绝不支持的 major；不承诺向后兼容。
2. **Minor（如 1.0 → 1.1）**：只允许新增可选字段或新增不影响 v1.0 语义的校验规则；设备只接受显式支持列表中的版本。旧设备遇到更高 minor 必须在字段校验前拒绝，并报告“不支持的 schema 版本”，而不是把它当作未知字段误报。
3. 在受支持的 major.minor 内，未知字段始终拒绝；不能以“向前兼容”为由放宽字段白名单。
4. 文档澄清、拼写修正且不改变 schema 字节和校验结果时不升级版本；一旦改变可观察校验行为，至少升 minor。
5. 已发布 manifest 不可原地修改；任何目标变化都必须生成新 `releaseId` 并重新签名。回滚引用旧 manifest，不修改旧文件。
