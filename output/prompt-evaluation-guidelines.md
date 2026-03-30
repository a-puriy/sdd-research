# RPIコマンドプロンプト妥当性評価指針

**目的**: PJ固有のRPIコマンドプロンプトを構築した後、その品質を評価するための観点と基準を提供する。
**評価方法**: 各観点について「必須/推奨/不要」の3段階で判定し、不要な観点を含めないことで過剰なプロセスを防止する。

---

## 評価の前提：何を評価しないか

以下は評価対象外とする。これらを含めると評価が肥大化し、本質を見失う。

- **プロンプトの文章品質**（日本語の自然さ、Markdownの整形等）：品質に影響しない
- **テンプレートの網羅性**（全セクションが埋まっているか）：SpecKitの教訓として、網羅性の追求がレビュー負荷を生む
- **他フレームワークとの機能的同等性**（SpecKitにある機能がRPIにもあるか）：PJ最適化が目的であり、機能の移植が目的ではない

---

## 評価軸1: コンテキスト設計（エージェントに何を渡すか）

3フレームワークすべてが「生成前にコンテキストを完全にロードする」戦略を採用している。これはエージェントの幻覚（hallucination）を防ぐための基本設計であり、最も重要な評価軸である。

### 1.1 必須：コンテキストロードの明示的指示

**根拠**: Kiro・SpecKit・OpenSpecの全コマンドが、冒頭で「何を読み込むか」を明示的にリストしている。

```
# Kiro spec-requirements.md の例
1. Load Context:
   - Read `.kiro/specs/$1/spec.json` for language and metadata
   - Read `.kiro/specs/$1/requirements.md` for project description
   - Load ALL steering context: Read entire `.kiro/steering/` directory
```

**評価チェック**:
- [ ] プロンプトの冒頭に「何を読み込むか」の明示的なリストがあるか
- [ ] 読み込み対象のファイルパスが具体的か（「関連ファイルを読む」のような曖昧な指示ではないか）
- [ ] 前フェーズの成果物が読み込み対象に含まれているか（Planなら Researchの成果物、Implementなら Planの成果物）

**不要と判断した観点**: 「全steering/を常時ロードすべきか」。Kiroは全量ロードを採用しているが、これはContext Rot（コンテキスト肥大による品質劣化）のリスクがある。RPIフローではフェーズに応じた選択的ロードの方が適切であり、全量ロードを必須とはしない。

### 1.2 必須：コンテキストの階層設計（Progressive Disclosure）

**根拠**: [Anthropicのエージェントスキル設計](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)では、3層のProgressive Disclosureにより不使用スキルのトークン消費を98%削減している。[逆瀬川の分析](https://nyosegawa.com/posts/coding-agent-workflow-2026/)でもコンテキスト利用率40-60%での予防的圧縮が推奨されている。

**評価チェック**:
- [ ] 常時ロード（CLAUDE.md等）とフェーズ固有ロード（steering等）の区別があるか
- [ ] フェーズ固有の情報が不要なフェーズで読み込まれていないか
- [ ] 補助リソース（詳細規約等）がオンデマンド参照として設計されているか

**反論への応答**: 「全量ロードの方がシンプルで確実ではないか」。確かにシンプルだが、運用中プロダクトの中規模コードベースではsteering + 仕様 + コード参照でコンテキストが容易に肥大化する。[Böckelerの報告](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html)でも、拡張コンテキストウィンドウでエージェントが仕様を無視する事象が確認されている。階層設計は「シンプルさ」よりも「信頼性」を優先する判断である。

---

## 評価軸2: フェーズゲート設計（人間の判断をどこに入れるか）

### 2.1 必須：承認ゲートの存在と強制メカニズム

**根拠**: 3フレームワークすべてが、フェーズ間に承認ゲートを設け、未承認時にハードストップする設計を採用している。

```
# Kiro spec-design.md の例
Validate requirements approval:
- If `-y` flag provided: Auto-approve requirements in spec.json
- Otherwise: Verify approval status (stop if unapproved, see Safety & Fallback)
```

**評価チェック**:
- [ ] Research → Plan、Plan → Implementのフェーズ間に承認チェックがあるか
- [ ] 未承認時の動作が「停止＋ユーザーへの通知」であるか（サイレントスキップではないか）
- [ ] 承認状態がメタデータに記録されるか（再開可能か）

### 2.2 推奨：Fast-Track（`-y`）オプションの提供

**根拠**: Kiro・SpecKitは `-y` フラグで承認を省略可能にしている。これはプロセスの柔軟性を担保する。

**評価チェック**:
- [ ] Lightweightワークフローでゲートを省略できるか
- [ ] 省略時にも最低限の記録（「承認省略」のログ等）が残るか

**是々非々の判断**: Fast-Trackは必須ではなく推奨とする。理由は、オフショア開発においてゲート省略の乱用リスクがオンサイト-オフショアの品質統制にとって高いため。ただし、Lightweightワークフローでフルゲートを強制すると形骸化するため、ワークフローレベルに応じた適用が望ましい。

### 2.3 不要と判断：全フェーズへのゲート設置

**根拠**: SpecKitは7段階のフェーズにそれぞれゲートを設け、これがApproval Fatigueの原因となった。[Böckelerも指摘する通り](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html)、SDDツールの冗長なプロセスはレビュー疲弊を招く。

RPIフローにおけるゲートは **Plan → Implement間の1箇所**（Standardワークフロー）または **Research → Plan間 + Plan → Implement間の2箇所**（Fullワークフロー）に限定すべきである。Lightweightではゲート不要。

---

## 評価軸3: 出力仕様の制御（エージェントに何を生成させるか）

### 3.1 必須：出力フォーマットの明示的定義

**根拠**: 3フレームワークすべてが、出力の構造を明示的に指定している。Kiroはテンプレートファイルを参照させ、SpecKitはプロンプト内に構造を記述し、OpenSpecはスキーマの指示に従わせる。

**評価チェック**:
- [ ] 各コマンドの出力として期待される成果物の構造（セクション、必須項目）が定義されているか
- [ ] 出力の長さに関する制約があるか（「1ページ以内」「100行以内」等）
- [ ] 成果物の保存先パスが具体的に指定されているか

**是々非々の判断**: 出力の長さ制約は必須とする。SpecKitの教訓として、制約なしの仕様生成は「小さなバグに16受入基準」のような肥大化を招く。ただし、制約は固定値ではなくワークフローレベルに応じた可変値が望ましい（Standardは1ページ以内、Fullは必要に応じて拡張可能）。

### 3.2 必須：要件トレーサビリティの仕組み

**根拠**: Kiro・SpecKitの両方が、数値IDによるトレーサビリティを厳格に強制している。

```
# Kiro design-principles.md の例
Requirement IDs:
- Reference requirements as `2.1, 2.3` without prefixes
- All requirements MUST have numeric IDs
- Use N.M-style numeric IDs
```

**評価チェック**:
- [ ] Research → Plan → Implement間で、元の要件/チケットへの参照が維持されるか
- [ ] 参照形式が統一されているか（自由記述ではなくID体系があるか）

**不要と判断した観点**: SpecKitのような「80%以上のトレーサビリティカバレッジ」の数値基準。RPIフローではタスクの粒度が小さく、全タスクが1-2個の要件に紐づくため、カバレッジ率の計測は過剰。

### 3.3 推奨：「仮定の明示」ルール

**根拠**: SpecKitはエージェントに「informed guessを行い、仮定をAssumptionsセクションに記録する」よう指示している。最大3つの`[NEEDS CLARIFICATION]`マーカーに制限し、それ以外は合理的なデフォルトで埋める。

```
# SpecKit speckit.specify.md の例
1. Make informed guesses: Use context, industry standards, and common patterns
2. Document assumptions: Record reasonable defaults in the Assumptions section
3. Limit clarifications: Maximum 3 [NEEDS CLARIFICATION] markers
```

**評価チェック**:
- [ ] エージェントが判断に迷った場合の行動指針があるか（「質問する」vs「仮定して進む」の基準）
- [ ] 仮定を明示的に記録する仕組みがあるか

**是々非々の判断**: 推奨とする。必須としない理由は、RPIのResearch/Planフェーズではオンサイトが主導するため、エージェント単独での仮定判断の場面が限定的であるため。ただし、Implementフェーズでオフショア+エージェントが作業する際には、エージェントが「仮定して進む」か「ブロックして質問する」かの判断基準が必要となる。

---

## 評価軸4: エラーハンドリングとフォールバック

### 4.1 必須：想定外状態への対処指示

**根拠**: 3フレームワークすべてが「Safety & Fallback」セクションを持ち、想定外の状態（ファイル不在、前フェーズ未完了、テンプレート不在等）への対処を明記している。

```
# Kiro spec-requirements.md の例
Error Scenarios:
- Missing Project Description: ask user for feature details
- Ambiguous Requirements: Propose initial version and iterate
- Template Missing: use inline fallback structure with warning
```

**評価チェック**:
- [ ] 前提条件が満たされない場合（前フェーズ未完了、ファイル不在等）の動作が定義されているか
- [ ] フォールバック時にユーザーに通知されるか（サイレントフォールバックではないか）
- [ ] リトライ上限があるか（無限ループにならないか）

SpecKitのチェックリスト検証はmax 3回のリトライ後にユーザー警告＋続行を採用しており、これは実用的な設計である。

### 4.2 推奨：曖昧性への対処方針

**根拠**: OpenSpecのapplyコマンドは、実装中に曖昧性や設計上の問題を発見した場合に「一時停止してユーザーに確認する」よう指示している。

**評価チェック**:
- [ ] 実装中にPlanの不備を発見した場合の行動指針があるか（「そのまま進む」「ブロックする」「仮実装して記録する」等）
- [ ] ブロック時の報告フォーマットがあるか（何が曖昧で、どの判断が必要か）

**是々非々の判断**: 推奨とする。必須としない理由は、RPIフローではPlan段階で双方向Q&Aを実施するため、Implement段階での曖昧性発見は構造的に減少するはずだからである。ただし、ゼロにはならないため対処方針の存在は望ましい。

---

## 評価軸5: 状態管理と再開可能性

### 5.1 必須：フェーズ進行の記録

**根拠**: 3フレームワークすべてが、メタデータファイル（spec.json、チェックリスト、.openspec.yaml）にフェーズ進行を記録している。

```
# Kiro spec-requirements.md の例
Update Metadata:
- Set phase: "requirements-generated"
- Set approvals.requirements.generated: true
- Update updated_at timestamp
```

**評価チェック**:
- [ ] 各フェーズの完了がメタデータに記録されるか
- [ ] 中断後に再開した場合、どこから続行すべきか判断できるか
- [ ] タスク単位の完了がチェックボックス等で追跡されるか（`- [ ]` → `- [x]`）

### 5.2 不要と判断：セマンティックバージョニングによるメタデータ管理

**根拠**: SpecKitのConstitutionはsemantic versioning（MAJOR/MINOR/PATCH）で管理されるが、これは日常的な機能開発のメタデータ管理には過剰である。RPIフローではタイムスタンプ＋フェーズ名の組み合わせで十分。

---

## 評価軸6: 実行時の品質担保メカニズム

### 6.1 必須：テスト駆動の実装指示

**根拠**: Kiroのspec-impl.mdが明示するTDDサイクル（RED → GREEN → REFACTOR → VERIFY → MARK COMPLETE）は、エージェントによる実装の品質を担保する最も実効的な仕組みである。

```
# Kiro spec-impl.md の例
1. RED - Write Failing Test
2. GREEN - Write Minimal Code (simplest solution to make test pass)
3. REFACTOR - Clean Up
4. VERIFY - All tests pass, no regressions
5. MARK COMPLETE - Update checkbox in tasks.md
```

**評価チェック**:
- [ ] Implementコマンドにテスト先行の指示があるか
- [ ] 「最小限の実装」を促す指示があるか（過剰実装の防止）
- [ ] リグレッション確認の指示があるか
- [ ] タスク完了時のマーキング指示があるか

**是々非々の判断**: TDD強制を必須とする。「テストなしで実装し、後からテストを書く」アプローチはエージェント実装においてリスクが高い。エージェントは「動くコード」を生成する能力は高いが、「正しいコード」かどうかの自己検証能力は限定的である。テストが唯一の客観的検証手段であり、実装前に書くことで「何を作るべきか」の明確化にも寄与する。

### 6.2 推奨：機械的品質チェックの組み込み

**根拠**: SpecKitのimplementコマンドは、実装前にプロジェクトのセットアップ（gitignore、eslintignore等）を検証する。[逆瀬川の分析](https://nyosegawa.com/posts/coding-agent-workflow-2026/)でもLinter/Hooks/型チェッカーによる機械的品質矯正が推奨されている。

**評価チェック**:
- [ ] 実装後にlint/型チェック/テスト実行を指示しているか
- [ ] CI連携の指示があるか（PRオープン時の自動チェック等）

### 6.3 不要と判断：仕様品質のメタ検証（"Unit Tests for Requirements"）

**根拠**: SpecKitのchecklistコマンドは「要件自体の品質を検証する」ユニークな仕組みだが、9カテゴリ（Completeness、Clarity、Consistency等）の全量評価は、RPIのStandard/Lightweightワークフローには過剰である。

ただし、Fullワークフローにおいて、以下の3観点に限定した簡易チェックは検討に値する：
- **Completeness**: 必要な要件がすべて記載されているか
- **Clarity**: 曖昧な用語が定量化されているか
- **Consistency**: 要件間に矛盾がないか

---

## 評価軸7: スコープ制御（何をやらないか）

### 7.1 必須：各コマンドの責務境界の明示

**根拠**: OpenSpecのexploreコマンドは「思考のためのモードであり、実装は禁止」と明示している。Kiroのtasks-generation.mdはデプロイ・ドキュメントタスクを明示的に除外している。

```
# OpenSpec explore.md の例
IMPORTANT: Explore mode is for thinking, not implementing.
You may read files, search code, and investigate the codebase,
but you must NEVER write code or implement features.
```

**評価チェック**:
- [ ] 各コマンドが「何をするか」だけでなく「何をしないか」を明示しているか
- [ ] Researchコマンドが実装を開始しないことが保証されているか
- [ ] Planコマンドがコードを生成しないことが保証されているか

**なぜ重要か**: エージェントは指示がなければ「親切に」余計な作業を行う傾向がある。Researchフェーズで「ついでに実装も始めました」という動作を防止するには、ネガティブ指示（「やってはいけないこと」）が不可欠。前回の検討資料でも述べた通り、ネガティブルールはモデル進化でも陳腐化しにくく、長期的に有効。

### 7.2 推奨：コンテキストクリアの指示

**根拠**: Kiroのspec-tasks.mdは「`/kiro:spec-impl`実行前に会話履歴をクリアし、コンテキストを解放せよ」と明示している。

**評価チェック**:
- [ ] フェーズ間でのコンテキストリセットの指示があるか
- [ ] 長時間セッションでのコンテキスト肥大化への対処があるか

---

## 評価軸8: フレームワーク間で発見されたアンチパターンの回避

既存フレームワークの分析から、RPIプロンプトに含めるべきでないパターンを特定した。

### 8.1 回避：テンプレートへの厳格な準拠指示と柔軟なフォールバックの矛盾

**問題**: Kiroは「テンプレート構造に厳密に従え」と指示しつつ、「テンプレートが不在の場合はインラインのフォールバック構造を使え」と記述している。これは一見矛盾しており、エージェントの判断を混乱させうる。

**RPIでの対策**: テンプレートを使う場合は「テンプレートを優先するが、不在の場合は以下の基本構造を使用」と段階的に記述する。「厳密に従え」と「柔軟にフォールバックせよ」を同一プロンプトに含めない。

### 8.2 回避：レビュー観点数の固定（「最大3つの問題を指摘せよ」）

**問題**: Kiroのdesign-review.mdは「Critical Issues（最大3件）」を求めるが、実際にはゼロ件（良好な設計）から5件以上（複雑な設計）まで変動しうる。件数の固定はパディング（無理やり問題を作る）またはトランケーション（重要な問題の見落とし）を招く。

**RPIでの対策**: 「重大な問題があれば指摘する。件数の目安はなく、実装をブロックすべき問題のみ報告」と記述する。

### 8.3 回避：過剰な質問制限

**問題**: SpecKitは「最大3つのNEEDS CLARIFICATIONマーカー」に制限し、それ以外はinformed guessで埋めるよう指示する。これは個人開発では効率的だが、30人チームでは「間違った仮定に基づく実装」のリスクが高い。

**RPIでの対策**: 質問数の上限は設けず、代わりに質問の粒度を制御する。「PlanレベルではYes/Noで回答可能な質問に限定する」「実装詳細の質問はImplementフェーズに回す」のように、フェーズに応じた質問スコープを定義する。

### 8.4 注意：変更管理の不在（3フレームワーク共通の欠落）

**問題**: 3フレームワークのいずれも、「実装中に要件変更が発生した場合」の対処手順を定義していない。Plan承認後に仕様変更が必要になった場合、どのフェーズに戻るべきか、再承認は必要か、が不明確。

**RPIでの対策**: Implementコマンドに以下の判断基準を含める。
- **Plan範囲内の微調整**（変数名変更、エラーメッセージ修正等）：そのまま続行し、完了報告で言及
- **Plan範囲外だがアーキテクチャに影響しない変更**：ブロックし、Plan更新を要求
- **アーキテクチャに影響する変更**：ブロックし、Research→Planの再実行を要求

---

## 評価サマリーテーブル

| # | 評価観点 | 判定 | 根拠 |
|---|---------|------|------|
| 1.1 | コンテキストロードの明示的指示 | **必須** | 3FW共通。幻覚防止の基本設計 |
| 1.2 | コンテキストの階層設計 | **必須** | Context Rot防止。Anthropic・逆瀬川の知見 |
| 2.1 | 承認ゲートの存在と強制 | **必須** | 3FW共通。品質統制の要 |
| 2.2 | Fast-Trackオプション | **推奨** | 柔軟性担保。ただしオフショア体制では慎重に |
| 2.3 | 全フェーズへのゲート設置 | **不要** | Approval Fatigueの原因。SpecKitの教訓 |
| 3.1 | 出力フォーマットの明示的定義 | **必須** | 3FW共通。出力品質の基盤 |
| 3.2 | 要件トレーサビリティ | **必須** | Kiro・SpecKit共通。監査・検証の基盤 |
| 3.3 | 仮定の明示ルール | **推奨** | SpecKit由来。チーム運用では限定的に有効 |
| 4.1 | 想定外状態への対処指示 | **必須** | 3FW共通。運用安定性の基盤 |
| 4.2 | 曖昧性への対処方針 | **推奨** | OpenSpec由来。Plan段階のQ&Aで軽減可能 |
| 5.1 | フェーズ進行の記録 | **必須** | 3FW共通。再開可能性の基盤 |
| 5.2 | セマンティックバージョニング | **不要** | 過剰。タイムスタンプ＋フェーズ名で十分 |
| 6.1 | テスト駆動の実装指示 | **必須** | Kiro由来。エージェント実装の品質担保に不可欠 |
| 6.2 | 機械的品質チェック | **推奨** | CI連携として実装すべきだがプロンプト内は推奨 |
| 6.3 | 仕様品質のメタ検証 | **不要** | SpecKit由来。RPIのスコープには過剰 |
| 7.1 | 各コマンドの責務境界の明示 | **必須** | OpenSpec由来。ネガティブ指示が不可欠 |
| 7.2 | コンテキストクリアの指示 | **推奨** | Kiro由来。長時間セッション対策 |

**必須: 9項目 / 推奨: 5項目 / 不要: 3項目**

---

## コマンド別チェックリスト

前述の17観点を各コマンドの性質に照らして適用可否を判定したものである。各コマンドで確認すべき観点のみを記載し、該当しない観点は省略している。

---

### `/research` — 既存コード調査・影響範囲分析

Researchはコードを書かず、情報を集めて構造化するフェーズである。最大のリスクは「調査が浅い」または「スコープが曖昧なまま次フェーズに進む」ことである。

#### R-1: コンテキストロード

チケット/課題の説明、CLAUDE.md、steering/（tech.md, structure.md）を明示的に読み込む指示があるか。product.mdは不要（ビジネスコンテキストはチケットに含まれる）。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-requirements.md` L24-30 | `Load ALL steering context: Read entire .kiro/steering/ directory including: Default files: structure.md, tech.md, product.md` |
| SpecKit | `speckit.specify.md` L14-19 | `Load context: Read FEATURE_SPEC and .specify/memory/constitution.md. Load IMPL_PLAN template` |
| OpenSpec | `opsx/explore.md` L87-97 | `Check what exists ... if the user mentioned a specific change name, read its artifacts for context` |

#### R-2: コンテキスト階層

steeringは参照するが、前フェーズ成果物のロードがないこと（Researchは起点のため）。不要なドメインスキルをロードしていないか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-requirements.md` L27-30 | `This provides complete project memory and context`（全量ロード。階層設計なし） |
| SpecKit | `speckit.checklist.md` L83-87 | `Load only necessary portions relevant to active focus areas (avoid full-file dumping), Prefer summarizing long sections, Use progressive disclosure` |
| OpenSpec | `opsx/explore.md` L81-97 | `You have full context... use it naturally, don't force it` |

#### R-3: 出力フォーマット

成果物の構造が定義されているか。最低限「影響範囲」「既存コードの制約」「リスク・懸念」の3セクションを含むか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-requirements.md` L60-70 | `Generated Requirements Summary, Document Status, Next Steps, Format Requirements: Use Markdown headings` |
| SpecKit | `speckit.plan.md` L119-125 | `Decision: [what was chosen], Rationale: [why chosen], Alternatives considered: [what else evaluated]` |
| OpenSpec | `opsx/explore.md` L56-72 | `Use ASCII diagrams liberally... System diagrams, state machines, data flows, architecture sketches` |

#### R-4: 出力の長さ制約

Standardは1ページ以内、Fullは必要に応じて拡張可、のようにワークフローレベルに応じた制約があるか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-requirements.md` L70 | `Keep summary concise (under 300 words)` |
| SpecKit | — | 該当なし（明示的な長さ制約なし） |
| OpenSpec | `opsx/explore.md` L147 | `Be brief (this is thinking time)`（制約ではなく方針） |

#### R-5: 責務境界（ネガティブ指示）

「コードを書かない」「設計判断を下さない（選択肢を列挙するに留める）」が明記されているか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-requirements.md` L47-48 / `gap-analysis.md` 全体 | `Focus on WHAT, not HOW (no implementation details)` / gap-analysis全体が「information over decisions」の原則 |
| SpecKit | `speckit.specify.md` L241-244 | `Focus on WHAT users need and WHY. Avoid HOW to implement (no tech stack, APIs, code structure)` |
| OpenSpec | `opsx/explore.md` L10 | `Explore mode is for thinking, not implementing... you must NEVER write code or implement features` |

#### R-6: エラーハンドリング

対象コードが存在しない場合、チケットの情報が不足している場合の対処が定義されているか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-requirements.md` L74-81 | `Missing Project Description: ask user for feature details` / `Ambiguous Requirements: Propose initial version and iterate` |
| SpecKit | `speckit.specify.md` L95-101 | `If empty: ERROR 'No feature description provided'` / `If no clear user flow: ERROR 'Cannot determine user scenarios'` |
| OpenSpec | `opsx/explore.md` L165-174 | `Don't fake understanding - If something is unclear, dig deeper` |

#### R-7: 仮定の明示（推奨）

調査中に判明しなかった事項を「未確認事項」として明記する指示があるか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-requirements.md` L47-52 | `Generate initial version first, then iterate with user feedback (no sequential questions upfront)` |
| SpecKit | `speckit.specify.md` L106 | `Use reasonable defaults for unspecified details (document assumptions in Assumptions section)` |
| OpenSpec | `opsx/explore.md` L74-77 | `Surface risks and unknowns - Identify what could go wrong, Find gaps in understanding` |

**Researchで不要な観点**:
- 承認ゲート（Researchの出力はPlanの入力であり、独立した承認は不要）
- TDD指示（コードを生成しない）
- フェーズ進行の記録（Lightweightでは省略可。Standardではオプション）
- トレーサビリティ（チケットIDの参照があれば十分。数値ID体系は不要）

---

### `/plan` — 実装方針の策定

Planは「何をどう作るか」を決定し、オンサイト-オフショア間の合意形成を担うフェーズである。最大のリスクは「曖昧な計画が承認され、Implementで手戻りが発生する」ことである。

#### P-1: コンテキストロード

Researchの成果物、チケット/課題、steering/（tech.md, structure.md, api-standards.md等）を明示的に読み込む指示があるか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-design.md` L26-31 | `Read all necessary context: .kiro/specs/$1/spec.json, requirements.md, design.md (if exists), Entire .kiro/steering/ directory` |
| SpecKit | `speckit.plan.md` L59 | `Load context: Read FEATURE_SPEC and .specify/memory/constitution.md. Load IMPL_PLAN template` |
| OpenSpec | `opsx/apply.md` L48-53 | `Read the files listed in contextFiles from the apply instructions output` |

#### P-2: コンテキスト階層

Researchの成果物は全量ロード。steeringは技術関連のみ。既存コードの関連箇所をオンデマンドで参照する指示があるか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-design.md` L26-32 | `Read all necessary context`（全量ロード。オンデマンド参照の指示なし） |
| SpecKit | `speckit.plan.md` L63-64 | `Fill Technical Context (mark unknowns as 'NEEDS CLARIFICATION'), Fill Constitution Check section` |
| OpenSpec | `opsx/apply.md` L48-53 | `The files depend on the schema being used: spec-driven: proposal, specs, design, tasks`（スキーマ依存の選択的ロード） |

#### P-3: 出力フォーマット

成果物の構造が定義されているか。最低限「実装方針」「タスク分解」「影響を受ける既存コード」の3セクションを含むか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-design.md` L121-135 | `Status, Discovery Type, Key Findings, Next Action, Research Log` |
| SpecKit | `speckit.plan.md` L101-148 | `Phase 0: Outline & Research... Phase 1: Design & Contracts... Output: research.md` |
| OpenSpec | `opsx/propose.md` L91-106 | `Artifact Creation Guidelines: Read dependency artifacts for context before creating new ones` |

#### P-4: 出力の長さ制約

Standardは1ページ以内。Fullは設計書として拡張可能だが上限あり。制約が明記されているか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-design.md` L133 | `Format: Concise Markdown (under 200 words)`（サマリ部分のみ。設計書本体の制約なし） |
| SpecKit | — | 該当なし（明示的な長さ制約なし） |
| OpenSpec | — | 該当なし（明示的な長さ制約なし） |

#### P-5: 承認ゲート

Plan完了後にImplementに進む前の承認チェックが存在するか。未承認時のハードストップがあるか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-design.md` L33-35 | `Validate requirements approval: If -y flag provided: Auto-approve. Otherwise: Verify approval status (stop if unapproved)` |
| SpecKit | `speckit.plan.md` L64 | `Evaluate gates (ERROR if violations unjustified)` |
| OpenSpec | `opsx/apply.md` L44-46 | `If state: "blocked" (missing artifacts): show message, suggest using /opsx:continue` |

#### P-6: トレーサビリティ

タスク分解の各タスクがチケット/要件IDに紐づいているか。「このタスクは何のために存在するか」が追跡可能か。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-tasks.md` L45-46 / `tasks-generation.md` L38-42 | `list numeric requirement IDs only (comma-separated) without descriptive suffixes` / `Requirements mapping: list numeric IDs only: X.X, Y.Y` |
| SpecKit | `speckit.checklist.md` L173-176 | `MINIMUM: ≥80% of items MUST include at least one traceability reference` |
| OpenSpec | — | 該当なし（スキーマ依存。トレーサビリティの強制なし） |

#### P-7: 責務境界（ネガティブ指示）

「コードを書かない」「テストを実行しない」が明記されているか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-design.md` L111 / `design-principles.md` L5-8 | `Design Focus: Architecture and interfaces ONLY, no implementation code` / `Focus on WHAT the system does, not HOW it's implemented` |
| SpecKit | `speckit.plan.md` 全体構造 | Plan phaseはdesign artifactsの生成のみで、コード生成の指示なし（暗黙的） |
| OpenSpec | `opsx/explore.md` L10 | `you must NEVER write code or implement features`（exploreのみ明示。proposeは暗黙的） |

#### P-8: フェーズ進行の記録

Plan完了・承認状態がメタデータに記録されるか。中断後の再開時にどこから続行すべきか判断できるか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-design.md` L95-99 | `Update Metadata: Set phase: "design-generated", Set approvals.design.generated: true, approved: false, Update updated_at` |
| SpecKit | — | 該当なし（speckit.plan.mdにメタデータ記録の明示なし） |
| OpenSpec | `opsx/apply.md` L28-30 | `Parse the JSON to get: applyRequires: array of artifact IDs needed before implementation`（CLIの状態管理に委譲） |

#### P-9: エラーハンドリング

Researchの成果物が不在/不十分な場合の対処が定義されているか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-design.md` L140-165 | `Requirements Not Approved: Cannot proceed without approved requirements` / `Missing Requirements: Requirements document must exist` / `Template Missing: Use inline basic structure with warning` |
| SpecKit | — | 該当なし（前提条件不足時のエラー処理の明示なし） |
| OpenSpec | `opsx/apply.md` L44-46 | `If state: "blocked" (missing artifacts): show message, suggest using /opsx:continue` |

#### P-10: 質問のスコープ制御（推奨）

質問数の上限を固定せず、質問の粒度を制御しているか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-requirements.md` L51 | `Generate initial version first, then iterate with user feedback (no sequential questions upfront)` |
| SpecKit | `speckit.specify.md` L100-101 | `LIMIT: Maximum 3 [NEEDS CLARIFICATION] markers total, Prioritize clarifications by impact` |
| OpenSpec | `opsx/explore.md` L25-30 | `Open threads, not interrogations - Surface multiple interesting directions and let the user follow what resonates` |

**Planで不要な観点**:
- TDD指示（コードを生成しない）
- 機械的品質チェック（lint/型チェックは不要）
- コンテキストクリア（Planは単一セッションで完結することが多い）

---

### `/implement` — 計画に基づく実装

Implementはエージェントがコードを生成する唯一のフェーズであり、品質リスクが最も高い。最大のリスクは「Planから逸脱した実装」「テストなしの実装」「Plan不備の発見時にブロックせず進行する」ことである。

#### I-1: コンテキストロード

承認済みPlanの成果物、CLAUDE.md、steering/（tech.md, structure.md）を明示的に読み込む指示があるか。Researchの成果物は直接参照不要（Planに集約されているため）。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-impl.md` L26-28 | `Read all necessary context: .kiro/specs/$1/spec.json, requirements.md, design.md, tasks.md, Entire .kiro/steering/ directory` |
| SpecKit | `speckit.implement.md` L82-88 | `REQUIRED: Read tasks.md for the complete task list, REQUIRED: Read plan.md for tech stack, architecture, and file structure` |
| OpenSpec | `opsx/apply.md` L48-53 | `Read the files listed in contextFiles from the apply instructions output` |

#### I-2: コンテキスト階層

Planは全量ロード。steeringは技術関連のみ。ドメイン固有スキルをオンデマンドで参照する指示があるか。**不要なResearch成果物やチケット全文をロードしていないか**。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-impl.md` L26-28 | `Read all necessary context`（全量ロード。階層指定なし） |
| SpecKit | `speckit.implement.md` L82-88 | `REQUIRED: Read tasks.md` / `REQUIRED: Read plan.md` / `IF EXISTS: Read data-model.md` / `IF EXISTS: Read research.md`（条件付きロード） |
| OpenSpec | `opsx/apply.md` L48-53 | `The files depend on the schema being used`（スキーマ依存の選択的ロード） |

#### I-3: 承認ゲートの検証

実装開始前にPlanの承認状態を検証する指示があるか。未承認時にハードストップするか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-impl.md` L30-31 | `Verify tasks are approved in spec.json (stop if not, see Safety & Fallback)` |
| SpecKit | `speckit.implement.md` L51-76 | `Check checklists status... If any checklist is incomplete: STOP and ask user if they want to proceed` |
| OpenSpec | `opsx/apply.md` L42-46 | `If state: "blocked" (missing artifacts): show message, suggest using /opsx:continue` |

#### I-4: TDD強制

RED（テスト先行）→ GREEN（最小実装）→ REFACTOR → VERIFY → MARK COMPLETEのサイクルが明記されているか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-impl.md` L41-65 | `1. RED - Write Failing Test, 2. GREEN - Write Minimal Code, 3. REFACTOR - Clean Up, 4. VERIFY - Validate Quality, 5. MARK COMPLETE` |
| SpecKit | `speckit.implement.md` L140-152 | `Follow TDD approach: Execute test tasks before their corresponding implementation tasks` |
| OpenSpec | `opsx/apply.md` L72 | `Mark task complete in the tasks file`（タスク完了マーキングのみ。TDDサイクルの明示なし） |

#### I-5: 最小実装の指示

「テストを通す最小限のコードを書く」指示があるか。過剰実装を防止する指示があるか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-impl.md` L48-51 | `Implement simplest solution to make test pass, Focus only on making THIS test pass, Avoid over-engineering` |
| SpecKit | `speckit.implement.md` L140-153 | `Keep changes minimal and focused, Phase-by-phase execution: Complete each phase before moving to the next` |
| OpenSpec | `opsx/apply.md` L65-76 | `Make the code changes required, Keep changes minimal and focused` |

#### I-6: リグレッション確認

既存テストの通過を確認する指示があるか。既存機能への影響検証が含まれるか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-impl.md` L59-62 | `All tests pass (new and existing), No regressions in existing functionality, Code coverage maintained or improved` |
| SpecKit | `speckit.implement.md` L163-166 | `Validation checkpoints: Verify each phase completion before proceeding` |
| OpenSpec | `opsx/apply.md` L72-76 | `Pause if: Error or blocker encountered → report and wait for guidance`（リグレッション検証の明示なし） |

#### I-7: タスク完了マーキング

タスク完了時に `- [ ]` → `- [x]` への更新指示があるか。進捗が可視化されるか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-impl.md` L64-65 | `Update checkbox from - [ ] to - [x] in tasks.md` |
| SpecKit | `speckit.implement.md` L160 | `For completed tasks, make sure to mark the task off as [X] in the tasks file` |
| OpenSpec | `opsx/apply.md` L69-70 | `Mark task complete in the tasks file: - [ ] → - [x]` |

#### I-8: 責務境界（ネガティブ指示）

「Planに記載のないタスクを追加しない」「アーキテクチャ判断を変更しない」が明記されているか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-impl.md` L69 | `Task Scope: Implement only what the specific task requires` |
| SpecKit | `speckit.implement.md` L140-146 | `Respect dependencies: Run sequential tasks in order, parallel tasks [P] can run together`（タスクリスト外の作業禁止は暗黙的） |
| OpenSpec | `opsx/apply.md` L65-70 | `For each pending task: Make the code changes required`（タスクリスト準拠は暗黙的） |

#### I-9: 変更管理（Plan逸脱時の対処）

実装中にPlanの不備を発見した場合の判断基準があるか。3段階（微調整→続行 / 非アーキ変更→ブロック / アーキ変更→R/P再実行）が定義されているか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | — | **該当なし**（3FW共通の欠落として評価軸8.4で指摘済み） |
| SpecKit | `speckit.implement.md` L154-159 | `Halt execution if any non-parallel task fails, For parallel tasks [P], continue with successful tasks, report failed ones`（タスク失敗時の対処のみ。Plan逸脱の判断基準なし） |
| OpenSpec | `opsx/apply.md` L72-76 | `Pause if: Implementation reveals a design issue → suggest updating artifacts`（設計問題発見時の一時停止のみ。判断基準の段階分けなし） |

#### I-10: エラーハンドリング

テストが通らない場合のリトライ上限があるか。自力解決不能な場合のエスカレーション指示があるか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-impl.md` L97-99 | `Test Failures: Stop Implementation: Fix failing tests before continuing` |
| SpecKit | `speckit.implement.md` L154-159 | `Halt execution if any non-parallel task fails, Provide clear error messages with context for debugging` |
| OpenSpec | `opsx/apply.md` L72-76 | `Error or blocker encountered → report and wait for guidance` |

#### I-11: 機械的品質チェック（推奨）

実装後にlint/型チェック/全テスト実行を指示しているか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | — | 該当なし（spec-impl.mdに明示的なlint/型チェック指示なし） |
| SpecKit | `speckit.implement.md` L90-132 | `Project Setup Verification: Create/verify ignore files based on actual project setup`（プロジェクト設定の検証のみ。lint実行の明示なし） |
| OpenSpec | — | 該当なし |

#### I-12: コンテキストクリア（推奨）

タスク間でのコンテキストリセット指示があるか。長時間セッションへの対処があるか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-tasks.md` L122-125 | `Before Starting Implementation: Clear conversation history and free up context before running /kiro:spec-impl, This applies when starting first task OR switching between tasks, Fresh context ensures clean state` |
| SpecKit | — | 該当なし |
| OpenSpec | — | 該当なし |

#### I-13: フェーズ進行の記録

各タスクの完了がメタデータに記録されるか。中断後に未完了タスクから続行できるか。

| FW | 参照箇所 | 該当記述 |
|----|---------|---------|
| Kiro | `spec-impl.md` L64-65 | `MARK COMPLETE: Update checkbox from - [ ] to - [x] in tasks.md` |
| SpecKit | `speckit.implement.md` L160 | `For completed tasks, make sure to mark the task off as [X] in the tasks file` |
| OpenSpec | `opsx/apply.md` L78-84 | `On completion or pause, show status: Tasks completed this session, Overall progress: N/M tasks complete` |

---

### 観点の適用マトリクス（全体俯瞰）

| 評価観点 | Research | Plan | Implement |
|---------|:--------:|:----:|:---------:|
| コンテキストロードの明示 | **必須** | **必須** | **必須** |
| コンテキスト階層設計 | **必須** | **必須** | **必須** |
| 承認ゲート | - | **必須**（出力に対して） | **必須**（入力の検証） |
| Fast-Trackオプション | - | 推奨 | - |
| 出力フォーマット定義 | **必須** | **必須** | - （コード自体が出力） |
| 出力の長さ制約 | **必須** | **必須** | - |
| トレーサビリティ | - | **必須** | 推奨（Planのタスクに紐づく） |
| 仮定の明示 | 推奨 | 推奨 | - |
| エラーハンドリング | **必須** | **必須** | **必須** |
| 曖昧性への対処 | - | 推奨 | **必須**（変更管理として） |
| フェーズ進行の記録 | - | **必須** | **必須** |
| TDD強制 | - | - | **必須** |
| 最小実装の指示 | - | - | **必須** |
| リグレッション確認 | - | - | **必須** |
| 機械的品質チェック | - | - | 推奨 |
| 責務境界（ネガティブ指示） | **必須** | **必須** | **必須** |
| コンテキストクリア | - | - | 推奨 |
| 質問スコープ制御 | - | 推奨 | - |

**凡例**: **必須** = 欠落は即修正 / 推奨 = PJの状況に応じて判断 / `-` = 該当なし（含めると過剰）
