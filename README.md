# dev-env-playbook

Ubuntu 開発サーバの環境を `ansible-pull` で宣言的に構成する playbook。ホスト自身がこのリポジトリを取得し、localhost に適用する（制御ノードに ansible を導入しない pull 型）。

**EC2（AWS）と自宅サーバ（mini-PC）の両環境を 1 つの playbook で構成する。** 環境差は次の 2 軸で吸収する:

- `target_env`（`ec2` / `home`）… 環境固有値（AWS リージョン・鍵の取得元・ufw 等）。`vars/env-<target_env>.yml`。
- `ansible_architecture`（`aarch64` / `x86_64`）… arch 固有値（AWS CLI / mise / chezmoi の arch・sha256）。`vars/arch-<arch>.yml`。

`target_env` は **明示の `-e target_env=...` が最優先**、無ければ DMI vendor で自動判定（Amazon EC2 → `ec2`、それ以外 → `home`）。これにより **EC2 側の既存コマンドは無変更**で動く。

## 適用方法（対象ホスト上で実行）

```bash
# 初回のみ: ansible / git を導入
sudo apt-get update
sudo apt-get install -y ansible git

# EC2: 自動判定で target_env=ec2 になる（明示不要）
sudo ansible-pull \
  -U https://github.com/xande0812/dev-env-playbook.git \
  -i inventory/hosts \
  site.yml

# 自宅サーバ: 自動判定でも home になるが、明示すると確実
sudo ansible-pull \
  -U https://github.com/xande0812/dev-env-playbook.git \
  -i inventory/hosts \
  -e target_env=home \
  site.yml
```

- `ansible-pull` はリポを `~/.ansible/pull/<hostname>` に clone/更新してから `site.yml` を実行する（`sudo` 実行なので root の home 配下）。
- 別ブランチを試すときは `-C <branch>` を付ける。
- 自宅サーバの OS インストール〜初回 `ansible-pull` 起動は別リポ [dev-server-bootstrap](https://github.com/xande0812/dev-server-bootstrap)（Ubuntu autoinstall + cloud-init）が担う。EC2 側のブートストラップは `ai-agent-sandbox-cdk`。

## 構成

| パス | 役割 |
|---|---|
| [ansible.cfg](ansible.cfg) | inventory / 出力整形 |
| [inventory/hosts](inventory/hosts) | localhost のみ（connection=local） |
| [site.yml](site.yml) | playbook。`pre_tasks` で `target_env` 判定 + vars 読込 → 下記 role を system→user の順で適用 |
| [group_vars/all.yml](group_vars/all.yml) | 両環境共通の変数のみ |
| [vars/env-ec2.yml](vars/env-ec2.yml) / [vars/env-home.yml](vars/env-home.yml) | 環境固有の変数 |
| [vars/arch-aarch64.yml](vars/arch-aarch64.yml) / [vars/arch-x86_64.yml](vars/arch-x86_64.yml) | アーキテクチャ固有の変数 |

### roles（[site.yml](site.yml) の適用順）

| role | 役割 |
|---|---|
| [base](roles/base) | apt 基本パッケージ + AWS CLI v2（arch 依存） |
| [sshd_hardening](roles/sshd_hardening) | sshd の公開鍵認証強制 + AcceptEnv |
| [ufw](roles/ufw) | **home のみ。** インバウンド制御（旧 EC2 Security Group の代替） |
| [apparmor_bwrap](roles/apparmor_bwrap) | `/usr/bin/bwrap` に userns を許可する AppArmor profile |
| [squid](roles/squid) | egress proxy（SNI allowlist） |
| [dev_user](roles/dev_user) | 開発ユーザー作成 + authorized_keys（ec2=SSM / home=git の公開鍵） |
| [dev_tools](roles/dev_tools) | apt ユーティリティ + mise binary（arch 依存） + login shell=zsh |
| [obsidian](roles/obsidian) | **home のみ。** Obsidian Desktop `.deb` 導入（同梱 CLI の前提） |
| [chezmoi](roles/chezmoi) | chezmoi 導入（arch 依存） + dotfiles 適用（mise install / sheldon lock 込み） |
| [bwrap_wrappers](roles/bwrap_wrappers) | claude/codex/pnpm の bwrap sandbox wrapper + leak 自テスト |

dotfiles 本体（zsh / git / tmux / mise config / claude・codex の設定）は chezmoi（公開リポ `dev-env-dotfiles`）が管理する。この playbook は system 層・ツール binary 導入・chezmoi 起動を担う。

### Obsidian CLI

Obsidian CLI は standalone binary ではなく Obsidian Desktop に同梱される。playbook は home 環境で Desktop `.deb` を version/sha256 pin して導入する。インストール後に Obsidian を起動し、`Settings -> General -> Command line interface` を有効化すると、CLI binary が `~/.local/bin/obsidian` に登録される。`~/.local/bin` は PATH に入っている必要がある。

### 自宅サーバ（home）で要設定の値

[vars/env-home.yml](vars/env-home.yml) の以下は環境に合わせて置換する:

- `dev_user_pubkey` … 開発ユーザーの SSH 公開鍵（dev-server-bootstrap の autoinstall と同一鍵）。
- `lan_subnet` … ufw で SSH を許可する自宅 LAN サブネット。

### AI agent sandbox（bwrap_wrappers）

claude / codex / pnpm を bwrap で隔離して起動する wrapper を `/usr/local/` に配置する。CLI バイナリ自体は mise（dotfiles の `dot_config/mise/config.toml` で版 pin）が `$HOME` 配下に install し、wrapper が `mise which` で実体パスを解決して sandbox 内で起動する。`~/.aws` / `~/.ssh` 等が sandbox 内から不可視であることを [roles/bwrap_wrappers](roles/bwrap_wrappers) の leak 自テストが検証し、leak を検出したら provisioning を fail させる。IMDS / instance role には依存しないため EC2 / home 両方で同一に動く。

## 秘密情報

このリポジトリは公開。**秘密値（API キー・鍵・トークン）は一切置かない**。秘密が必要な処理は実行時にホスト側のセキュアな仕組み（SSM Parameter Store / `SendEnv`+`AcceptEnv` の session env 等）から取得する設計に寄せる。`dev_user_pubkey` は**公開鍵**なので git 管理してよい。
