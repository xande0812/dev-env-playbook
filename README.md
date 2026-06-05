# dev-env-playbook

Ubuntu 開発サーバの環境を `ansible-pull` で宣言的に構成する playbook。ホスト自身がこのリポジトリを取得し、localhost に適用する（制御ノードに ansible を導入しない pull 型）。

## 適用方法（対象ホスト上で実行）

```bash
# 初回のみ: ansible / git を導入
sudo apt-get update
sudo apt-get install -y ansible git

# このリポから pull して localhost に適用
sudo ansible-pull \
  -U https://github.com/xande0812/dev-env-playbook.git \
  -i inventory/hosts \
  site.yml
```

- `ansible-pull` はリポを `~/.ansible/pull/<hostname>` に clone/更新してから `site.yml` を実行する（`sudo` 実行なので root の home 配下）。
- 別ブランチを試すときは `-C <branch>` を付ける。

## 構成

| パス | 役割 |
|---|---|
| [ansible.cfg](ansible.cfg) | inventory / 出力整形 |
| [inventory/hosts](inventory/hosts) | localhost のみ（connection=local） |
| [site.yml](site.yml) | playbook。下記 role を system→user の順で適用 |

### roles（[site.yml](site.yml) の適用順）

| role | 役割 |
|---|---|
| [base](roles/base) | apt 基本パッケージ + AWS CLI v2 |
| [sshd_hardening](roles/sshd_hardening) | sshd の公開鍵認証強制 + AcceptEnv |
| [apparmor_bwrap](roles/apparmor_bwrap) | `/usr/bin/bwrap` に userns を許可する AppArmor profile |
| [squid](roles/squid) | egress proxy（SNI allowlist） |
| [dev_user](roles/dev_user) | 開発ユーザー作成 + authorized_keys（SSM 経由） |
| [dev_tools](roles/dev_tools) | apt ユーティリティ + mise binary + login shell=zsh |
| [chezmoi](roles/chezmoi) | chezmoi 導入 + dotfiles 適用（mise install / sheldon lock 込み） |
| [bwrap_wrappers](roles/bwrap_wrappers) | claude/codex/pnpm の bwrap sandbox wrapper + leak 自テスト |

dotfiles 本体（zsh / git / tmux / mise config / claude・codex の設定）は chezmoi（公開リポ `dev-env-dotfiles`）が管理する。この playbook は system 層・ツール binary 導入・chezmoi 起動を担う。

### AI agent sandbox（bwrap_wrappers）

claude / codex / pnpm を bwrap で隔離して起動する wrapper を `/usr/local/` に配置する。CLI バイナリ自体は mise（dotfiles の `dot_config/mise/config.toml` で版 pin）が `$HOME` 配下に install し、wrapper が `mise which` で実体パスを解決して sandbox 内で起動する。`~/.aws` / `~/.ssh` 等が sandbox 内から不可視であることを [roles/bwrap_wrappers](roles/bwrap_wrappers) の leak 自テストが検証し、leak を検出したら provisioning を fail させる。

## 秘密情報

このリポジトリは公開。**秘密値（API キー・鍵・トークン）は一切置かない**。秘密が必要な処理は実行時にホスト側のセキュアな仕組み（パラメータストア等）から取得する設計に寄せる。
