---
title: "Gitコマンド、実は160種類以上あった"
emoji: "🔀"
type: "tech"
topics: ["git", "github", "コマンドライン", "CLI"]
published: false
published_at: 2025-12-27 22:00
---

## きっかけ

ふと「Gitコマンドって何種類あるんだろう」と気になって調べてみた。

普段使うのは `add`、`commit`、`push` くらい。せいぜい20種類程度だと思っていたが、公式ドキュメントを見たら **160種類以上** あって驚いた。

せっかくなので整理してみることに。

## Gitコマンドの分類

Gitのコマンドは2層構造になっている。

**Porcelain（磁器）**
ユーザーが直接使う「表側」のコマンド。約90種類。

**Plumbing（配管）**
Gitの内部で動く「裏側」のコマンド。約80種類。

洗面台に例えると、Porcelainは触る部分、Plumbingは見えない配管部分。なるほど。

## Porcelain コマンド

### Main Porcelain（主要コマンド）

日常的に使うやつ。約45種類。

**基本操作**

```bash
git init                        # リポジトリ初期化
git clone <url>                 # クローン
git clone --depth 1 <url>       # 浅いクローン（履歴1件のみ）
git add .                       # 全ファイルをステージング
git add -p                      # 変更を対話的にステージング
git commit -m "message"         # コミット
git commit --amend              # 直前のコミットを修正
git status                      # 状態確認
git status -s                   # 短縮表示
```

**ブランチ操作**

```bash
git branch                      # ブランチ一覧
git branch -a                   # リモート含む全ブランチ一覧
git branch -d <branch>          # ブランチ削除
git checkout <branch>           # ブランチ切替
git checkout -b <branch>        # ブランチ作成して切替
git switch <branch>             # ブランチ切替（新しい方）
git switch -c <branch>          # ブランチ作成して切替
git merge <branch>              # マージ
git merge --no-ff <branch>      # fast-forward禁止でマージ
git rebase <branch>             # リベース
git rebase -i HEAD~3            # 直近3コミットを対話的にリベース
```

**リモート操作**

```bash
git fetch                       # 全リモートから取得
git fetch origin                # originから取得
git pull                        # 取得してマージ
git pull --rebase               # 取得してリベース
git push                        # 送信
git push -u origin <branch>     # 上流ブランチを設定してプッシュ
git push --force-with-lease     # 安全な強制プッシュ
git remote -v                   # リモート一覧（URL付き）
git remote add origin <url>     # リモート追加
```

**履歴・差分**

```bash
git log                         # 履歴表示
git log --oneline               # 1行表示
git log --graph                 # グラフ表示
git log -p                      # 差分付きで表示
git diff                        # ワーキングツリーの差分
git diff --staged               # ステージングの差分
git diff HEAD~3                 # 3コミット前との差分
git show <commit>               # 特定コミットの内容
git blame <file>                # 行ごとの変更者表示
git blame -L 10,20 <file>       # 10〜20行目のみ表示
```

**取り消し系**

```bash
git reset HEAD <file>           # ステージング取り消し
git reset --soft HEAD~1         # コミット取り消し（変更は残す）
git reset --hard HEAD~1         # コミット取り消し（変更も消す）
git restore <file>              # ファイル復元
git restore --staged <file>     # ステージング取り消し
git revert <commit>             # 打ち消しコミット作成
git revert --no-commit <commit> # コミットせずに打ち消し
git cherry-pick <commit>        # 特定コミット適用
git cherry-pick -n <commit>     # コミットせずに適用
```

reset / restore / revert の違いがずっと曖昧だったのでメモ。

- `reset`: HEADを移動して履歴を書き換える
- `restore`: ファイルを復元するだけ（履歴に影響なし）
- `revert`: 打ち消すコミットを新しく作る（履歴は残る）

### Ancillary Commands（補助コマンド）

約30種類。

```bash
git config      # 設定
git remote      # リモート管理
git reflog      # 参照ログ
git gc          # ガベージコレクション
git help        # ヘルプ
git fsck        # DB検証
```

### 他システム連携

約11種類。SVNやPerforceとの連携用。

```bash
git svn    # Subversion連携
git p4     # Perforce連携
```

今の時代あまり使わなそう。

## Plumbing コマンド

普段は使わない内部コマンド。約80種類。

**Manipulation（操作系）**

```bash
git hash-object   # ハッシュ計算
git update-index  # インデックス更新
git write-tree    # ツリー作成
git commit-tree   # コミット作成
git read-tree     # ツリー読込
```

**Interrogation（調査系）**

```bash
git cat-file   # オブジェクトの中身表示
git ls-files   # インデックスのファイル一覧
git ls-tree    # ツリーの中身
git rev-parse  # リビジョン情報パース
git rev-list   # コミット一覧
```

**Syncing（同期系）**

```bash
git fetch-pack    # パック受信
git send-pack     # パック送信
git receive-pack  # プッシュ受取
git upload-pack   # フェッチ応答
```

`git commit` は内部でこれらのPlumbingコマンドを組み合わせて動いているらしい。

## コマンド数まとめ

| カテゴリ | 数 |
|---------|-----|
| Porcelain - Main | 約45 |
| Porcelain - Ancillary | 約30 |
| Porcelain - 連携系 | 約11 |
| Plumbing - Manipulation | 約22 |
| Plumbing - Interrogation | 約25 |
| Plumbing - Syncing | 約12 |
| Plumbing - Internal | 約19 |
| **合計** | **約160〜170** |

## 知らなかったコマンド

調べてて「これ便利そう」と思ったものをメモ。

**git bisect**
バグが混入したコミットを二分探索で特定。

```bash
git bisect start
git bisect bad
git bisect good v1.0.0
```

**git worktree**
別ディレクトリで別ブランチを同時に作業可能。

```bash
git worktree add ../hotfix hotfix-branch
```

**git reflog**
HEADの移動履歴。やらかした時の救世主。

```bash
git reflog
git reset --hard HEAD@{2}
```

**git shortlog**
コントリビューター別のコミット数。

```bash
git shortlog -sn
```

## 感想

160種類以上あるとはいえ、日常で使うのは20〜30種類程度。

ただ全体像を知っておくと「こういう時はこのコマンドがあるはず」と探せるようになる気がする。

Plumbingの存在を知ったことで、Gitの内部構造にも興味が出てきた。今度 `git cat-file` とか使って遊んでみたい。

## 参考

- https://git-scm.com/docs
- https://git-scm.com/docs/git#_git_commands

