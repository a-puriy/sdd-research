# RPIフロー定着化 — JIRA Initiative / Epic 設計案

---

## 設計の参考フレームワーク

Initiative/Epicの設計にあたり、以下の業界標準フレームワークの観点を取り入れている。

| フレームワーク | 提唱 | 活用した観点 |
|--------------|------|------------|
| [DORA Metrics](https://dora.dev/guides/dora-metrics-four-keys/) | Google DORA Team | デプロイ頻度、リードタイム、変更障害率、MTTR の4指標。開発生産性の定量基盤 |
| [SPACE Framework](https://queue.acm.org/detail.cfm?id=3454124) | GitHub / Microsoft Research (Forsgren, et al.) | Satisfaction, Performance, Activity, Communication, Efficiency の5次元。定量だけでなく開発者体験の定性面もカバー |
| [DevEx Framework](https://queue.acm.org/detail.cfm?id=3595878) | Noda, Storey, Forsgren (2023) | Flow State, Cognitive Load, Feedback Loops の3次元。開発者が「集中して開発できる状態」を構造的に捉える |
| [Block AI-Assisted Development](https://engineering.block.xyz/blog/ai-assisted-development-at-block) | Block (Angie Jones) | AI Champions, Repo Quest, 段階的定着化の実績モデル |
| [ThoughtWorks Technology Radar](https://www.thoughtworks.com/en-us/radar/techniques/spec-driven-development) | ThoughtWorks | SDDの「Assess」段階の位置づけ。慎重な段階的評価の推奨 |
| [Discovery-Driven Planning](https://hbr.org/1995/07/discovery-driven-planning) | Rita McGrath & Ian MacMillan (Columbia / Wharton) | 不確実性の高い取り組みでは「達成目標」ではなく「検証すべき仮説」と「学習マイルストン」で計画を組む |

---

## Initiative設計の前提：なぜ「仮説検証型」で組むか

CodingAgent活用によるRPIフローは、[ThoughtWorksがAssess段階と位置づけている](https://www.thoughtworks.com/en-us/radar/techniques/spec-driven-development)通り、業界全体がまだ「本当に効くのか検証中」のフェーズにある。この段階でInitiativeに「レビュー時間30%短縮」のような効果達成を置くと、検証前にコミットメントが発生し、以下のリスクを招く：

- 効果が出なかった場合に「失敗」と見なされ、学習の機会が失われる
- 目標達成のためにプロセスが硬直化し、本来の目的（チームの改善）を見失う
- PoC結果に関わらず「目標未達」を避けるために撤退判断が遅れる

[McGrath & MacMillanのDiscovery-Driven Planning](https://hbr.org/1995/07/discovery-driven-planning)（HBR, 1995）は、不確実性の高い取り組みにおいて「学習すべきこと」を計画の中心に据えるアプローチである。本PJではこの考え方を採用し、Initiativeを**「検証すべき仮説」**として定義する。

仮説が検証された場合（＝効果が確認された場合）に、初めて具体的な数値目標を持つInitiativeに昇格させる。これにより、PoC段階で過大なコミットメントを避けつつ、学習を構造的に蓄積できる。

---

## Initiative 一覧

Initiativeは「CodingAgent＋RPIフローに関する検証すべき仮説」を表す。PoC段階では仮説の検証自体が価値であり、検証結果に基づいて次のアクション（拡大/ピボット/撤退）を判断する。

### Initiative 1: レビュー負荷の構造的な軽減は可能か

> **仮説**: ワークフローレベル分類（Lightweight/Standard/Full）と機械的品質チェックの自動化により、オンサイトレビュアーのレビュー対象をPRの本質（設計意図・ビジネスロジック）に絞り込むことができる。

**背景**: SpecKit試験運用で顕在化した「レビュー負荷の二重化」が現チームの最大のボトルネックである。ただし、CodingAgentによるレビュー補助が本当にレビュー負荷を減らすのか、それとも「AIの出力を確認する」という新たな負荷を生むのかは未検証。

**検証すべき問い**:
1. ワークフローレベル分類により、Lightweightに分類されるチケットは全体の何%か（多すぎても少なすぎても分類基準の問題）
2. 機械的チェックの自動化で、レビュアーのレビューコメントの内訳（規約指摘 vs 設計指摘）はどう変化するか
3. エージェントによるPRサマリー自動生成は、レビュアーに「有用」と評価されるか

**検証の計測指標**:
- DORA: リードタイム（PR→マージ）のベースラインとの差分
- SPACE: Efficiency（レビュアーアンケート「焦点の明確さ」「承認判断のしやすさ」）
- DevEx: Feedback Loops（レビュー待ち時間の変化）

**仮説が棄却される条件**: エージェントのPRサマリーがレビュアーに「読む手間が増えるだけ」と評価される、またはレビュー所要時間がベースラインから改善しない場合。この場合、エージェントによるレビュー補助ではなく、CI自動化＋ワークフロー分類のみにスコープを縮小する。

---

### Initiative 2: Plan段階の合意形成は手戻りを減らすか

> **仮説**: Implement前にResearch→Planフェーズを挟み、オンサイト-オフショア間でPlan段階の双方向Q&Aを行うことで、PR差し戻し率（Request Changes）を有意に削減できる。

**背景**: タイムゾーン差のある体制では1回の手戻りが1営業日のロスに直結する。しかし、RPIのPlanフェーズ自体がオーバーヘッドとなり、「Plan作成のコスト > 手戻り削減の効果」となる可能性もある。

**検証すべき問い**:
1. Plan実施チケットとPlan未実施チケットで、差し戻し率に統計的に有意な差があるか
2. Plan作成にかかる工数（時間）は、手戻り1回あたりのコスト（時間）と比較して妥当か
3. オフショアからのPlan段階の質問は実際に発生するか（形骸的な承認になっていないか）

**検証の計測指標**:
- DORA: 変更障害率（差し戻し率をPlan有/無で比較）
- SPACE: Communication（Plan段階の質問数/チケット）
- DevEx: Cognitive Load（開発者アンケート「Plan文書の理解しやすさ」）

**仮説が棄却される条件**: Plan有/無で差し戻し率に有意差がない場合、またはPlan作成工数が手戻りコストを上回る場合。この場合、Planの粒度をさらに簡素化するか、Fullワークフローのみに適用範囲を限定する。

---

### Initiative 3: CodingAgentはチームの開発能力を拡張するか

> **仮説**: AI Championsを中心としたエージェント活用の段階的導入により、チームの開発キャパシティ（ストーリーポイント消化速度）が向上し、かつ本番障害率が悪化しない。

**背景**: [Block社はAI Championsプログラム開始3ヶ月で、AI著作コード量69%増・自動PR作成21倍増を達成した](https://engineering.block.xyz/blog/ai-assisted-development-at-block)。ただし、Block社と本PJではチーム規模・プロダクト特性・オンサイト-オフショア構成が異なるため、同様の効果が得られるかは未検証。

**検証すべき問い**:
1. Champions制度は本PJの体制（15名オンサイト＋15名オフショア）で機能するか。Championsの30%時間投資は他業務に影響しないか
2. オフショア開発者はエージェントを「自発的に」活用するか、それとも指示がないと使わないか
3. エージェント生成コードの品質は、人間が書いたコードと比較して本番障害率に差があるか

**検証の計測指標**:
- SPACE: Satisfaction（開発者アンケート「エージェントは役立っているか」）、Activity（co-authored-byコミット比率、セッション数/人・週）
- DORA: 変更障害率（エージェント関与PR vs 非関与PR）
- DevEx: Flow State（「定型作業が減り、創造的作業に集中できているか」）

**仮説が棄却される条件**: パイロットチームでエージェント利用率が30%未満に留まる場合、またはエージェント関与PRの本番障害率が有意に高い場合。この場合、エージェント活用の対象フェーズを限定（例: Researchのみ）するか、チームのスキル習熟に追加投資する。

---

### Initiative間の関係と段階的な意思決定

```
Initiative 3（エージェントは能力を拡張するか）
    │
    ├── 検証OK → Initiative 1, 2 の本格推進
    │
    └── 検証NG → エージェントに依存しない改善策（CI自動化・ワークフロー分類のみ）にピボット
```

Initiative 3がすべての前提条件となる。「エージェントがチームに受け入れられ、品質を損なわない」ことが確認できなければ、Initiative 1・2のエージェント活用部分は成立しない。ただし、Initiative 1のワークフローレベル分類やCI自動化、Initiative 2のPlanフェーズ自体は、エージェントなしでも独立した価値がある。

**PoC → Scale の判断ポイント**: Initiative 1-3の仮説がすべて棄却されなかった場合に、初めて具体的な数値目標（例: レビュー時間30%短縮、差し戻し率50%削減）を設定し、全チーム展開のInitiativeに昇格させる

---

## Epic 一覧

EpicはPOが意識すべきマイルストンとして設計している。各Initiativeの下にフェーズ順で配置し、前のEpicの成果が次のEpicの前提条件となる依存構造を持たせている。

### Initiative 1: レビュー負荷の構造的な軽減は可能か

| Epic | 名称 | 完了条件 | 計測指標 | 性質 |
|------|------|---------|---------|------|
| **E1.1** | レビュー負荷のベースライン計測 | 2-4スプリント分のレビュー所要時間・件数/人のデータ収集が完了 | DORA: リードタイム（PR→マージ）のベースライン値が確定 | 計測 |
| **E1.2** | ワークフローレベル分類の導入 | Lightweight/Standard/Fullの判定基準が策定され、スプリントプランニングで適用されている | 分類ごとのPR件数比率が可視化されている | 施策（エージェント非依存） |
| **E1.3** | 機械的品質チェックの自動化 | CI/pre-commitでlint・型チェック・テスト通過が自動検証され、人間のレビュー対象から除外されている | CIパイプラインの自動チェックカバー率 | 施策（エージェント非依存） |
| **E1.4** | エージェントによるレビュー補助の試行 | PRサマリー自動生成・変更影響分析をパイロットチームで試行し、レビュアーからの評価を収集 | レビュアーアンケート「有用だったか」の回答分布 | **仮説検証** |
| **E1.5** | 仮説検証の判定とNext Action決定 | E1.1-E1.4の結果を総合し、レビュー負荷がベースラインから改善傾向にあるかを判定。改善あり→数値目標設定へ、改善なし→スコープ縮小の判断 | 判定結果＋Next Actionの文書化 | **判断** |

---

### Initiative 2: Plan段階の合意形成は手戻りを減らすか

| Epic | 名称 | 完了条件 | 計測指標 | 性質 |
|------|------|---------|---------|------|
| **E2.1** | 手戻り率のベースライン計測 | PR差し戻し率（Request Changes / 全PR）の2-4スプリント分データが確定 | 差し戻し率のベースライン値 | 計測 |
| **E2.2** | Researchフェーズの定着 | Standardワークフロー以上のチケットで、Implement前にResearch成果物が作成されている | Research実施率（Standard以上のチケット対象） | 施策 |
| **E2.3** | Planフェーズの承認フロー確立 | Plan→Implement間の承認ゲートが運用され、オフショアからのPlan質問・確認のやり取りが発生している | Plan段階での質問数/チケット（ゼロは形骸化の兆候） | 施策 |
| **E2.4** | Plan有無による差し戻し率の比較検証 | Plan実施チケットと未実施チケットの差し戻し率を比較し、差分を定量化。Plan作成工数と手戻りコストのROI試算を完了 | 差し戻し率の差分＋ROI試算 | **仮説検証** |
| **E2.5** | 仮説検証の判定とNext Action決定 | E2.4の結果に基づき、Plan有効性を判定。有効→適用範囲拡大＋数値目標設定、限定的→Fullのみに限定、無効→Plan粒度の再設計 | 判定結果＋Next Actionの文書化 | **判断** |

---

### Initiative 3: CodingAgentはチームの開発能力を拡張するか

| Epic | 名称 | 完了条件 | 計測指標 | 性質 |
|------|------|---------|---------|------|
| **E3.1** | AI Champions選出とリポジトリ整備 | 3-5名のChampionsが選出され、CLAUDE.md + steering/（3ファイル）が整備されている | steering/のファイル存在＋Championsの活動開始 | 基盤構築 |
| **E3.2** | RPIコマンドの初期版構築と検証 | `/research`, `/plan`, `/implement` の初期プロンプトが作成され、`/rpi-validate` でCONDITIONAL以上を達成 | rpi-validate の総合判定 | 基盤構築 |
| **E3.3** | パイロットチームでのRPIフロー試行 | Champions + オフショア3-5名でStandardワークフローを1-2スプリント試行し、定量・定性評価を収集 | パイロット評価レポートの完成 | **仮説検証** |
| **E3.4** | パイロット結果の判定とScale判断 | E3.3の結果に基づき、全チーム展開の可否を判定。利用率・品質・開発者体験の3軸で判断 | 判定結果＋展開計画 or ピボット計画 | **判断** |
| **E3.5** | 全チーム展開 or スコープ限定展開 | E3.4の判断に基づき実行。全面展開 or 特定フェーズ限定 or 撤退のいずれか | 展開範囲の実績＋利用率 | 実行 |

---

## Epic間の依存関係

```
E1.1 ─→ E1.2 ─→ E1.3 ─→ E1.4 ─→ E1.5
              ↘
E2.1 ─→ E2.2 ─→ E2.3 ─→ E2.4 ─→ E2.5
              ↗
E3.1 ─→ E3.2 ─→ E3.3 ─→ E3.4 ─→ E3.5
```

- **E3.1（Champions + リポジトリ整備）** は全Initiativeの起点。最優先で着手
- **E1.1 / E2.1（ベースライン計測）** はE3.1と並行して実施可能
- **E3.2（RPIコマンド構築）** が完了しないとE2.2 / E2.3に進めない
- **E1.3（CI自動化）** はRPIフローと独立して進行可能（先行着手推奨）

---

## タイムライン目安

| 期間 | 対象Epic | 備考 |
|------|---------|------|
| **Month 1** | E3.1, E1.1, E2.1 | 基盤整備＋ベースライン計測。並行実施可能 |
| **Month 2** | E3.2, E1.2, E1.3 | RPIコマンド構築＋ワークフロー分類＋CI自動化 |
| **Month 3-4** | E3.3, E2.2, E2.3, E1.4 | パイロット試行。全Initiativeが動き始める |
| **Month 5-6** | E2.4, E3.4, E1.5, E2.5 | 効果検証＋全チーム展開 |
| **Month 7+** | E3.5 | 自律運用体制の確立。継続改善フェーズ |

---

## 補足: フレームワーク観点の適用ガイド

### DORA Metricsの活用法

[DORA](https://dora.dev/guides/dora-metrics-four-keys/)の4指標のうち、本PJで特に注視すべきは以下の2つ:

- **リードタイム（Lead Time for Changes）**: チケット起票からマージまでの時間。Initiative 1の主指標。GitHub APIで自動計測可能
- **変更障害率（Change Failure Rate）**: 差し戻し率＋本番障害率。Initiative 2の主指標

デプロイ頻度・MTTRは、RPIフロー導入の直接的な改善対象ではないが、悪化していないことの確認として月次でモニタリングする。

### SPACE Frameworkの活用法

[SPACE](https://queue.acm.org/detail.cfm?id=3454124)は「生産性を単一指標で測るな」という原則に基づく。本PJでは:

- **S（Satisfaction）**: スプリント毎のアンケート。「RPIフローは開発に役立っているか」
- **P（Performance）**: DORA指標で代替
- **A（Activity）**: エージェント利用セッション数、co-authored-byコミット比率
- **C（Communication）**: Plan段階の質問数/チケット（ゼロは形骸化、過多は仕様不明確）
- **E（Efficiency）**: レビュー所要時間、ポイントあたり実工数

### DevEx Frameworkの活用法

[DevEx](https://queue.acm.org/detail.cfm?id=3595878)の3次元は、定性評価の設計に活用:

- **Flow State**: 「RPIフローによって集中が途切れる場面はあるか」（不要なプロセスの検出）
- **Cognitive Load**: 「Planの記述は理解しやすいか」「RPIコマンドの使い分けに迷うか」
- **Feedback Loops**: 「レビュー待ち時間は許容範囲か」「エージェントの応答は有用か」
