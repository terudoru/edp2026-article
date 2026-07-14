# ProxmoxMCP-Plus 完全設定メモ

年刊EDP2026「Proxmoxやりましょう」の補足資料です。本文では省略した、常用tokenと管理用tokenの分離、Proxmox側の権限、Codex側の完全な設定例をまとめます。

token secretを含む `config.json` と `config.admin.json` はGitへ追加しないでください。以下の値はすべて例です。

## 認証情報を分ける

```text
常用ユーザー: codex@pve
常用token: codex@pve!codex
常用config: proxmox-config/config.json

作業用ユーザー: codex-admin@pve
作業用token: codex-admin@pve!admin
作業用config: proxmox-config/config.admin.json
```

常用tokenは読み取りと低リスク操作に絞ります。作業用のAdministrator tokenは別に作り、必要なときだけ使います。

## Proxmox側の権限

`Privilege Separation`を有効にしたtokenでは、元ユーザーとtokenの両方に権限が必要です。DatacenterのPermissionsで、常用ユーザー本体とtokenの両方へ`PVEAuditor`を付けます。

```text
/  codex@pve        PVEAuditor  true
/  codex@pve!codex  PVEAuditor  true
```

管理用も、ユーザー本体とtokenの両方へ`Administrator`を付けます。

```text
/  codex-admin@pve        Administrator  true
/  codex-admin@pve!admin  Administrator  true
```

## ProxmoxMCP-Plusの設定ファイル

同梱のexampleをコピーし、`host`、`user`、`token_name`、`token_value`を環境に合わせて編集します。

```bash
# 同梱のexampleをコピーして、secret入りの設定ファイルを作る。
cp proxmox-config/config.example.json proxmox-config/config.json

# host、user、token_name、token_valueを環境に合わせて編集する。
nano proxmox-config/config.json

# secret入り設定ファイルを自分だけ読める権限にする。
chmod 600 proxmox-config/config.json
```

自己署名証明書のまま使う場合は、`config.json`に`verify_ssl=false`と`dev_mode=true`を設定します。管理用は同じ要領で`config.admin.json`を作ります。

## Codexの常用MCP設定

`~/.codex/config.toml`へ追記します。すでに`[mcp_servers.proxmox]`がある場合は、見出しを重複させず中身を置き換えます。

```toml
# uvxでProxmoxMCP-Plusを起動する。
[mcp_servers.proxmox]
command = "uvx"
args = ["proxmox-mcp-plus"]

# secret入りconfig.jsonの絶対パスを指定する。
env = { PROXMOX_MCP_CONFIG = "/path/to/proxmox-config/config.json" }

# 状態変更操作は毎回確認する。
default_tools_approval_mode = "prompt"
enabled_tools = [
  "get_nodes",
  "get_cluster_status",
  "get_storage",
  "get_vms",
  "get_containers",
  "list_snapshots",
  "start_vm",
  "start_container",
  "stop_container",
  "create_snapshot",
  "list_jobs",
  "get_job",
  "poll_job"
]

# 読み取り系だけ自動承認する。
[mcp_servers.proxmox.tools.get_nodes]
approval_mode = "approve"

[mcp_servers.proxmox.tools.get_cluster_status]
approval_mode = "approve"

[mcp_servers.proxmox.tools.get_storage]
approval_mode = "approve"

[mcp_servers.proxmox.tools.get_vms]
approval_mode = "approve"

[mcp_servers.proxmox.tools.get_containers]
approval_mode = "approve"

[mcp_servers.proxmox.tools.list_snapshots]
approval_mode = "approve"
```

`stop_vm`はforce stopなので常用側の`enabled_tools`から外します。停止するときはgraceful shutdownを使います。

Codexを再起動したら、最初は読み取りだけを確認します。

```text
Proxmoxのnode、storage、VM、LXC一覧を取得して。
変更操作はしないで。
```

## Codexの管理用MCP設定

管理用MCPは`proxmox_admin`という別名で登録します。削除、rollback、force stopまでできるため自動承認は付けません。

```toml
# 常用のproxmoxとは別名で登録する。
[mcp_servers.proxmox_admin]
command = "uvx"
args = ["proxmox-mcp-plus"]

# admin tokenのsecretが入ったconfig.admin.jsonを指す。
env = { PROXMOX_MCP_CONFIG = "/path/to/proxmox-config/config.admin.json" }

# 読み取りを含め、すべて実行前に確認する。
default_tools_approval_mode = "prompt"
```

管理用MCPを使う前に、対象、目的、戻し方を確認します。普段は常用MCPだけを使い、管理用は必要な作業の間だけ有効にします。

## 参照先

- [ProxmoxMCP-Plus](https://github.com/RekklesNA/ProxmoxMCP-Plus)
