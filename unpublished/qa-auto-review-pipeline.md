---
title: "バグの振り返りを「仕組み」にしたら、チームが勝手に改善し始めた"
emoji: "⚙️"
type: "tech"
topics: ["QA", "品質分析", "自動化", "Qase", "CI"]
published: false
publication_name: "bitkey_dev"
---

:::message
ダッシュボードを作ったのに誰も見てくれない。品質データを集めたのに次のアクションに繋がらない。心当たりはないでしょうか。この記事では、バグの振り返りを「仕組み」に変えて、チームが自然に改善を回し始めた方法を紹介します。
:::

<!--
## コンセプト

### 一言で
週15分のバグ振り返りを、cron→API→AI分析→Slack通知で自動化したら、
開発者を巻き込んだ振り返りが「逃げ道なし」で回り始めた。

### ターゲット
- qa-first-analysis-ritualを読んで「手動は続かない」と思った人
- QAの振り返りに開発者を巻き込みたいが、会議を増やしたくない人
- テスト管理ツールのデータを活用したいが、ダッシュボードを見てもらえない人

### キーメッセージ
1. 「見に来い」ではなく「届ける」。ダッシュボードの限界
2. 自動パイプライン: cron → テスト管理ツール → AI分析 → Slack通知
3. 開発者のチャンネルに流すことで「逃げ道」をなくす

### 差別化ポイント
- 実装の具体例（擬似コード+構成図）を見せる
- ダッシュボード→通知への発想転換が核心
- 「AI分析」はオプション。ルールベースでも十分という現実的なスタンス

### 既存記事との関係
- qa-first-analysis-ritual → 手動の振り返り（前提）。本記事はその自動化
- quality-analysis-starting-point → データ品質の課題。本記事は「不完全でも回す」実践
- test-equals-qa → QA＝仕組みをつくる。本記事はその具体例

### 図解案
- パイプライン全体図（cron→API→分析→Slack）
- Before/After: 「ダッシュボードを見に行く」vs「Slackに届く」
- Slackメッセージのモック画像
-->

## ダッシュボードの限界

自分もダッシュボードを作った。データも入っている。グラフもきれい。

でも、**誰も見ていない。**

ダッシュボードは「見に来い」モデルです。忙しい開発者が、自分からダッシュボードを開いて、バグの傾向を分析することはありません。QAチームですら、作ったダッシュボードを定期的に見ているか怪しい。

[前回の記事](https://zenn.dev/bitkey_dev/articles/qa-first-analysis-ritual)で「週15分のバグ振り返り」を提案しました。でも正直に言います。自分の現場でも**手動の習慣は、3週間で途切れました。**

忙しくなったら後回しになる。メンバーが変わったら引き継がれない。品質の振り返りは、常に「優先度が低い」扱いを受けます。

だから、**仕組みにする**。人の意志に頼らず、パイプラインとして自動で回す。

## 全体像

```
┌──────────┐     ┌──────────────┐     ┌──────────┐     ┌──────────┐
│  cron    │────▶│ テスト管理    │────▶│  分析    │────▶│  Slack   │
│ (毎週月曜)│     │ ツールAPI    │     │ (3つの問い)│     │  通知    │
└──────────┘     └──────────────┘     └──────────┘     └──────────┘
                   Qase / Jira          ルールベース      開発者チャンネル
                   から不具合取得        or AI           に直接届く
```

## 毎週決まった時間にトリガーする

毎週月曜の朝、自動でパイプラインを起動します。

```bash
# crontab -e
# 毎週月曜 9:00 にバグ振り返りレポートを生成・送信
0 9 * * 1 /usr/bin/python3 /path/to/weekly_bug_review.py
```

なぜ月曜の朝か。週の始まりに前週の振り返りを届けることで、今週のテスト計画に反映できるからです。

## テスト管理ツールAPIで直近のバグを取得する

テスト管理ツールから、直近1週間の不具合データを自動取得します。

```python
# 擬似コード: Qase APIからDefectsを取得する例
import urllib.request
import json
from datetime import datetime, timedelta

def fetch_recent_defects(api_token, project_codes):
    """直近7日間の不具合を全プロジェクトから取得"""
    one_week_ago = (datetime.now() - timedelta(days=7)).strftime("%Y-%m-%d")
    defects = []

    for code in project_codes:
        url = f"https://api.qase.io/v1/defect/{code}?status=open&limit=100"
        req = urllib.request.Request(url, headers={"Token": api_token})
        resp = json.loads(urllib.request.urlopen(req).read())

        for d in resp.get("result", {}).get("entities", []):
            if d["created_at"][:10] >= one_week_ago:
                defects.append({
                    "project": code,
                    "title": d["title"],
                    "severity": d.get("severity", "未設定"),
                    "created_at": d["created_at"],
                    "status": d["status"],
                })
    return defects
```

ポイントは**既に毎日取得しているデータを流用する**こと。自分のチームでは、Qaseのデータをダッシュボード用に定期取得する仕組みが既にありました。新しいAPIを叩く必要はなく、既存の取得処理に振り返り用のフィルタを足すだけで済みました。

## 3つの問いに自動で答える

取得したデータに対して、[前回の記事](https://zenn.dev/bitkey_dev/articles/qa-first-analysis-ritual)で紹介した3つの問いを自動で投げます。

### ❶ どこに多い？（偏り検出）

```python
def find_concentration(defects):
    """プロジェクト・領域ごとの集中度を検出"""
    from collections import Counter
    by_project = Counter(d["project"] for d in defects)

    total = len(defects)
    alerts = []
    for project, count in by_project.most_common(3):
        ratio = count / total * 100
        if ratio >= 30:  # 全体の30%以上が集中していたらアラート
            alerts.append(f"⚠️ {project} にバグが集中 ({count}件, {ratio:.0f}%)")
    return alerts
```

### ❷ 前にも見た？（再発検出）

```python
def find_recurrence(this_week, last_week):
    """前週と同じプロジェクト・同じSeverityのバグが再発しているか"""
    this_areas = set((d["project"], d["severity"]) for d in this_week)
    last_areas = set((d["project"], d["severity"]) for d in last_week)
    recurring = this_areas & last_areas  # 両方に出現 = 再発パターン

    alerts = []
    for project, severity in recurring:
        alerts.append(f"🔁 {project} で {severity} バグが先週に続いて発生")
    return alerts
```

### ❸ 防げた？（予防可能性の示唆）

ここが**AIをつかうと効果的なポイント**です。

```python
def suggest_prevention(defects):
    """ルールベース or AIでバグの予防可能性を判定"""
    alerts = []

    # ルールベース: Severityが高いバグは予防の余地を示唆
    critical_bugs = [d for d in defects if d["severity"] in ("critical", "blocker")]
    if critical_bugs:
        alerts.append(
            f"🛡️ Critical以上が {len(critical_bugs)}件。"
            f"設計レビューやリスク分析で防げた可能性を検討してください"
        )

    # AI強化版: バグのタイトルからパターンを分析
    # prompt = f"以下のバグ一覧から、設計・要件段階で防げた可能性のあるものを特定してください:\n{titles}"
    # response = call_llm(prompt)

    return alerts
```

**ルールベースで十分始められます。** AIは「あると嬉しい」であって「なければ動かない」ではありません。まずルールベースで回して、精度を上げたくなった時点でAIを足します。この順番が大事です。

## 分析結果をSlackで開発者に届ける

分析結果をSlackに投稿します。**QAチャンネルではなく、開発者のチャンネルに。**

:::details Webhookとは
外部サービスからのHTTPリクエストでイベントを受け取る仕組みです。Slack Webhookをつかうと、スクリプトからSlackチャンネルにメッセージを投稿できます。
:::

```python
def send_to_slack(webhook_url, defects, alerts):
    """分析結果をSlackに送信"""
    message = f"""📊 *週次バグ振り返り* ({len(defects)}件)

*今週の不具合サマリー*
• 新規: {len(defects)}件
• Critical以上: {len([d for d in defects if d['severity'] in ('critical','blocker')])}件

*3つの問い*
"""
    for a in alerts:
        message += f"  {a}\n"

    message += "\n_このレポートは自動生成です。15分だけ、チームで話してみてください。_"

    data = json.dumps({"text": message}).encode()
    req = urllib.request.Request(
        webhook_url, data=data,
        headers={"Content-Type": "application/json"}
    )
    urllib.request.urlopen(req)
```

### なぜ開発者チャンネルなのか

QAチャンネルに送ると、QAだけが見て終わります。開発チャンネルに送ることで:

- 開発者の目に強制的に入る（逃げ道がない）
- 「今週こんなバグ出てるんだ」という認識が自然に共有される
- QAから「見てください」と頼む必要がなくなる

ダッシュボードは「見に来い」。Slack通知は**「届ける」**。この違いが、振り返りの継続率を決めます。

## 通知メッセージの設計

Slackに届くメッセージは**15秒で読める長さ**にします。

```
📊 週次バグ振り返り (12件)

今週の不具合サマリー
• 新規: 12件 (前週比 +3)
• Critical以上: 2件

3つの問い
⚠️ 決済モジュールにバグが集中 (5件, 42%)
🔁 認証まわりで先週に続いてNormalバグ発生
🛡️ Critical 2件。仕様の曖昧さが原因の可能性

💬 15分だけ、チームで話してみてください。
```

ポイント:

- **数字で始める**：件数と前週比だけで「増えた/減った」がわかる
- **3つの問いへの回答**：偏り・再発・予防をアイコンで一目で識別
- **最後に行動を促す**：「15分話してください」の一言

## AI分析を足すなら

ルールベースで十分回った後、AIを足すと効果的なポイントがあります。

| 用途 | AIなし（ルールベース） | AIあり |
|---|---|---|
| 偏り検出 | 件数カウント | パターン類似度も加味 |
| 再発判定 | プロジェクト×Severity一致 | タイトル自然言語マッチング |
| 予防示唆 | Severity高→レビュー推奨 | バグ内容から原因工程を推定 |
| メッセージ | テンプレ固定 | 文脈に合わせた自然な表現 |

AIを入れる場合も、**判断はルールベース、表現だけAI**くらいが実用的です。AIの出力を鵜呑みにして通知すると、的外れな示唆が信頼を壊します。

```python
# AI強化の例: バグタイトルから予防可能性を判定
prompt = f"""以下は今週見つかった不具合の一覧です。
この中から「設計・要件の段階で防げた可能性がある」ものを選び、
理由を1行で添えてください。

{bug_titles}
"""
```

## 閾値設計：「鳴ったら本物」にする

ここまでの仕組みで、毎週Slackに通知が届くようになります。でも、毎週通知が来ると**オオカミ少年**になります。

```
毎週通知が来る    → 「またか」 → スルー
閾値を超えた時だけ → 「何かあった」 → 見る
```

通知を2段階に分けます。

### サイレントログ（毎週）

閾値を下回っている週は、Slackには送らずログだけ残します。データは蓄積するが、チームの注意は消費しない。

### アラート通知（閾値超過時のみ）

以下の指標が閾値を超えた時だけ、Slackに通知します。

| 指標 | 計算式 | 閾値 | 意味 |
|---|---|---|---|
| 不具合検出率 | 不具合 ÷ 実行ケース | ≥ 5% | 品質の黄信号 |
| 前週比 | 今週件数 ÷ 先週件数 | ≥ 1.5倍 | 急な悪化 |
| Critical件数 | 週あたりCritical以上 | ≥ 3件 | 即トリアージ |
| 集中度 | 1領域 ÷ 全体 | ≥ 40% | 構造的な問題 |
| 再発 | 同領域で2週連続検出 | 2週連続 | パターン化 |

```python
def should_alert(analysis):
    """閾値を超えたらTrue、下回ったらFalse（ログのみ）"""
    if analysis["defect_rate"] >= 0.05:
        return True, f"不具合検出率 {analysis['defect_rate']:.1%} (閾値5%超過)"
    if analysis["week_over_week"] >= 1.5:
        return True, f"前週比 {analysis['week_over_week']:.1f}倍 (閾値1.5倍超過)"
    if analysis["critical_count"] >= 3:
        return True, f"Critical {analysis['critical_count']}件 (閾値3件超過)"
    if analysis["max_concentration"] >= 0.4:
        return True, f"集中度 {analysis['max_concentration']:.0%} (閾値40%超過)"
    return False, None
```

### 不具合/テストケース比について

「不具合 ÷ テストケース」はもっともシンプルな品質指標ですが、注意点があります。テストケースの粒度がチームで異なると、数字の意味が変わります。1ケース3ステップのチームと30ステップのチームでは、同じ5%でも品質状態は違います。

**自チームの推移を追う**用途でつかうのが正しい使い方です。他チームや業界平均との比較には、不具合密度（Defects/KLOC）や不具合除去効率（DRE）の方が適しています。

:::details 不具合密度（Defects/KLOC）とは
コード1,000行あたりの不具合件数です。チーム横断や業界比較に使える標準的な品質指標です。
:::

:::details DRE（不具合除去効率）とは
リリース前に除去できた不具合の割合（リリース前検出数 / 全不具合数）です。テストプロセスの有効性を測る指標として使われます。
:::

### 閾値の育て方

最初から完璧な閾値は設定できません。

1. **最初の4週間**: 閾値なしで毎週通知を送り、データを貯める
2. **5週目**: 4週間の平均値を出す。平均の1.5〜2倍を閾値にする
3. **以降**: 閾値超過が多すぎたら上げる、少なすぎたら下げる

閾値は固定値ではなく、チームの成熟度に合わせて動かすものです。

## 運用のコツ

### 週1回、同じ曜日の同じ時間

月曜9時なら毎週月曜9時。リズムが崩れると見なくなります。

### 最初は通知だけ。会議にしない

「振り返り会議」にすると、予定が合わない・面倒・形骸化、の三重苦になります。まずはSlack通知だけ。反応がなくてもいい。目に入ることが大事です。

チームが通知に反応し始めたら（「これ前もあったよね」等）、自然と振り返りが生まれます。

### 通知を@メンションにする

「全員向け」の通知は誰にも刺さりません。バグが集中した領域の担当者を名指しで@メンションすることで、「自分ごと」になります。

```
❌ ⚠️ 決済モジュールにバグが集中 (5件)
✅ ⚠️ @tanaka @suzuki 決済モジュールにバグが集中 (5件)
```

### 止めない、でも黙らせる

最大の敵は「今週は特に何もなかったから送らなくていいか」という判断です。閾値設計を入れることで、この問題は解消します。閾値以下ならサイレント、超えたらアラート。**人が「送る/送らない」を判断しない**のがポイントです。

## 最小構成まとめ

```
必要なもの:
  ✅ cron (どのサーバーにもある)
  ✅ テスト管理ツールのAPIトークン (Qase/Jira等)
  ✅ Slack Webhook URL
  ✅ Pythonスクリプト 1本 (100行程度)

不要なもの:
  ❌ 専用サーバー (ローカルPCで十分)
  ❌ データベース (CSVで十分)
  ❌ BI ツール (Slackで十分)
  ❌ AI API (ルールベースで十分)
```

100行のスクリプトと1つのcron設定で、チームの品質改善サイクルが回り始めます。

## おわりに

品質の振り返りが続かないのは、チームの意識が低いからではありません。**仕組みがないから**です。

ダッシュボードは「見に来い」。でも忙しい開発者は見に来ない。だから**届ける**。毎週、同じ時間に、同じフォーマットで、開発チャンネルに。

逃げ道をなくすのではなく、逃げる必要がない状態をつくる。振り返りの結果が自動で届けば、あとはチームが勝手に反応し始めます。

必要なのは、100行のスクリプトと15分の設定だけです。

---

## 関連記事

- [テストしかしていないQAチームが、来週から始められる分析習慣](https://zenn.dev/bitkey_dev/articles/qa-first-analysis-ritual) ── まずは手動で始める第一歩
- [「テスト＝QA」を卒業する日](https://zenn.dev/bitkey_dev/articles/test-equals-qa) ── テスト・QC・QAの区別をつける
- [バグ報告しても「ふーん」で終わる理由](https://zenn.dev/bitkey_dev/articles/trust-building-for-testers) ── 開発者に届けるための信頼構築

---

*仕組みが品質を変える。❤️ いいねで届けてください。*
