# Weekly Sentinel Incident Report (Markdown)

> Microsoft Sentinel のインシデントを週次で集計し、日次トレンド・カテゴリー別分布・送信元IP分析・前週比較・危険度上位5件の詳細解説・対策方針を含む Markdown レポートを自動生成する Security Copilot Agent。KQL のみで完結し、外部スキルセットへの依存なし。

---

## 概要

| 項目 | 内容 |
|---|---|
| プラグイン種別 | Standard Agent（スケジュール / 手動） |
| モデル | gpt-4o |
| トリガー | PowerAutomate（手動 / スケジュール） |
| データソース | Sentinel `SecurityIncident` / `SecurityAlert` |
| 出力形式 | Markdown レポート |
| 外部依存 | なし（KQL のみで完結） |

---

## アーキテクチャ

```
┌────────────────┐     トリガー      ┌────────────────────┐
│  Power Automate │ ────────────────▶ │  Security Copilot  │
│  (スケジュール) │ ◀──────────────── │  Standard Agent    │
└────────────────┘   Markdown レポート └────────────────────┘
                                             │
                                  ┌──────────┼──────────┐
                                  ▼          ▼          ▼
                            KQL Skills   KQL Skills   KQL Skills
                           (Incidents)   (Alerts)    (Details)
                                  │          │          │
                                  ▼          ▼          ▼
                            SecurityIncident  SecurityAlert
                            (Sentinel)        (Sentinel)
```

---

## ワークフロー

```
PowerAutomate トリガー
        ↓
Phase 1: 集計データ収集（今週 + 先週）
    ├─ GetSentinelIncidentsByDay        … 日次インシデント件数 & Severity 内訳
    ├─ GetSentinelIncidentsByCategory   … アラート種別・ProviderName 別集計
    └─ GetSentinelIncidentsBySourceIP   … 送信元 IP 別集計
        ↓
Phase 2: 危険度上位5件の調査
    ├─ GetSentinelTopIncidents          … 重大度・アラート数順で上位5件取得
    └─ GetSentinelIncidentAlerts (×5)   … 各インシデントの関連アラート詳細
        ↓
Phase 3: Markdown レポート生成
    エージェントが全データを統合し、構造化レポートを出力
```

---

## レポート構成

生成されるレポートは以下のセクションで構成されます。

| セクション | 内容 |
|---|---|
| ヘッダー | レポートタイトル・対象期間・生成日時 |
| Section 1 | サマリー（合計件数・High 件数・1日平均・最多カテゴリー／今週 vs 先週） |
| Section 2 | 日次トレンド（今週 vs 先週の曜日別 Severity 内訳テーブル） |
| Section 3 | カテゴリー別分布（アラート種別 × ProviderName の今週 vs 先週比較） |
| Section 4 | 送信元 IP ランキング（件数・代表アラート・NEW 判定） |
| Section 5 | 週次傾向サマリー（自然言語での前週比分析） |
| Section 6 | 危険度上位5件のインシデント詳細（Severity / ステータス / MITRE ATT&CK / 推奨対応） |
| Section 7 | 推奨対策方針（優先度別アクション + 来週の重点監視ポイント） |

---

## 前提条件

### Microsoft Security Copilot
- カスタムプラグインのアップロード権限

### Microsoft Sentinel
- **SecurityIncident** テーブルおよび **SecurityAlert** テーブルにデータが取り込まれていること
- Sentinel ワークスペースへの読み取りアクセス権限

---

## セットアップ

### 1. プラグイン YAML の編集

`WeeklySentinelIncidentReportMd.yaml` 内の各 KQL スキルに含まれる以下のプレースホルダーを、自身の Sentinel ワークスペース情報に置き換えてください。

| プレースホルダー | 説明 | 例 |
|---|---|---|
| `<YOUR_TENANT_ID>` | Azure AD テナント ID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `<YOUR_SUBSCRIPTION_ID>` | Azure サブスクリプション ID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `<YOUR_RESOURCE_GROUP>` | Sentinel ワークスペースのリソースグループ名 | `rg-sentinel` |
| `<YOUR_WORKSPACE_NAME>` | Sentinel ワークスペース名 | `sentinel-workspace` |

### 2. プラグインのアップロード

1. [Microsoft Security Copilot](https://securitycopilot.microsoft.com) にサインイン
2. **Settings** → **Custom plugins** → **Add plugin**
3. `WeeklySentinelIncidentReportMd.yaml` をアップロード
4. プラグインを有効化

### 3. Power Automate との連携

Power Automate フローから Security Copilot エージェントを呼び出すことで、週次スケジュール実行や Markdown レポートのメール配信・Teams 投稿が可能です。

---

## スキル一覧

| スキル名 | 種別 | データソース | 説明 |
|---|---|---|---|
| `OrchestrateWeeklyReportMd` | Agent | — | レポート生成オーケストレーション |
| `GetSentinelIncidentsByDay` | KQL | Sentinel `SecurityIncident` | 過去14日間の日次インシデント集計（Severity 内訳・週別） |
| `GetSentinelIncidentsByCategory` | KQL | Sentinel `SecurityAlert` | アラート種別・ProviderName 別集計（週別） |
| `GetSentinelIncidentsBySourceIP` | KQL | Sentinel `SecurityAlert` | 送信元 IP 別集計（Entities から IP を抽出、週別） |
| `GetSentinelTopIncidents` | KQL | Sentinel `SecurityIncident` | 過去7日間の重大度・アラート数上位5件取得 |
| `GetSentinelIncidentAlerts` | KQL | Sentinel `SecurityIncident` + `SecurityAlert` | 指定インシデント番号の関連アラート詳細取得 |

---

## KQL クエリの注意点

### SecurityIncident の重複排除

`SecurityIncident` テーブルはインシデント更新ごとにレコードが追加されるため、`summarize arg_max(TimeGenerated, *) by IncidentNumber` で最新レコードのみを使用しています。

### IP アドレスの抽出

`SecurityAlert` の `Entities` カラムは JSON 配列です。`mv-expand` で展開し、`Type == "ip"` でフィルタして `Address` フィールドから IP を取得しています。

```kql
SecurityAlert
| mv-expand Entity = todynamic(Entities)
| where Entity.Type == "ip"
| extend SourceIP = tostring(Entity.Address)
```

### インシデントとアラートの結合

`SecurityIncident.AlertIds` にアラート ID の配列が格納されています。`mv-expand` で展開後、`SecurityAlert.SystemAlertId` と `join` して関連アラートを取得します。

---

## Defender 版との違い

| 項目 | Defender 版 | Sentinel 版（本プラグイン） |
|---|---|---|
| KQL Target | `Defender` | `Sentinel` |
| テーブル | `AlertInfo`, `AlertEvidence` | `SecurityIncident`, `SecurityAlert` |
| 外部スキル依存 | M365 (`GetDefenderIncidents`), Fusion (`GetIncident`) | **なし（KQL のみで完結）** |
| RequiredSkillsets | 自身 + M365 + Fusion | 自身のみ |
| Top 5 取得 | 組み込み M365 / Fusion スキル | `GetSentinelTopIncidents` (KQL) |
| アラート詳細 | `GetIncident` (Fusion) | `GetSentinelIncidentAlerts` (KQL, パラメータ付き) |
| Sentinel 設定 | 不要 | TenantId / SubscriptionId / ResourceGroupName / WorkspaceName が必要 |

---

## ファイル構成

```
WeeklySentinelIncidentReportMdAgent/
├── WeeklySentinelIncidentReportMd.yaml        # プラグイン本体（Agent + KQL スキル）
├── WeeklySentinelIncidentReportMd_card.html   # プラグインカード（視覚的な説明）
└── README.md                                   # このファイル
```

---

## 免責事項

- 本プラグインは公知のテーブルスキーマをもとに作成しています。環境によりカラム名が異なる場合があります。
- KQL クエリは本番環境での動作確認を推奨します。
- レポートの内容は収集されたデータに基づきます。データが存在しない場合は「データなし」として報告されます。
- 架空のデータは生成されません。

---

## 参考リンク

- [Microsoft Security Copilot カスタムプラグイン ドキュメント](https://learn.microsoft.com/ja-jp/copilot/security/custom-plugins)
- [Security Copilot Agent の作成](https://learn.microsoft.com/ja-jp/copilot/security/developer/build-agents)
- [SecurityIncident テーブル リファレンス](https://learn.microsoft.com/ja-jp/azure/azure-monitor/reference/tables/securityincident)
- [SecurityAlert テーブル リファレンス](https://learn.microsoft.com/ja-jp/azure/azure-monitor/reference/tables/securityalert)
- [Microsoft Sentinel の概要](https://learn.microsoft.com/ja-jp/azure/sentinel/overview)
