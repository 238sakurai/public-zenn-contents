---
title: "Datadogのpupと戯れあったメモ"
emoji: "🐶"
type: "tech"
topics: ["datadog", "cli", "ai", "monitoring"]
published: true
---

## はじめに

みなさん、犬は好きですか？我が家は愛犬家一家です。

さて、Datadogから子犬（pup）が誕生しました。その名も **[pup](https://github.com/DataDog/pup)** コマンドです。

Datadogでの調査といえば、ブラウザでUIを開いてポチポチ操作するのが一般的ですよね。それでも十分ですが、**ターミナル操作ができるAIエージェント（CursorやClaude Code等）と組み合わせて調査を自動化**できたら、もっと速くなると思いませんか？

今回、この pup CLI を使い、Cursorエージェントに「このエラー調査して」とお願いするだけで、原因特定まで持っていけた体験を紹介します。その過程で見えてきた **pup CLIに欲しい機能** もまとめます。

## pup CLI とは

[DataDog/pup](https://github.com/DataDog/pup) は、今週公開されたばかりのDatadog公式Go製CLIです。ログ検索、APMサービス情報、モニター管理などをターミナルから操作できます。

```bash
# Go でインストール
go install github.com/DataDog/pup@latest

# Homebrew でもインストール可能
brew install datadog/pack/pup

# GitHub Releases からバイナリを直接ダウンロードすることもできます
# https://github.com/DataDog/pup/releases
```

認証はOAuth2で行います。

```bash
# 利用リージョンに合わせて DD_SITE を設定
export DD_SITE="ap1.datadoghq.com"

# ブラウザが開いてDatadogアカウントで認証
pup auth login

# 認証状態の確認
pup auth status
```

## 実際の調査フロー

### 発端：Datadog APMのトレースURLを渡された

ある日、SlackでDatadog APMのURLが飛んできました。

```
https://ap1.datadoghq.com/apm/trace/xxxxxxxxxxxxxxxx...
```

「このエラー、何が起きてるか調べて」——よくある依頼です。

### Step 1: トレースIDからログを引く

最初に `pup traces` コマンドを試しましたが、現時点では未実装でした。そこで **ログ検索でtrace_idを指定**するアプローチに切り替えます。

```bash
pup logs search \
  --query="trace_id:<your-trace-id>" \
  --from="7d" \  # trace_idはユニークなので、時間範囲は広めでOK
  --limit=10
```

これでトレースに紐づくログが取得でき、エラーの詳細が分かりました。

### Step 2: エラー内容の分析

pup の出力はJSON形式ですが、ネストが深く `jq` だけでは扱いづらいため、パイプでPythonに渡して整形します。

```bash
pup logs search --query="trace_id:<id>" --from="7d" --limit=10 \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
for log in data.get('data', []):
    attrs = log['attributes']['attributes']
    err = attrs.get('error', {})
    print(f'Time: {attrs.get(\"time\", \"\")}')
    print(f'Service: {attrs.get(\"service\", \"\")}')
    print(f'Error: {err.get(\"message\", \"\")[:200]}')
    print('---')
"
```

結果、PostgreSQLの **`23505` (unique constraint violation)** が原因と判明。あるテーブルに対して、2つの非同期処理が同じキーで同時にINSERTを実行し、ユニーク制約違反で競合していたことが分かりました。

### Step 3: 影響範囲の調査

同じエラーがどれくらい発生しているか、30日分を調べます。

```bash
pup logs search \
  --query='"unique constraint" status:error' \
  --from="30d" \
  --limit=500 \
  | python3 -c "
import json, sys
from collections import Counter
data = json.load(sys.stdin)
logs = data.get('data', [])
days = Counter()
for l in logs:
    ts = l['attributes']['attributes'].get('time', '')
    days[ts[:10]] += 1
print(f'=== 30日間: {len(logs)}件 ===')
for d in sorted(days.keys()):
    print(f'  {d}: {days[d]}件')
"
```

### Step 4: 時系列でタイムラインを追う

エラー前後のイベントを時系列で追跡し、競合の因果関係を特定します。

```bash
pup logs search \
  --query='"<person_id>" "<処理名>"' \
  --from="2d" \
  --limit=50 \
  --sort=asc
```

これにより、2つの処理が数ミリ秒差で同じレコードを INSERT しようとしていたことが確認できました。

## Cursor ルールで調査手順を再利用可能に

この調査フローを毎回手動で組み立てるのは非効率です。そこで **Cursorルール** として定義しました。

```markdown
# .cursor/rules/datadog-investigation.mdc
---
description: Datadog pup CLIを使ったエラー調査手順
globs: docs/**/*
alwaysApply: false
---

## 調査フロー
1. DD_SITE設定 → pup auth status で認証確認
2. logs search --query="trace_id:<id>" でトレースからログ検索
3. エラー内容を抽出・分類
4. 影響範囲（日別件数、環境別、Organization別）を集計
5. タイムラインで前後イベントを追跡
```

これにより、次回からは「このDatadogエラー調べて」と言うだけで、AIエージェントが同じ手順を自動で実行してくれます。

## ぶつかった壁と pup に欲しい機能

まだ登場したばかりのツールなので当然ではありますが、実際に使い込んでみるといくつかの課題も見えてきました。今後の発展に期待を込めて、気づいた点をまとめます。

### 壁1: 結果0件でJSONが返らない 🚨 最重要

```bash
$ pup logs search --query="..." --from="7d" --limit=5
No logs found matching your query.

Tips:
- Try a broader time range (e.g., --from="30d")
...
```

AIエージェントがパイプで `python3` に渡すと、**`JSONDecodeError`** で即死します。これが調査中に何度も発生し、最大の障害でした。

**欲しい挙動**: 0件でも `{"data": []}` を返す。あるいは `--output=json` フラグで強制JSON出力。

### 壁2: 集計機能の存在に気づかずPythonスクリプトを書いていた

「日別のエラー件数」「エラーパターン別の内訳」——これらを見るために、毎回 Python ワンライナーをパイプで書いていました。

……が、実は **`pup logs aggregate` コマンドが既に存在していました**。記事を書いた後に気づきました。

```bash
# ステータス別のカウント
pup logs aggregate --query="*" --from="1h" --compute="count" --group-by="status"

# エラーメッセージ別の集計
pup logs aggregate --query="status:error" --from="7d" --compute="count" --group-by="@error.message"

# 総件数だけ取得
pup logs aggregate --query="status:error" --from="30d" --compute="count"

# サービス別の平均レスポンスタイム
pup logs aggregate --query="*" --from="1h" --compute="avg(@duration)" --group-by="service"

# 99パーセンタイルのレイテンシ
pup logs aggregate --query="service:api" --from="2h" --compute="percentile(@duration, 99)"
```

`--compute` には `count`, `avg`, `sum`, `min`, `max`, `cardinality`, `percentile` などが使え、`--group-by` でフィールド別のグルーピングが可能です。Step 3 の影響範囲調査もこれで大幅に楽になりそうです。

ただし、日別・時間帯別といった **時系列でのグルーピング**（Datadog UIの "Group by > Timestamp" 相当）は `--group-by` だけでは難しく、その用途では引き続き Python スクリプトが必要になります。

### 壁3: traces コマンドが未実装

APMトレースのURLから調査を始めるのは非常に一般的なユースケースですが、`pup traces` は `under development` でした。

**欲しい機能**:

```bash
pup traces get <trace_id>
# → スパン一覧、各スパンのレイテンシ、エラー情報を出力
```

### 壁4: テーブル出力のカラムが固定

`-o table` で表形式出力ができますが、表示カラムがカスタマイズできません。`env`、`organizationId` など頻出の属性を見たいのに、毎回JSONをパースする必要がありました。

**欲しい機能**:

```bash
pup logs search --query="status:error" -o table \
  --columns="time,dd.env,organizationId,error.message"
```

### 壁5: タイムゾーンとISO 8601

出力が常にUTCで、JSTへの変換を毎回手動で行いました。

実は調べてみると、**部分的には既に対応されていました**。

まず **RFC3339（ISO 8601）形式は `--from` / `--to` で既にサポートされています**:

```bash
# これは動く！
pup logs search --query="..." \
  --from="2026-02-13T04:00:00Z" \
  --to="2026-02-13T05:00:00Z"
```

また **`pup logs query` コマンドには `--timezone` フラグが存在**します（ただし `logs search` にはありません）:

```bash
# logs query なら --timezone が使える
pup logs query --query="service:web" --from="4h" --timezone="Asia/Tokyo"
```

残る課題としては、`logs search` での `--timezone` サポートと、出力側のタイムゾーン変換です。

## 戯れあった所感

今回触ってみて、正直、一番つらかったのは **空結果でJSONが返らない** でした。Cursorに調査させるとパイプが壊れて何度もやり直しになりました。集計機能は `logs aggregate` を見つけたので解消しましたが、時系列グルーピングはまだ課題です。

CLIって人間が使うものとして設計されてきたけど、これからはAIエージェントが叩くことも増えていくんだろうなと。pup に限らず、「マシンが読みやすい出力」を意識したCLI設計が大事になりそうだなと感じました。

## まとめ

pup CLI、登場したばかりですが普通にDatadogのログ調査に使えました。Cursorルールと組み合わせると「このエラー調べて」で勝手に調査が進むので体験はかなり良いです。

まだ荒削りな部分はあるものの、開発が活発なプロジェクトなので今後に期待しています。気が向いたらIssueでも立ててみようかなと思います。

冒頭で「我が家は愛犬家一家です」と書きましたが、Datadogの子犬（pup）もAIエージェントと組み合わせれば立派な **AI犬** ですね。よろしくメカドッグ！🐕‍🦺
