# 发布清单 Schema v1.0 与校验规则

状态：v1.0 候选规范，尚未冻结。

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
- Compose 文件的字段子集（禁止 `build`、远程 `include`、`profiles` 的默认激活等）由[三类参考发布包与 Compose 规则](../stories/Story-Release-Packages.md)定义；[Compose 载荷](#compose-载荷)规定设备如何锁定输入、服务集合和镜像 digest。
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
| `sharedResources` | 否 | array | 本次声明保护的共享卷、网络和宿主机路径；最多 32 项；缺省不撤销已有保护，见[共享资源声明](#共享资源声明) |
| `container` | 条件必填 | object | `releaseType=single_container` 时必填 |
| `compose` | 条件必填 | object | `releaseType=compose` 时必填 |
| `config` | 条件必填 | object | `releaseType=config` 时必填 |
| `secrets` | 否 | array | 受控密钥引用；默认空数组；禁止任何内联值 |
| `applyPolicy` | 是 | object | 配置生效策略；纯配置发布作用于同一 `appId` 的现有部署，见[配置生效策略](#配置生效策略) |
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
- 部署回滚按目标部署状态恢复**完整集合**：恢复目标有、当前没有的服务，并删除目标没有、但属于同一 `appId` 的受管服务；不得留下占用端口的残留服务。纯配置发布的文件恢复见[回滚策略](#回滚策略)。
- `takeover.preserveSharedResources` 只引用本清单的 `sharedResources[]`；未列入接管引用的资源仍按设备上已有和本次新增的保护记录处理。
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

设备在执行发布或接管前，必须按 `appId`、资源类型和实际资源身份持久记录保护。卷和网络在 Compose 部署中按有效模型的同类资源键解析为 Docker 实际名称；未引用模型资源时使用显式 Docker 名称。`bind_path` 使用宿主机上的规范化真实路径。若无法确定唯一的实际资源，则拒绝本次发布。执行升级、回滚或垃圾回收时，以设备上仍有效的保护记录为准；不得直接删除受保护资源，也不得通过递归删除上级目录间接删除受保护路径。

新版本省略 `sharedResources`、提交空数组、变更逻辑名，或回滚到不含该声明的旧版本，都不会撤销已有保护。只有单独授权的显式操作才可撤销某应用对该资源的保护；只要还有任一应用的有效记录，就不得删除。撤销保护本身不是删除授权，也不放宽 `rollback.preserveVolumes`。

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
| `envFiles` | 否 | 制品数组；默认空数组；按声明顺序提供插值变量，同名变量以后一个文件为准 |
| `services` | 是 | 启用的完整服务集合，非空且 `name` 不重复；每项包含 `name`（Compose service 名）、`image`（digest 固定）和 `required`（布尔，显式必填，参与[健康判定](#健康检查与观察窗口)） |
| `profiles` | 否 | 字符串数组；默认空数组；只启用显式声明的 profile |

本页的校验器只验证 manifest 侧和文件哈希；Compose 字段子集和有效模型由设备在 M1 预检查：

1. 使用 `compose.file`、已校验的 `compose.envFiles`、`compose.profiles` 和 `ownership.composeProjectName` 显式解析有效模型。禁止继承设备环境中的插值变量或 Compose 控制变量，禁止自动读取 `.env` 或发现追加的 Compose 文件；发布的环境文件不得覆盖 Compose 控制参数。未解析且无显式默认值的变量导致预检查失败。除按策略在设备上解析的宿主机 bind 路径和受控密钥引用外，模型引用的发布包文件必须由 manifest 声明并校验；服务的 `env_file` 也只能引用 `compose.envFiles` 中的制品。
2. 按有效模型比对服务集合和每个服务的镜像仓库及 digest，不能用 tag。未绑定 profile 的服务及命中显式 profile 的服务纳入比对；未启用的服务不纳入。启用的依赖不完整则拒绝；不得靠指定单个服务隐式启用额外 profile。若声明 `image.platform`，其架构必须属于 `compatibility.arch`。
3. 部署和回滚必须使用对应版本预检查通过的有效模型，或用同一份已校验的输入和同样的解析参数得到相同模型后再执行。不得在执行时重新读取设备环境、隐式文件或未经校验的内容；[bind mount 安全边界](#bind-mount-安全边界)也以该模型为准。

Compose 的默认行为见 Docker 的[变量插值](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/)和 [profile](https://docs.docker.com/compose/how-tos/profiles/) 文档；DockRoll 用上述显式输入替代默认发现行为。

## 配置载荷

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `files` | 是 | 非空数组；每项包含 `source` 制品、`targetPath`、`mode`、`uid`、`gid` |
| `source` | 是 | 相对路径 + `sha256` + `sizeBytes`；文件内容不内联 |
| `targetPath` | 是 | 绝对路径，必须位于 `/etc/dockroll/apps/<appId>/` 下，规范化且不含 `..`；任意两个 `targetPath` 不能互为文件前缀 |
| `mode` | 是 | 四位八进制字符串（如 `0640`）；拒绝 world-writable（其他用户可写） |
| `uid` / `gid` | 是 | 非负整数；落盘时按此设置属主 |

`config.files[]` 只列出本次要写入的文件；省略的文件保持原状。配置源文件按哈希和大小校验通过后，设备必须先持久保存每个目标文件的原内容、权限、UID/GID，或记录该文件原本不存在，再开始写入。设备须在任何写入前拒绝无法安全备份或恢复的目标，并保留这些记录直到发布提交或回滚完成。配置通过 bind mount 暴露给现有部署；具体生效范围见[配置生效策略](#配置生效策略)。

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
| `http` | `path`、`port`、`scheme`、`expectedStatus` | `expectedStatus` 可填 100～599 的整数或 `2xx`/`3xx`/`4xx`/`5xx` 类 |
| `tcp` | `port` | 探测容器内端口 |
| `command` | `command`（非空数组） | 在受检容器内执行，退出码 `0` 为通过 |

公共字段：`intervalSeconds`（1～300）、`timeoutSeconds`（1～120，且必须小于 `intervalSeconds`）、`retries`（1～10）、`startupGraceSeconds`（0～600）、`observationWindowSeconds`（10～3600）、`successCriteria`（`all_services_healthy` / `primary_service_healthy`）。三种探针都支持同样的 `check.service` 规则：只可用于 Compose 部署，且必须引用启用的服务。`config` 发布以预检查绑定的现有部署及其服务集合为准，由设备校验引用。

| 部署与判定条件 | 每轮通过条件 |
| --- | --- |
| 单容器，两种 `successCriteria` | 容器在运行且显式探针通过；禁止 `check.service` |
| Compose，`primary_service_healthy` | 只检查主服务；多服务必须显式指定 `check.service`，只有一个启用服务时可省略。受检服务的每个预期实例都必须在运行且探针通过 |
| Compose，`all_services_healthy`，未指定 `check.service` | 必须至少有一个 `required: true` 的启用服务；对每个必需服务的全部预期实例运行同一个探针，每个实例都必须在运行且通过探针 |
| Compose，`all_services_healthy`，指定了 `check.service` | 指定服务必须是启用的必需服务，其全部预期实例都必须在运行且通过显式探针；其他必需服务的全部预期实例都必须在运行且 Docker 容器健康状态为 `healthy`。其他必需服务缺少有效 `HEALTHCHECK` 时在预检查拒绝，不能用「正在运行」代替健康 |

`all_services_healthy` 下，`required: false` 不参与成功判定，但仍属于部署集合。预期实例缺失、退出或探针超时均视为该轮未通过。例如，`api` 的 HTTP 探针通过但必需的 `worker` 已退出，整轮仍然失败。

`health.check.port` 表示**容器内端口**，从应用网络内部探测，不要求该端口已发布到宿主机；`expectedStatus` 支持 `204` 这类精确值和 `2xx` 类匹配。`rollback.deadlineSeconds` 从发布判定失败时开始计时（包括配置生效失败或观察判定失败），不包含观察时间；总恢复时间应按设备 RTO 单独核算。

启动宽限期结束后立即开始观察并执行首轮检查，首轮超时不得超过剩余观察时间。之后每隔 `intervalSeconds` 发起一轮，但剩余观察时间不足 `timeoutSeconds` 时不再发起新一轮。若连续 `retries` 轮失败，则提前判定失败；观察窗口结束时，只有至少完成一轮且发起的每一轮均已通过，才可提交，否则回滚。清单校验器仅检查结构和清单内可判定的引用，已部署对象及其健康检查能力由设备预检查确认。

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
| `applyPolicy.reload.timeoutSeconds` | 条件必填 | `strategy=reload` 时必填，1～300 秒 |
| `applyPolicy.reload.service` | 条件必填 | 仅 Compose 部署；有多个启用服务时必填，且必须引用启用的服务。只有一个启用服务时可省略 |

`strategy` 非 `reload` 时禁止出现 `reload` 块。对于 `config`，预检查必须按 `appId` 找到并绑定设备当前已提交的单容器或 Compose 部署，记录部署版本、启用服务及其配置挂载。找不到唯一目标、配置未挂载到该部署、策略不能使修改生效，或目标在应用前已变化时，拒绝写入。设备按 `appId` 串行执行配置更新和部署切换。

- `reload`：在绑定的单容器或指定 Compose 服务的全部运行实例中执行命令。若受影响的配置还被其他服务使用，不得用一次定向 reload 作为整个发布的生效策略。只有选定实例能读到新配置且命令全部成功时才进入观察。
- `restart_container`：重启绑定部署中挂载受影响配置的容器，不改镜像或服务集合。
- `recreate_container`：按绑定的部署版本重建挂载受影响配置的容器，不改镜像或服务集合。若原单文件 bind mount 不能看到替换后的文件，必须使用此策略。

## 回滚策略

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `strategy` | 是 | `previous_release`：回到本次发布开始前的已提交状态；`explicit_release`：恢复指定发布的状态 |
| `targetReleaseId` | 条件必填 | `explicit_release` 时必填；必须属于同一 `appId`，且不能等于当前 `releaseId`。`previous_release` 时禁止 |
| `maxAttempts` | 是 | 1～3 |
| `failureAction` | 是 | `escalate`（停止自动切换并转人工）/ `keep_failed` |
| `preserveVolumes` | 是 | 固定 `true`；禁止删除业务数据卷 |
| `restoreServiceSet` | 是 | 固定 `true`；回滚必须恢复目标的完整服务集合，纯配置回滚不得把配置清单当作部署清单 |
| `deadlineSeconds` | 是 | 30～1800；超时进入人工处置 |

部署回滚使用目标版本的完整部署状态，包括部署清单及与之对应的受管配置；配置清单本身不能重建服务。采用 `explicit_release` 时，设备须在变更前确认指定目标可完整恢复。若该目标是 `config`，必须能取得其提交时记录的部署版本和完整受管配置状态，仅有 `config.files[]` 不够。

纯配置发布失败时，设备先撤销本次写入：恢复原文件内容和权限、属主，删除本次创建且原先不存在的配置文件。不触碰其他文件或受保护资源；若预检查发现撤销写入与共享资源保护冲突，必须拒绝该配置发布。然后根据策略恢复运行状态：

- `previous_release`：保留变更前绑定的部署。若已经开始使新配置生效，用恢复后的配置和原部署定义重建受影响的容器，不重新执行本次失败的 reload 命令。
- `explicit_release`：在撤销本次写入后恢复已确认的目标部署和配置状态。按目标状态处理受管文件，不能只重放目标配置清单。

例如，Compose 版本 A 已在运行，纯配置版本 B 新建了 `extra.conf` 后探针失败；`previous_release` 必须撤销该文件。即使重新应用 A 的 Compose 文件，也不能代替这一步。

回滚后必须按恢复目标核对服务集合、端口占用、共享资源、数据卷和健康状态。回滚达到尝试上限或超时仍未成功时，保留诊断并按 `failureAction` 处理，不自动切到其他版本。

## 签名

| 字段 | 必填 | 规则 |
| --- | --- | --- |
| `algorithm` | 是 | `ed25519` / `ecdsa-p256-sha256` |
| `keyId` | 是 | 稳定密钥标识 |
| `signedAt` | 是 | UTC 时间 |
| `canonicalization` | 是 | 固定 `RFC8785` |
| `signature` | 是 | 使用 [RFC 4648 §4](https://www.rfc-editor.org/rfc/rfc4648.html#section-4) 标准 Base64，按需保留 `=` 填充；拒绝空白、非规范填充和 Base64URL |

规范化采用 [RFC 8785 JCS](https://www.rfc-editor.org/rfc/rfc8785.html) 的受限子集。解析 manifest 时，在对象归并或数值转换前拒绝重复的对象键（按解码后的键名比较）、非法 UTF-8 和不成对的 UTF-16 代理项。所有数值字段只接受十进制整数记法（禁止小数点和指数），先精确检查范围 `-9007199254740991`～`9007199254740991`（含端点），再应用具体字段的更窄约束。生成 JCS 字节时按 UTF-16 code unit 排序对象键，保持数组顺序和 Unicode 字符串内容不变，并输出 UTF-8。

签名输入由删除 `signature.signature` 后的 manifest 生成；`algorithm`、`keyId`、`signedAt`、`canonicalization` 仍受签名保护。签名和验签使用以下规则：

| 算法 | 对什么签名 | Base64 解码后的签名 |
| --- | --- | --- |
| `ed25519` | 直接签 JCS 字节，不预哈希，也不使用 `Ed25519ph` | 固定 64 字节 |
| `ecdsa-p256-sha256` | 用 P-256 签 JCS 字节的单次 SHA-256 摘要 | ASN.1 DER 编码的 `SEQUENCE { r INTEGER, s INTEGER }`；拒绝裸 `R‖S`、非 DER 编码和尾随数据 |

校验器输出的 `manifest_digest=sha256:<…>` 是同一份 JCS 字节的 SHA-256，用于引用和比对；不签名这段文本。

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
| `sharedResources` | 缺省不新增保护；已有保护不受影响 |
| `metadata` | 缺省不存在 |

以下版本演进规则适用于首次冻结后：

1. **Major（如 1.0 → 2.0）**：新增必填字段、删除/重命名字段、收窄类型或枚举、改变字段语义、改变签名规范化方式。设备必须拒绝不支持的 major；不承诺向后兼容。
2. **Minor（如 1.0 → 1.1）**：只允许新增可选字段或新增不影响 v1.0 语义的校验规则；设备只接受显式支持列表中的版本。旧设备遇到更高 minor 必须在字段校验前拒绝，并报告“不支持的 schema 版本”，而不是把它当作未知字段误报。
3. 在受支持的 major.minor 内，未知字段始终拒绝；不能以“向前兼容”为由放宽字段白名单。
4. 文档澄清、拼写修正且不改变 schema 字节和校验结果时不升级版本；一旦改变可观察校验行为，至少升 minor。
5. 已发布 manifest 不可原地修改；任何目标变化都必须生成新 `releaseId` 并重新签名。回滚引用旧 manifest，不修改旧文件。
