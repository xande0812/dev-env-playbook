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
| [dev_user](roles/dev_user) | 開発ユーザー作成 + authorized_keys（ec2=SSM / home=git の公開鍵） |
| [dev_tools](roles/dev_tools) | apt ユーティリティ + mise binary（arch 依存） + login shell=zsh |
| [chezmoi](roles/chezmoi) | chezmoi 導入（arch 依存） + dotfiles 適用（mise install / sheldon lock 込み） |
| [squid](roles/squid) | egress proxy（SNI allowlist + TCP/80,443 透過 redirect） |
| [tailscale](roles/tailscale) | **home のみ。** WireGuard mesh VPN（tailscaled 導入 + ufw 受信許可）。認証は手動 |
| [bwrap_wrappers](roles/bwrap_wrappers) | claude/codex/pnpm の bwrap sandbox wrapper + leak 自テスト |

dotfiles 本体（zsh / git / tmux / mise config / claude・codex の設定）は chezmoi（公開リポ `dev-env-dotfiles`）が管理する。この playbook は system 層・ツール binary 導入・chezmoi 起動を担う。

### 自宅サーバ（home）で要設定の値

[vars/env-home.yml](vars/env-home.yml) の以下は環境に合わせて置換する:

- `dev_user_pubkey` … 開発ユーザーの SSH 公開鍵（dev-server-bootstrap の autoinstall と同一鍵）。
- `lan_subnet` … ufw で SSH を許可する自宅 LAN サブネット。

### AI agent sandbox（bwrap_wrappers）

claude / codex / pnpm を bwrap で隔離して起動する wrapper を `/usr/local/` に配置する。CLI バイナリ自体は mise（dotfiles の `dot_config/mise/config.toml` で版 pin）が `$HOME` 配下に install し、wrapper が `mise which` で実体パスを解決して sandbox 内で起動する。`~/.aws` / `~/.ssh` 等が sandbox 内から不可視であることを [roles/bwrap_wrappers](roles/bwrap_wrappers) の leak 自テストが検証し、leak を検出したら provisioning を fail させる。IMDS / instance role には依存しないため EC2 / home 両方で同一に動く。

agent が作業メモを書き残せるよう、Obsidian vault の **AI 専用サブフォルダ `~/obvault/AI` のみ** を rw bind する（claude / codex 両 base）。個人ノート本体（`~/obvault/` 直下）は bind せず sandbox 内から不可視のままにし、prompt injection 経由の個人ノート読取・改変を防ぐ。bind source は `--tmpfs /home` マスク下で実在しないと `--bind-try` が skip するため、各 base の `AGENT_MKDIR_DIRS` で起動前に `mkdir -p` して保証する。実際に何を記録させるか（slash command / frontmatter 規約）は dotfiles 側の `CLAUDE.md` / `AGENTS.md` / `~/.claude/commands` が担う。

Rust（cargo）を使う project では、host で mise（`core:rust` = rustup backend）が `~/.rustup` / `~/.cargo` に install した toolchain を sandbox 内で消費する。toolchain 本体（`~/.rustup`）は ro bind、cargo の registry/git cache は `~/.cache/cargo` に分離して rw bind で永続化する（pnpm の `~/.cache/pnpm` と同じ思想）。`~/.cargo` 自体は `credentials.toml`（crates.io publish token）を含みうるため bind せず不可視のままにし、leak 自テストが `~/.cargo/credentials.toml` の不可視を検証する。sandbox 内の mise auto-install は OFF（`MISE_NOT_FOUND_AUTO_INSTALL=false`）のため、新しい rust 版を使う project は **host 側（sandbox 外シェル）で先に `mise install`** しておく（ro bind 先への install は "Read-only file system" で失敗する）。

direct な TCP/80,443 は host 側 iptables で Squid の intercept port に redirect される。Squid 自身の outbound と localhost / private address / link-local 宛ては local control plane や LAN 内通信を壊さないため redirect 対象外。bwrap は network namespace を分けないため、sandbox 内の agent / pnpm 通信も同じ Squid allowlist を通る。

### Tailscale（home のみ）

自宅 dev-server をどこからでも安全に SSH/サービス接続できるよう、WireGuard ベースの mesh VPN [Tailscale](https://tailscale.com/) を [roles/tailscale](roles/tailscale) で導入する。

- **役割分担**: `tailscaled`（root 常駐 daemon）+ apt repo + systemd は system 層なので playbook が担う。
- **認証は手動**: このリポは公開のため auth key（秘密値）を置かない。`tailscale up` は **sandbox 外のシェル**で利用者が 1 度だけ実行する（network operation 手動方針とも整合）:

  ```bash
  sudo tailscale up      # 表示 URL をノート PC のブラウザで開き SSO 認証
  tailscale ip -4        # 付与された 100.x.y.z を確認
  ```

- **egress proxy 連携**: control plane / DERP relay / apt repo は TCP/443 で公開ホストに出るため squid の透過 redirect 対象。[roles/squid/files/allowlist.txt](roles/squid/files/allowlist.txt) に `*.tailscale.com` / `log.tailscale.io` を追加済。WireGuard data plane（UDP/41641）は redirect されない。tailnet（`100.64.0.0/10`）宛て TCP/80,443 を squid に横取りさせないよう [redirect script](roles/squid/templates/dev-egress-squid-redirect.j2) に RETURN 除外を追加済。
- **ufw 連携**: tailnet 経由の受信を成立させるため `ufw allow in on tailscale0` と `ufw allow 41641/udp` を追加する。**これにより従来の「有線直結 LAN のみ SSH 許可」方針が緩み、認証済み tailnet からのリモート SSH が可能になる**。誰が tailnet にいるかは Tailscale 側の認証 + ACL が制御するため、到達範囲は [Tailscale ACL](https://tailscale.com/kb/1018/acls) で最小化する（IdP に 2FA、可能なら Tailnet Lock も推奨）。
- リモート SSH が安定したら、公開ポートの SSH 露出（あれば）は閉じてよい。

### allowlist にドメインを足す

新しいツールが弾かれたとき（`handshake failed` / `tls handshake eof` / `error sending request for url` 等）は、そのエラー出力を project-scoped slash command [`/squid-allow`](.claude/commands/squid-allow.md) に渡す。エラーから到達失敗ホストを抽出し、`roles/squid/files/allowlist.txt` に最小一致の SNI regex を追記して検証まで行う（追加のみ・既存許可は緩めない）。sandbox 内は `no_new_privs` で sudo が使えないため、反映だけは sandbox 外のシェルで実行する:

```bash
sudo install -m644 roles/squid/files/allowlist.txt /etc/squid/allowlist.txt
sudo squid -k parse && sudo systemctl restart squid
```

### squid を一時停止する（実験用）

egress 制御を一時的に外して素通し（直出）にしたいときは [group_vars/all.yml](group_vars/all.yml) の `squid_enabled` を `false` にして再実行する（one-shot なら `-e squid_enabled=false`）。`false` のとき [site.yml](site.yml) は squid role を skip し、pre_tasks が iptables の `DEV_SQUID_OUTPUT` redirect chain を撤去したうえで `squid` / `dev-egress-squid-redirect` の両 service を stop + disable する。

> **注意**: `systemctl stop squid` 単体は禁物。redirect chain が残ったまま squid を止めると TCP/80,443 が listen していない intercept port に飛び続け、egress が全滅する（必ず redirect 撤去が先）。playbook 経由なら順序は自動で守られる。

サーバ上で手動で素通しにする場合（sandbox 外シェル, root）:

```bash
# 1. redirect 撤去 (これが先)
for b in /usr/sbin/iptables /usr/sbin/ip6tables; do
  while "$b" -t nat -D OUTPUT -j DEV_SQUID_OUTPUT 2>/dev/null; do :; done
  "$b" -t nat -F DEV_SQUID_OUTPUT 2>/dev/null || true
  "$b" -t nat -X DEV_SQUID_OUTPUT 2>/dev/null || true
done
# 2. service 停止
sudo systemctl disable --now dev-egress-squid-redirect squid
```

復旧は `squid_enabled` を `true` に戻して ansible-pull を再実行すれば squid role が両 service を再 enable し、redirect を貼り直す。

## 秘密情報

このリポジトリは公開。**秘密値（API キー・鍵・トークン）は一切置かない**。秘密が必要な処理は実行時にホスト側のセキュアな仕組み（SSM Parameter Store / `SendEnv`+`AcceptEnv` の session env 等）から取得する設計に寄せる。`dev_user_pubkey` は**公開鍵**なので git 管理してよい。
