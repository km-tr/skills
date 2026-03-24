---
name: address-reviews
description: 直前の会話に含まれるレビュー結果を精査し、妥当な指摘に対して修正を実施する。レビュアーの指摘がすべて正しいとは限らないため、各コメントを批判的に評価する。「レビュー対応して」「レビューコメント修正して」「指摘を直して」「レビュー指摘に対応して」などのリクエストで使用。
---

# address-reviews

## Overview

直前の会話に含まれるレビュー結果（severity-classified review、PRレビューコメント、コードレビュー指摘など形式を問わない）を精査し、各指摘を批判的に評価した上で、妥当なもののみ修正を実施する。

## When to Use

- レビュー結果を受けて修正対応が必要なとき
- codex-exec-reviewなどの自動レビュー出力に対応するとき
- PRのレビューコメントに対応するとき
- Symptoms: "レビュー対応して", "レビューコメント修正して", "指摘を直して", "fix review issues", "address review findings"

**Not for:** レビュー指摘がない場合、レビュー自体の実行（それはcodex-exec-reviewなど別スキルの役割）

## Scope

レビュー指摘の**評価・修正・再サマリー**を行う。レビュー自体の実行やブランチのマージは行わない。

## Workflow

### Step 1 — Collect Review Findings

直前の会話からレビュー指摘を特定・一覧化する。以下のいずれの形式にも対応する:

- **severity-classified形式**（codex-exec-review等）: `[SEVERITY] Title` + `File:` + `Issue:` + `Fix:`
- **PRレビューコメント**: ファイル・行番号付きのインラインコメント
- **自由形式のレビュー**: 箇条書きやテキストベースの指摘

各指摘について、元のレビュアー名（エージェント名、GitHub ユーザー名など）と対象ファイル・行を記録する。

レビュー結果が会話中に見つからない場合は、ユーザーに提供を依頼する。

### Step 2 — Evaluate Each Finding Independently

各指摘について以下を実施する:

1. **対象コードを実際に読む** — 指摘されたファイル・行を確認する
2. 妥当性を評価する:
   - 指摘された問題は実際にコード上に存在するか？
   - 提案された修正は適切か、より良いアプローチはないか？
   - false positive（レビュアーがプロジェクトのコンテキストを欠いている等）ではないか？
3. severityが付与されていない指摘には、自分で CRITICAL / HIGH / MEDIUM / LOW を割り当てる
4. 各指摘を分類する:
   - **FIX** — 指摘は妥当であり、修正すべき
   - **SKIP** — 指摘は不正確、false positive、または対応不要（理由を明記すること）
   - **DOWNGRADE** — 問題は実在するがseverityが過大（正しいseverityを明記）

**重要原則: レビュアーの指摘を鵜呑みにしない。** 各指摘を実コードに照らして独立に判断すること。

### Step 3 — Apply Fixes

CRITICAL → HIGH → MEDIUM → LOW の優先順で処理する。

FIXと分類した各指摘について:

1. 対象ファイルを開く
2. 修正を適用する（レビュアーの提案を出発点にしつつ、より良い方法があれば改善する）
3. 修正が周辺コードを壊さないか確認する
4. 何を変更したか記録する

DOWNGRADE した指摘: 修正後のseverityがCRITICALまたはHIGHの場合のみ修正する。

### Step 4 — Produce Summary

以下の形式でサマリーを出力する:

```
## Address-Reviews Summary

### Addressed Issues

#### [CRITICAL] Issue Title (reviewer-name)
- **File**: path/file.ts:line
- **Problem**: brief description of what was wrong
- **Fix applied**: brief description of what was changed

#### [HIGH] Issue Title (reviewer-name)
- **File**: path/file.ts:line
- **Problem**: brief description
- **Fix applied**: brief description

### Not Addressed

#### [MEDIUM] Issue Title (reviewer-name)
- **File**: path/file.ts:line
- **Reason**: explanation of why this was skipped (false positive / severity downgraded / stylistic preference / etc.)

#### [LOW] Issue Title (reviewer-name)
- **File**: path/file.ts:line
- **Reason**: explanation

### Updated Review Summary

| Severity | Original | Fixed | Remaining |
|----------|----------|-------|-----------|
| CRITICAL | N        | N     | N         |
| HIGH     | N        | N     | N         |
| MEDIUM   | N        | N     | N         |
| LOW      | N        | N     | N         |

Verdict: APPROVE/WARNING/BLOCK — explanation based on remaining issues
```

## Approval Criteria

**残存する**（未修正の）指摘のみに基づいてverdictを再評価する:

- **Approve**: CRITICALおよびHIGHの残存なし
- **Warning**: HIGHのみ残存（注意付きでマージ可）
- **Block**: CRITICALが残存 — マージ前に修正必須

全てのCRITICAL/HIGH指摘が修正済み、または妥当にSKIP/DOWNGRADEされた場合、verdictはAPPROVEに変更する。

## Common Mistakes

| Mistake | Fix |
|---|---|
| 全指摘を無条件に修正する | 各指摘を独立に評価し、false positiveはスキップ（理由付き） |
| 実コードを読まずに修正を判断する | 必ず対象ファイル・行を確認してから判断する |
| Verdictを再カウントせずに変更する | 全修正後にseverity別の残数を再集計する |
| CRITICALがあるのにMEDIUM/LOWを先に対応する | CRITICAL → HIGH → MEDIUM → LOW の優先順で処理 |
| レビュアーの修正案をそのまま適用する | 提案を出発点にしつつ、より良い方法があれば改善する |
| 会話中にレビュー結果がない | ユーザーにレビュー結果の提供を依頼する |
| 修正が新たな問題を生む | 各変更後に周辺コードへの影響を確認する |
| severity未付与の指摘をそのまま放置する | 自分でseverityを割り当ててからトリアージする |
