# ハーネスエンジニアリング定着ロードマップ

**作成日**: 2026-03-28
**目的**: RPIフローとハーネスエンジニアリングの段階的導入ロードマップ。集権型（AI推進チーム主導）から分権型（各チーム自律運用）への移行を描く。

---

## 設計の前提

### ハーネスエンジニアリングとは

[逆瀬川（2026）の定義](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/)によれば、ハーネスエンジニアリングとは「人間によるAGENTS.mdの継続的改善と、エージェントが自分の作業の正誤を自己検証するためのツール群」であり、より広くは「Coding Agentが自律的に稼働し出力を安定させるためのシステム設計」を指す。

本ロードマップでは、ハーネスを以下の4層で構成されるものとして扱う。

| 層 | 構成要素 | 腐敗リスク | 管理方式 |
|----|---------|-----------|---------|
| **決定論的ガードレール** | Linter・型チェック・テスト・CI・Hooks | 極低（実行すれば嘘をつけない） | 設定ファイルで管理 |
| **永続化コンテキスト** | CLAUDE.md・steering/・ADR | 中（記述文書は時間とともに腐敗する） | ポインタ型設計＋四半期レビュー |
| **ワークフロー定義** | RPIコマンド（/research, /plan, /implement） | 中（モデル進化で陳腐化する可能性） | Bitter Lesson対策＋定期簡素化 |
| **ドメインスキル** | 業務ロジック・外部API連携パターン等 | 高（ビジネス変化に直接連動） | 利用頻度ベースの棚卸し |

### なぜ段階的に分権化するのか

[逆瀬川が「ハーネスなしでエージェントの数を増やすと複利的な認知的負債が生まれる」と警告している](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/)通り、ハーネス品質が確保されないまま全チームに自律運用を任せると、チームごとにハーネスの質がばらつき、エージェント出力の品質にも格差が生じる。

一方、AI推進チームが永続的に全チームのハーネスを管理し続けることはスケールしない。[Stripeの教訓「エージェント専用インフラを構築するな。優れた開発者インフラを構築せよ」](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/)が示すように、ハーネスは特別なものではなく「良い開発インフラ」の一部として各チームが自然に保守できる状態を目指すべきである。

したがって、**集権→分権の移行には、ハーネスの品質を機械的に検証する仕組み**が前提条件となる。人間のレビューだけでは属人化するため、検証スキルやCIチェックで「ハーネスが健全か」を自動的に確認できる状態を作ってから分権化する。

---

## Phase 1: 集権型 — ワークフローレベル分類の全チーム横断導入

### 目的

全チームがスプリントプランニングでLite/Middle/Highのワークフローレベルを判断し、レビューステップを適切にスケーリングできる状態を作る。ハーネスエンジニアリング（CLAUDE.md・steering・コマンド・スキル・Hooks）はAI推進チームが集中管理する。

### Phase 1a: 基盤構築（Month 1-2）

**AI推進チームの作業**:

1. **最小実行可能ハーネス（MVH）の構築**

   [逆瀬川のMVHロードマップ](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/)に準拠し、以下を整備する。

   ```
   CLAUDE.md                    # 50行以下。ポインタ型設計
   .kiro/steering/
     product.md                 # プロダクト概要（〜50行）
     tech.md                    # 技術スタック・規約（〜100行）
     structure.md               # ディレクトリ構成パターン（〜80行）
   .claude/settings.json        # Hooks設定
   ```

   CLAUDE.mdの設計原則（[逆瀬川](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/)より）:
   - **50行以下を目指す**。IFScaleの研究で150-200指示の時点でprimacy biasが顕著
   - **ポインタ型**: テストコマンド・ADR場所・禁止事項をリスト化。詳細は参照先に持たせる
   - **指すファイルパスが存在しなければ404相当のエラー**で腐敗を機械的に検出

2. **決定論的ガードレールの整備**

   | ツール | 役割 | 導入方法 |
   |-------|------|---------|
   | Linter（Oxlint/Ruff等） | コーディング規約の自動矯正 | pre-commit + PostToolUse Hook |
   | 型チェッカー | 型安全性の保証 | CI + PostToolUse Hook |
   | テストスイート | 仕様の実行可能な表現 | CI + Stop Hook |
   | カスタムLinter | PJ固有のアーキテクチャ境界 | 段階的に追加 |

   PostToolUse Hooksパターン（[逆瀬川](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/)が「最強」と評価）:
   > ファイル編集 → 自動的にLinter実行 → エラーをadditionalContextとしてエージェントに注入 → 自己修正を駆動

   これにより、**エージェントの出力品質を決定論的に底上げ**する。プロンプトに「Lintを守れ」と書く必要がない。

3. **RPIコマンドの初期版構築**

   `/rpi-validate` でCONDITIONAL以上を達成するプロンプトを作成する（評価基準: `output/prompt-evaluation-guidelines.md`）。

4. **ワークフローレベル判定基準の策定**

   スプリントプランニングで使用する判定チェックリスト:

   | 判定項目 | Yes → レベルを上げる |
   |---------|-------------------|
   | 変更対象ファイルが3ファイル超か | Lite → Middle |
   | 既存のpublic APIシグネチャを変更するか | Lite → Middle |
   | DBスキーマ変更を伴うか | Middle → High |
   | 他チーム/サービスとのインターフェースに影響するか | Middle → High |
   | 新しい外部依存（ライブラリ/API）を追加するか | Middle → High |

   各レベルに対応するレビューステップ:

   | レベル | Research | Plan | Plan承認 | Implement | Code Review |
   |--------|----------|------|---------|-----------|-------------|
   | **Lite** | — | — | — | 開発者＋エージェント | PRレビュー（簡易） |
   | **Middle** | エージェント補助 | 1ページ以内の方針書 | 非同期承認 | TDDサイクル | PRレビュー（標準） |
   | **High** | Gap分析含む調査 | 設計書＋タスク分解 | 同期レビュー会 | TDDサイクル＋段階的PR | PRレビュー＋設計レビュー |

### Phase 1b: パイロット展開（Month 2-3）

**対象**: AI Champions（3-5名）＋ オフショアパイロットチーム（3-5名）

1. パイロットチームでMiddleワークフローを1-2スプリント試行
2. スプリントプランニングでの判定チェックリスト運用を確認
3. ベースラインデータとの比較（レビュー所要時間、差し戻し率、リードタイム）

### Phase 1c: 全チーム横断導入（Month 3-5）

**前提条件**: パイロット結果で仮説が棄却されていないこと（`output/jira-initiative-epic-proposal.md` の判定基準参照）

1. **ワークフローレベル分類をスプリントプランニングの標準プラクティスに**
   - PO/SMが判定チェックリストでチケットを分類
   - 判断に迷う場合は上位レベルを適用する原則
   - レトロスペクティブで判定基準自体を見直し

2. **ハーネスの運用はAI推進チームが集中管理**
   - CLAUDE.md / steering/ の更新
   - RPIコマンドの改善（パイロットフィードバック反映）
   - Hooks設定の調整
   - カスタムLinterの追加（エージェント固有のアンチパターン対策）

3. **各チームの役割はハーネスの「利用」に限定**
   - ワークフローレベルの判定と適用
   - RPIコマンドの実行
   - フィードバック（「このコマンドが使いにくい」「この場面でLinterが邪魔」等）の報告

### Phase 1の完了条件

- [ ] 全チームがスプリントプランニングでワークフローレベルを判定している
- [ ] Middle/Highチケットで RPIフローが実行されている
- [ ] AI推進チームがハーネスを一元管理し、改善サイクルが回っている
- [ ] ベースライン比でレビュー所要時間に改善傾向（または悪化していない）

---

## Phase 2: 分権型 — 各チーム自律運用への移行

### 目的

ハーネスエンジニアリングの保守・改善を各チームが自律的に行える状態に移行する。AI推進チームは「ガバナンスと基盤提供」に特化し、日常的なハーネス管理から手を引く。

### Phase 2を開始するための前提条件

1. **Phase 1で仮説が検証されていること** — RPIフローが実際にレビュー負荷軽減・手戻り削減に寄与していることが確認済み
2. **ハーネス品質の自動検証が可能であること** — 後述する検証スキル・CIチェックが整備済み
3. **各チームにハーネスの基礎知識を持つメンバが存在すること** — AI Champions経験者またはトレーニング修了者

### Phase 2a: ハーネス検証の自動化（Month 5-7）

ハーネスの品質を「人が見なくても分かる」状態にするための仕組みを整備する。

#### 1. ハーネス衛生チェックスキル（`/harness-health`）

AI推進チームが開発する、ハーネスの健全性を自動検証するスキル。

**チェック観点**:

| # | チェック項目 | 判定方法 | 根拠 |
|---|-----------|---------|------|
| H-1 | CLAUDE.mdの行数 | 50行以下: PASS, 51-100行: WARN, 101行以上: FAIL | [逆瀬川: IFScale研究のprimacy bias](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/) |
| H-2 | CLAUDE.mdのポインタ完全性 | 参照先ファイルパスの存在チェック | [逆瀬川: 「404で腐敗を機械的に検出」](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/) |
| H-3 | steering/の記述文書チェック | 「現在状態を説明する記述」の検出（パターン記述であるべき） | [逆瀬川: リポジトリ衛生原則](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/) |
| H-4 | Hooks設定の存在 | PostToolUse Hookが設定されているか | [逆瀬川: 「ほぼ毎回」と「例外なく毎回」の差](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/) |
| H-5 | ADRの鮮度 | 直近90日以内に更新されたADRが存在するか | 意思決定記録の継続性 |
| H-6 | テストカバレッジ閾値 | エージェント生成コード領域のカバレッジが基準以上か | [逆瀬川: 「人間開発以上の高いテストカバレッジが必須」](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/) |
| H-7 | RPIプロンプト妥当性 | `/rpi-validate` のCONDITIONAL以上 | `output/prompt-evaluation-guidelines.md` |

**出力**: PASS / WARN / FAIL の総合判定 + 項目別詳細 + 改善アクション

#### 2. 永続化コンテキストレビュースキル（`/context-review`）

steering/やCLAUDE.mdの内容品質を評価するスキル。

**チェック観点**:

| # | チェック項目 | 意図 |
|---|-----------|------|
| C-1 | パターン vs 列挙の比率 | 列挙（ファイルパス羅列等）が多すぎないか |
| C-2 | 禁止事項のADR/Linterルール紐づけ | 「なぜ禁止か」が追跡可能か |
| C-3 | 実コードとの整合性 | steering/の記述が現在のコードベースと矛盾していないか |
| C-4 | 重複・冗長の検出 | CLAUDE.mdとsteering/間の重複記述 |
| C-5 | 更新日からの経過日数 | 長期未更新ファイルの検出（腐敗リスク） |

#### 3. CI統合

ハーネスファイル（CLAUDE.md, steering/, .claude/settings.json）の変更PRに対して、上記チェックをCIで自動実行する。ハーネス変更もコード変更と同様に、PRベースでレビュー＋自動検証を通す。

### Phase 2b: チーム内Champions育成（Month 6-8）

1. **AI推進チームからの知識移転**
   - ハーネスエンジニアリングの原則（[逆瀬川の7原則](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/)ベース）
   - CLAUDE.md / steering/ の書き方と設計思想
   - Hooks設定の変更方法
   - `/harness-health` と `/context-review` の読み方と対応方法

2. **各チームに1-2名のハーネスオーナーを設置**
   - 初期はAI Champions経験者が担う
   - 四半期ローテーションで属人化を防止
   - 30%ではなく10-15%程度の時間配分（Phase 1で仕組みが整っているため工数が減る）

3. **段階的な権限移譲**

   | 移譲ステップ | AI推進チームの役割 | チームハーネスオーナーの役割 |
   |------------|-----------------|------------------------|
   | Step 1 | ハーネス変更の実施＋レビュー | フィードバック提供 |
   | Step 2 | ハーネス変更のレビューのみ | 変更の起案＋実施 |
   | Step 3 | ガバナンス（基準策定・監査） | 自律的な変更管理 |

### Phase 2c: 自律運用体制の確立（Month 8-12）

1. **各チームが自律的にハーネスを管理**
   - steering/の更新: ハーネスオーナーが実施、チーム内PRレビュー
   - RPIコマンドのカスタマイズ: チーム固有のドメインスキル追加
   - Hooks設定の調整: PJ特性に応じた最適化

2. **AI推進チームはガバナンスに特化**
   - 四半期のクロスチーム監査（`/harness-health` の全チーム実行結果を横断分析）
   - ハーネス検証スキル自体のメンテナンス
   - 新しいベストプラクティスの発見と標準化
   - モデル更新時のプロンプト検証と全チーム向けガイダンス

3. **ハーネスのナレッジベース化**
   - チーム間でのハーネス改善事例の共有（月次）
   - 「このカスタムLinterが効いた」「このsteering記述でエージェントの精度が上がった」等の実績ベースの知見蓄積
   - [逆瀬川の「ハーネス効果の複利性」](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/): Linterルール1つ追加すれば以降すべてのセッションでそのミスが防がれる

### Phase 2の完了条件

- [ ] 各チームにハーネスオーナーが設置され、自律的にハーネス変更を管理している
- [ ] `/harness-health` がCIに統合され、ハーネス変更PRで自動検証されている
- [ ] AI推進チームはガバナンス＋基盤提供に特化し、日常的なハーネス管理から手を引いている
- [ ] クロスチーム監査で、チーム間のハーネス品質格差がWARN以下に収まっている

---

## タイムライン全体像

```
Month 1-2    [Phase 1a: 基盤構築]
             AI推進: MVH構築、ガードレール整備、RPIコマンド作成
             全チーム: ベースライン計測

Month 2-3    [Phase 1b: パイロット]
             パイロットチーム: Middle WFでRPIフロー試行

Month 3-5    [Phase 1c: 全チーム横断導入]
             全チーム: WFレベル分類をプランニングに組込み
             AI推進: ハーネス集中管理＋改善サイクル
             ─────────────────────────────────────────
Month 5-7    [Phase 2a: 検証自動化]
             AI推進: /harness-health, /context-review 開発
             AI推進: CI統合

Month 6-8    [Phase 2b: Champions育成]
             AI推進→チーム: 知識移転、ハーネスオーナー設置
             段階的な権限移譲（Step 1→2→3）

Month 8-12   [Phase 2c: 自律運用]
             各チーム: 自律的ハーネス管理
             AI推進: ガバナンス特化、クロスチーム監査
```

---

## リスクと対策

| リスク | Phase | 影響度 | 対策 |
|-------|-------|--------|------|
| AI推進チームがボトルネックになる（Phase 1で改善要望が集中） | 1c | 高 | 改善要望のトリアージ基準を明確化。「決定論的ガードレールで解決できるものを優先」「プロンプト調整は週次バッチ」 |
| 分権化後にハーネス品質がチーム間でばらつく | 2c | 高 | `/harness-health` CI統合＋四半期クロスチーム監査。FAIL判定時はAI推進チームが介入 |
| ハーネスオーナーが「余計な仕事」と感じ定着しない | 2b | 中 | ハーネス改善の効果（複利性）を定量で示す。ハーネスオーナーの工数を10-15%に抑制（自動検証で省力化） |
| モデル更新でハーネス全体の再設計が必要になる | 全Phase | 中 | [Bitter Lesson対策](https://www.thoughtworks.com/en-us/radar/techniques/spec-driven-development): プロンプトの簡素化を常に意識。決定論的ガードレール（テスト・Lint）はモデル非依存のため残存価値が高い |
| [逆瀬川の警告](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/)「この分野の寿命は数カ月〜最大1年」 | 全Phase | 低-中 | ハーネスの投資は「良い開発インフラ」と重なる部分が大きい（テスト充実・CI強化・ADR）。エージェント不要になっても資産として残る |

---

## 参考文献

| 出典 | 活用した観点 |
|------|------------|
| [ハーネスエンジニアリング・ベストプラクティス 2026](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/) — 逆瀬川 | MVHロードマップ、7原則、CLAUDE.md設計思想、PostToolUseパターン、複利性、腐敗防止 |
| [Coding Agent時代の開発ワークフロー](https://nyosegawa.com/posts/coding-agent-workflow-2026/) — 逆瀬川 | Context Rot対策、機械的品質矯正 |
| [AI-Assisted Development at Block](https://engineering.block.xyz/blog/ai-assisted-development-at-block) — Angie Jones | AI Champions、段階的定着化 |
| [SDD Technology Radar](https://www.thoughtworks.com/en-us/radar/techniques/spec-driven-development) — ThoughtWorks | Assess段階、Bitter Lesson警告 |
| [Discovery-Driven Planning](https://hbr.org/1995/07/discovery-driven-planning) — McGrath & MacMillan | 仮説検証型の計画設計 |
| [Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) — Anthropic | Progressive Disclosure |
