# AgentTasks 0.1.0

面向 agent 的本地任务服务，运行于 macOS 菜单栏，没有主窗口。提供 CLI、MCP、session 心跳和飞书多维表格同步。

本仓库仅用于二进制分发和使用说明，不包含实现源码。

## 下载与安装

从 [Releases](https://github.com/stacezhou/agt-daemon/releases/latest) 下载 `AgentTasks-0.1.0-macos-universal.dmg`，打开后将 AgentTasks 拖入 Applications，再启动 App。

要求 macOS 13 或更新版本，支持 Apple Silicon 和 Intel。无需自行安装 Node、Python 或 Swift。主安装包内已包含飞书同步插件，独立插件包供需要单独安装或更新插件的用户使用。

发布的 DMG 使用 Developer ID 签名并经过 Apple 公证，可通过同一版本的 `SHA256SUMS.txt` 检查下载完整性。

## CLI 与 MCP

在终端安装命令行入口：

```sh
/Applications/AgentTasks.app/Contents/Resources/bin/agenttasks install-cli
```

入口位于 `~/.local/bin/agenttasks`。如该目录未在 PATH 中，请自行加入；也可以直接使用 App 内 CLI 的绝对路径。

App 运行时，下面的命令输出 MCP 客户端配置：

```sh
agenttasks mcp config
```

将输出加入 agent 的 MCP 配置。MCP 提供项目、任务和 session 工具，agent 可以查询、创建、认领和完成任务。

## 任务与心跳

```sh
agenttasks project create --name demo --json
agenttasks session start --agent my-agent --json
agenttasks task create --project PROJECT_ID --title '处理任务' --json
agenttasks task claim TASK_ID --session SESSION_ID
agenttasks hook activity SESSION_ID
agenttasks hook heartbeat SESSION_ID
agenttasks task list --attention --watch
agenttasks task complete TASK_ID --session SESSION_ID --summary '已验证'
agenttasks hook stop SESSION_ID --reason '工作结束'
```

用创建命令返回的实际 ID 替换占位符。同一次 agent 工作会话的 MCP 操作和 hook 应使用同一个 session ID。Hook 也可用 `--stdin` 接收 JSON 对象，默认 2 秒超时。

建议每 30 秒发送心跳，默认 120 秒没有信号显示失联；`activity` 同时记录最近活动。失联只说明近期未收到信号，不能证明进程已经退出。显式停止的 session 不会被迟到心跳复活。

任务状态与 session 状态分别记录。`--attention` 显示任务未完成、但认领 session 已停止或失联的情况；不会自动释放、转派或完成任务。

CLI 支持 `--json`、列表分页 `--limit` / `--offset` 和部分查询的 `--watch`。数据默认保存在 `~/Library/Application Support/AgentTasks`。不需要菜单栏时可使用 `agenttasks daemon run`；同一用户和数据目录只运行一个服务实例。

## 飞书多维表格

先在飞书准备企业自建应用及可读写的多维表格，取得 App ID、App Secret、app_token 和 table_id。按接口要求开通字段读取、记录查询/创建/更新权限，并授予应用相应表格访问权限；高级权限下需要确认应用能看到整张表。

默认字段：

| 列名 | 类型 |
|---|---|
| 标题 | 文本 |
| 描述 | 文本 |
| 状态 | 文本或单选：待办、进行中、已完成 |
| AgentTasks操作ID | 文本，供插件维护，请勿手动修改 |

注册主 App 内的插件：

```sh
agenttasks plugin register /Applications/AgentTasks.app/Contents/Resources/Plugins/FeishuBitable/manifest.json
```

若已注册旧路径，先停用使用该插件的同步绑定，再用 `plugin update` 指向新 manifest。独立插件包需要先将整个 `FeishuBitable` 文件夹复制到长期保留的位置，再注册其中的 manifest，不要注册临时挂载的 DMG 路径。

在 zsh 中隐藏输入 App Secret，并存入系统 Keychain：

```zsh
read -s 'FEISHU_APP_SECRET?Feishu App Secret: '
printf '\n'
printf '%s' "$FEISHU_APP_SECRET" | agenttasks credential set feishu-demo --stdin
unset FEISHU_APP_SECRET
```

配置项目绑定：

```sh
agenttasks sync configure --project PROJECT_ID \
  --plugin feishu-bitable --account cli_APP_ID \
  --container APP_TOKEN/TABLE_ID --credential-ref feishu-demo --interval 30
agenttasks sync enable PROJECT_ID
agenttasks sync now PROJECT_ID
agenttasks sync status PROJECT_ID
```

每个项目最多启用一个云端目标，不同项目可使用不同目标。可通过 `--config` 自定义字段名和状态值；配置结构如下：

```json
{
  "fields": {
    "title": "任务名称",
    "description": "说明",
    "status": "进度",
    "operation_id": "AgentTasks操作ID"
  },
  "statuses": {"todo": "待处理", "in_progress": "处理中", "done": "完成"},
  "max_records": 10000
}
```

首次同步导入云端记录，并为尚未关联的本地任务创建记录；同名任务不会自动合并。已知对应关系可先用 `sync link PROJECT_ID --task TASK_ID --remote-id RECORD_ID` 关联。

飞书插件双向同步标题、描述和状态，同字段双方修改时采用云端值；不上传 session、心跳或认领关系。它分页扫描表格并返回摘要差异；确认记录不存在后才标记删除，权限隐藏不会被当作删除。默认最多处理 10000 条记录。

## 当前边界

- 飞书适配器通过了离线 API 契约和进程测试，尚未在真实租户完成联调；请先在测试表中验证权限与字段。
- 飞书缺少已核实的原子条件更新，写前检查与写后校验仍有竞争窗口。结果不明的创建不会被盲目重试，可通过远端操作 ID 或显式关联恢复。
- 版本验证包括 40 个回归场景、MCP/CLI 端到端流程及 arm64 原生、x86_64/Rosetta 运行。未在独立 Intel 硬件或 macOS 13 实机验证。
- 没有登录自启、自动升级或任务自动接管功能。

第三方组件许可见安装包内及 Release 附件的 `ThirdPartyNotices.txt`。
