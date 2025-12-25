---
title: "GitHub CLI(gh)、ターミナルからGitHubを操作できて便利"
emoji: "🐙"
type: "tech"
topics: ["git", "github", "cli", "コマンドライン"]
published: false
published_at: 2025-12-28 22:00
---

## きっかけ

昨日Gitコマンドを調べたので、今日はGitHub CLI（gh）を調べてみた。

@[card](https://zenn.dev/saku_238/articles/2025-12-27_git-commands)

普段PRを作るときはブラウザを開いていたが、ghコマンドを使えばターミナルで完結するらしい。

## GitHub CLIとは

2020年にGitHubがリリースした公式CLIツール。

公式サイト: https://cli.github.com/

GitとGitHub CLIは別物。

- **Git**: バージョン管理システム（Linus Torvaldsが開発）
- **gh**: GitHubの操作に特化したCLI（GitHubが開発）

## インストール

```bash
# Mac
brew install gh

# Windows
winget install --id GitHub.cli

# 認証
gh auth login
```

## コマンド数

約30種類のトップレベルコマンド、サブコマンドを含めると150種類以上。

Gitの160種類と比べると少ないが、GitHub操作に特化している分シンプル。

## コマンド分類

### Core commands（10種類）

日常的に使うやつ。

```bash
gh auth login                      # ログイン
gh auth logout                     # ログアウト
gh auth status                     # 認証状態確認
gh browse                          # 現在のリポジトリをブラウザで開く
gh browse --settings               # 設定ページを開く
gh browse <file>                   # 特定ファイルを開く
gh codespace create                # Codespace作成
gh codespace list                  # Codespace一覧
gh gist create <file>              # Gist作成
gh gist list                       # Gist一覧
gh issue list                      # Issue一覧
gh issue create                    # Issue作成
gh org list                        # Organization一覧
gh pr list                         # PR一覧
gh pr create                       # PR作成
gh project list                    # Projects一覧
gh release list                    # リリース一覧
gh release create <tag>            # リリース作成
gh repo clone <repo>               # クローン
gh repo create <name>              # リポジトリ作成
```

### GitHub Actions commands（3種類）

```bash
gh cache list                      # キャッシュ一覧
gh cache delete <id>               # キャッシュ削除
gh run list                        # ワークフロー実行一覧
gh run view <id>                   # 実行詳細
gh run view <id> --log             # ログ表示
gh run watch <id>                  # リアルタイムで監視
gh run rerun <id>                  # 再実行
gh workflow list                   # ワークフロー一覧
gh workflow run <workflow>         # 手動実行
gh workflow enable <workflow>      # ワークフロー有効化
gh workflow disable <workflow>     # ワークフロー無効化
```

### Additional commands（17種類）

```bash
gh alias set <alias> '<command>'   # エイリアス設定
gh alias list                      # エイリアス一覧
gh api <endpoint>                  # API呼び出し
gh api <endpoint> --jq '<query>'   # jqでフィルタリング
gh api <endpoint> -X POST          # POSTリクエスト
gh config set editor vim           # エディタ設定
gh config list                     # 設定一覧
gh extension install <repo>        # 拡張機能インストール
gh extension list                  # 拡張機能一覧
gh search repos <query>            # リポジトリ検索
gh search repos <query> --limit 50 # 件数指定
gh search issues <query>           # Issue検索
gh search prs <query>              # PR検索
gh search code <query>             # コード検索
gh secret set <name>               # シークレット設定
gh secret list                     # シークレット一覧
gh ssh-key add <file>              # SSHキー追加
gh ssh-key list                    # SSHキー一覧
gh status                          # 自分に関連する通知表示
gh variable set <name>             # 変数設定
gh variable list                   # 変数一覧
```

## よく使いそうなコマンド

### PR関連

```bash
# PR作成
gh pr create --title "タイトル" --body "説明"

# PR一覧
gh pr list

# PRの差分確認
gh pr diff 123

# PRをローカルにチェックアウト
gh pr checkout 123

# PRマージ
gh pr merge 123
```

### Issue関連

```bash
# Issue作成
gh issue create --title "タイトル" --body "説明"

# Issue一覧
gh issue list

# Issue詳細
gh issue view 123
```

### リポジトリ関連

```bash
# クローン（URLを覚えなくていい）
gh repo clone owner/repo

# リポジトリ作成
gh repo create my-repo --public

# ブラウザで開く
gh browse
```

### 検索

```bash
# リポジトリ検索
gh search repos "keyword"

# Issue検索
gh search issues "keyword"

# PR検索
gh search prs "keyword"

# コード検索
gh search code "function"
```

### GitHub Actions

```bash
# ワークフロー一覧
gh workflow list

# 実行履歴
gh run list

# ワークフロー実行
gh workflow run ci.yml

# ログ確認
gh run view 123 --log
```

## Git vs gh の使い分け

| 操作 | Git | gh |
|------|-----|-----|
| コミット | `git commit` | ❌ |
| プッシュ | `git push` | ❌ |
| ブランチ作成 | `git branch` | ❌ |
| PR作成 | ❌ | `gh pr create` |
| Issue作成 | ❌ | `gh issue create` |
| クローン | `git clone <url>` | `gh repo clone owner/repo` |

基本的な使い分け：
- **ローカル操作** → Git
- **GitHub操作** → gh

## 便利だと思った点

**1. URLを覚えなくていい**

```bash
# これが
git clone https://github.com/owner/repo.git

# こうなる
gh repo clone owner/repo
```

**2. ブラウザを開かなくていい**

PRの作成、Issue確認、ワークフローの実行がターミナルで完結。

**3. API呼び出しが簡単**

```bash
# 自分のリポジトリ一覧
gh api user/repos --jq '.[].name'

# 特定リポジトリの情報
gh api repos/owner/repo
```

## コマンド数まとめ

| カテゴリ | 数 |
|---------|-----|
| Core commands | 10 |
| Actions commands | 3 |
| Additional commands | 17 |
| **トップレベル合計** | **30** |
| サブコマンド含む | 150+ |

## 感想

Gitは必須、ghはあると便利という位置づけ。

PRやIssueをよく触るならghを入れておくと生産性が上がりそう。特に `gh pr checkout` と `gh browse` は地味に便利。

ワークフローの確認もターミナルでできるのは良い。ブラウザのActionsタブを開く手間が省ける。

## 参考

- https://cli.github.com/
- https://cli.github.com/manual/

