---
name: virtual-monorepo
description: tolo-virtual-monorepo の中で TOLO の各リポジトリをクローン・更新する手順。作業に要るリポジトリが直下にないとき、コードを読む・変える前に最新にするとき、どのリポジトリに何があるかを決めるときに使う。
---

# バーチャルモノレポ（tolo-virtual-monorepo）

このリポジトリの直下に、TOLO の各リポジトリを独立した git リポジトリとしてクローンする。
ルートのリポジトリが管理するのは `mise.toml`、`Taskfile.yml`、`.gitignore`、`.agents/`、`.claude/` だけで、クローンしたディレクトリは `.gitignore` で管理外にしてある。
クローンと更新は Taskfile の task で行う。task の名前とグループの中身は `task --list` と `Taskfile.yml` で確かめ、このスキルの記述と食い違うときは `Taskfile.yml` に従う。

## 手順

### 1. 道具を確かめる

```sh
task --version
gh auth status
```

`task` が見つからなければ、ルートで `mise trust && mise install` を実行する。
`gh` が未ログインなら、ユーザーに `! gh auth login` の実行を頼む。private のリポジトリ（hoakari-root、PoC 群）はログインしないとクローンできない。

完了条件: 2 つのコマンドがどちらも成功する。

### 2. 要るリポジトリを決める

依頼の内容から、読む・変えるリポジトリを決める。

| 知りたいこと・変えるもの | リポジトリ |
|---|---|
| 仕様、ADR、ドメイン用語 | `hoakari-root`（読み方は hoakari-root の `spec-lookup` スキル） |
| 個々のサービスの実装 | `tolo-<サービス>`（service-gateway、tenant-management、graph-authoring、observation、operation、flow-control、edge-bridge、guest、realtime、idp） |
| サービス同士をつないだ compose の結合環境 | `tolo` |
| Kubernetes の manifest | `tolo-manifest` |
| サービスが共有する Go のライブラリ・proto | `tolo-kernel-proto`、`internal-jwt-handling`、`protoc-gen-authz-go`、`jetstream-http-push`、`tolo-relation-stub`、`go-service-template` |
| Web フロントエンド | `tolo-web`、`web-crowd-detection-utils` |
| スタッフ用モバイルアプリ | `tolo-staff-app` |
| CI の共通設定 | `actions`、`renovate-config` |

サービス間のやり取りを調べるときは、両端のサービスのリポジトリと `hoakari-root` を揃える。
`task --list` にないリポジトリ（アーカイブ済みのもの、`tolo-signaling`）は、ユーザーが名指ししたときだけ扱う。TOLO のリポジトリなのに `task --list` にないものを見つけたら、`Taskfile.yml` への追加をユーザーに提案する。

完了条件: 読む・変えるリポジトリが名前で挙がっている。

### 3. ないものをクローンする

```sh
task clone:<リポジトリ名> [clone:<リポジトリ名> ...]
```

依頼が領域全体（「バックエンド全部」「Web まわり」など）にわたるときは、グループの task（`clone:backend`、`clone:web` など）を使う。グループの task は `hoakari-root` も一緒にクローンする。
既にあるリポジトリはスキップされるので、有無を確かめずに実行してよい。

完了条件: 手順 2 で挙げたリポジトリがすべて直下にある。

### 4. 最新にする

読む・変える前に、そのセッションでまだ更新していないリポジトリを更新する。

```sh
git -C <リポジトリ名> pull --ff-only   # 1 つだけのとき
task pull                               # クローン済みのすべて
```

`task pull` は失敗したリポジトリがあっても残りを続け、最後に `pull failed: <名前> ...` を出す。
失敗したリポジトリは手元の状態のまま残し、名前と git の出力が示す理由（ローカルのコミットのずれ、コミットしていない変更、upstream がない、など）をユーザーに報告する。reset、rebase、stash、merge で解消するのは、ユーザーに頼まれたときだけにする。
失敗したリポジトリや、既定のブランチ以外がチェックアウトされているリポジトリを読んで答えるときは、どのブランチのどの状態を読んだかを答えに書く（`git -C <リポジトリ名> status -sb`）。

完了条件: 手順 2 の各リポジトリについて、更新できたか、失敗したならその理由が報告されている。

### 5. 各リポジトリの中で作業する

クローンしたリポジトリは、それぞれ独立したリポジトリとして扱う。

- そのリポジトリの `CLAUDE.md`、`AGENTS.md`、`.agents/skills/` があれば従う
- git の操作は `git -C <リポジトリ名> ...` か、そのディレクトリに入って行う。コミットはそのリポジトリに入る
- ルートのリポジトリへのコミットは Taskfile やこのスキルなど、バーチャルモノレポの道具を変えたときだけにする
