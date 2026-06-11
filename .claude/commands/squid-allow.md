---
description: egress proxy (squid) でブロックされた error 出力からドメインを抽出し allowlist に追加する
argument-hint: <mise/git/curl/squid access.log 等のエラー出力を貼り付け>
allowed-tools: Read, Edit, Bash(grep:*), Bash(getent:*), Bash(git diff:*), Bash(git --no-pager diff:*)
---

あなたはこの repo (dev-env-playbook) の squid egress allowlist を保守する。
egress proxy (squid) は `roles/squid/files/allowlist.txt` の **SNI 正規表現 allowlist**
(1 行 1 entry, anchored regex) でホスト単位に許可判定する。allowlist 外は
`ssl_bump terminate` で TLS error になり、tool 側は "handshake failed" /
"tls handshake eof" / "error sending request for url" 等で失敗する。

## 入力 (ブロックされて失敗したコマンドのエラー出力)

<error>
$ARGUMENTS
</error>

`$ARGUMENTS` が空なら、エラー出力を貼り付けてもらうよう促して終了する。

## 手順

1. **到達失敗ホストを抽出する。** error 文字列から、許可が必要なホスト名を全て拾う:
   - URL 形式: `https://HOST/...`, `error sending request for url (https://HOST/...)`
   - git: `git ls-remote ... https://HOST/...`, `unable to access 'https://HOST/...'`
   - squid access.log の `CONNECT IP:443`(ホスト名でなく IP)の場合: error 中の
     他の手がかり(同時に出ている URL や対象ツール)から候補ホストを推定し、
     `getent ahosts <候補ホスト>` で forward 解決して、その IP が log の IP と
     一致するか確認する。一致を確認できない IP は **勝手に追加しない**。何のホストか
     ユーザーに確認する。

2. **SNI regex に変換する(最小一致)。** 各ホストを anchored regex にする:
   - 基本: `^host$`(ドットは `\.` でエスケープ。例 `dl.google.com` → `^dl\.google\.com$`)
   - apex と subdomain の両方に到達するツールなら両方足す
     (例 `github.com` は apex 用 `^github\.com$` と subdomain 用 `^.*\.github\.com$`)。
   - `.*` 単体や TLD 丸ごとのような **広すぎる pattern は使わない**。必要なホストだけ。

3. **既出を除外する。** `roles/squid/files/allowlist.txt` を Read し、各ホストが既に
   既存行にマッチするか `grep -E -i` で確認する。マッチするものは skip。

4. **不足分だけ追加する。** ファイルの近い section
   (`# ---- Package registries ----` / `# ---- GitHub ----` 等)に、
   **どのツール・何の用途か** の短い由来コメントを 1 行添えて追記する。
   既存 entry の **削除・緩和は一切しない(追加のみ)**。

5. **検証する。** 以下で新規ホストが ALLOW、`example.com` が DENY のままを確認:
   ```bash
   for h in <追加した各ホスト> example.com; do
     grep -E -i -q -f roles/squid/files/allowlist.txt <<<"$h" && echo "ALLOW $h" || echo "DENY  $h"
   done
   ```
   追加ホストが ALLOW にならない／`example.com` が ALLOW になる場合は regex を直す。

6. **差分を見せる。** `git --no-pager diff roles/squid/files/allowlist.txt` を表示する。

7. **反映方法を案内する。** sandbox 内は `no_new_privs` で sudo が使えないため、
   反映は **ユーザーが sandbox 外のシェルで** 実行する。次を提示する:
   ```bash
   sudo install -m644 roles/squid/files/allowlist.txt /etc/squid/allowlist.txt
   sudo squid -k parse && sudo systemctl restart squid
   ```
   その後、元の失敗コマンドを再実行して直ったか確認するよう伝える。

## 制約 / 運用ルール (`allowlist.txt` 冒頭のルールに従う)

- **追加のみ。** 既存の許可を消したり広げたりしない。
- write 系(production への CREATE/UPDATE/DELETE)credential を要するドメインは
  この allowlist に載せない判断を優先する。判断に迷うドメインは追加前にユーザーに確認。
- agent の WebFetch 用 documentation domain を足す場合は、`settings.json` の
  `WebFetch(domain:...)` allowlist も同時更新が要る点をユーザーに念押しする
  (このコマンドは squid 側 = `allowlist.txt` のみ編集する)。
- commit はユーザーの指示があるときだけ行う(network 操作は sandbox 外)。
