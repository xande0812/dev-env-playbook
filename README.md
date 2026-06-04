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
| [site.yml](site.yml) | playbook。現状は疎通確認のみ。role は順次追加 |

## 秘密情報

このリポジトリは公開。**秘密値（API キー・鍵・トークン）は一切置かない**。秘密が必要な処理は実行時にホスト側のセキュアな仕組み（パラメータストア等）から取得する設計に寄せる。
