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

Datadogでの調査といえば、ブラウザでUIを開いてポチポチ操作するのが一般的ですよね。それでも十分ですが、**AIエージェント（Cursor）と組み合わせて調査を自動化**できたら、もっと速くなると思いませんか？

今回、この pup CLI を使い、Cursorエージェントに「このエラー調査して」とお願いするだけで、原因特定まで持っていけた体験を紹介します。その過程で見えてきた **pup CLIに欲しい機能** もまとめます。

## pup CLI とは

[DataDog/pup](https://github.com/DataDog/pup) は、今週公開されたばかりのDatadog公式Go製CLIです。ログ検索、APMサービス情報、モニター管理などをターミナルから操作できます。

```bash
# Go でインストール
go install github.com/DataDog/pup@latest

# Homebrew でもインストール可能
brew install datadog/tap/pup

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

### 壁2: 集計にはPythonスクリプトが必須

「日別のエラー件数」「エラーパターン別の内訳」「時間帯ヒストグラム」——これらを見るために、毎回 Python ワンライナーをパイプで書いていました。

**欲しい機能**:

```bash
# 日別カウント
pup logs search --query="status:error" --from="30d" --group-by=date

# 時間帯別
pup logs search --query="status:error" --from="1d" --group-by=hour

# エラーメッセージ別
pup logs search --query="status:error" --from="7d" --group-by="@error.message"

# 件数だけ
pup logs search --query="status:error" --from="30d" --count-only
```

Datadog UIではこれらのグルーピングは標準機能です。CLIでも使えると、スクリプト不要で調査速度が大幅に上がります。

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

出力が常にUTCで、JSTへの変換を毎回手動で行いました。また `--from` に ISO 8601 形式が使えず、特定時刻の範囲指定が面倒でした。

**欲しい機能**:

```bash
# タイムゾーン指定
pup logs search --query="..." --timezone="Asia/Tokyo"

# ISO 8601 での絶対時刻指定
pup logs search --query="..." \
  --from="2026-02-13T04:00:00Z" \
  --to="2026-02-13T05:00:00Z"
```

## 個人的な優先度

正直、一番つらかったのは **空結果でJSONが返らない** 問題です。Cursorに調査させるとパイプが壊れて何度もやり直しになりました。次点で集計機能。毎回Pythonワンライナーを書くのは地味にしんどい。

| 優先度 | 機能 | 体感 |
|---|---|---|
| 🔴 高 | 空結果時も有効なJSON出力 | これがないとパイプラインが毎回壊れる |
| 🔴 高 | 集計・グルーピング機能 | Pythonワンライナー書くのが毎回つらい |
| 🟡 中 | traces get コマンド | APMのURLから調査を始めることが多いので |
| 🟡 中 | table出力のカラム指定 | 欲しいカラムが出なくて結局JSONを見る |
| 🟢 低 | タイムゾーン・ISO 8601 | あったら嬉しいけどなくても困らない |

## AIエージェントにCLIを使わせてみて思ったこと

今回やってみて感じたのは、CLIの出力が機械的に扱いやすいかどうかで、AIエージェントとの相性がかなり変わるということでした。

具体的には、0件のときに人間向けのメッセージが出るとパイプが壊れます。これは人間が使う分には親切なんですが、エージェントに使わせると途端に困る。JSONで一貫して返してくれるだけで、だいぶ楽になるはずです。

CLIって人間が使うものとして設計されてきたけど、これからはAIエージェントが叩くことも増えていくんだろうなと。pup に限らず、「マシンが読みやすい出力」を意識したCLI設計が大事になりそうだなと感じました。

## まとめ

pup CLI、登場したばかりですが普通にDatadogのログ調査に使えました。Cursorルールと組み合わせると「このエラー調べて」で勝手に調査が進むので体験はかなり良いです。

まだ荒削りな部分はあるものの、開発が活発なプロジェクトなので今後に期待しています。気が向いたらIssueでも立ててみようかなと思います。

冒頭で「我が家は愛犬家一家です」と書きましたが、Datadogの子犬（pup）もAIエージェントと組み合わせれば立派な **AI犬** ですね。よろしくメカドッグ！🐕‍🦺
