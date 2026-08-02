---
name: pro
description: Consult GPT-5.6 Sol Pro through Oracle browser mode. Use when the user asks "proに相談して", "proに聞いて", "proに確認して", or requests an independent expert review.
---

# Pro Consultation Skill

GPT-5.6 Sol Proへセカンドオピニオンを依頼するためのSkill。
通常のAI実装・調査ではなく、重要な判断、設計レビュー、リスク確認、別視点が必要な場合に利用する。

## Trigger

以下のような依頼で使用する。

- proに相談して
- proに聞いて
- proに確認して
- Proにレビューしてもらって
- 別の専門家視点で確認して
- この判断が正しいか確認して

## Role

あなた自身が回答を作る前に、GPT-5.6 Sol Proへ相談する。
ただし、現在の会話やリポジトリ全体をそのまま渡さない。
Proへ渡す前に、現在のコンテキストから「判断に必要な情報」のみ整理する。

## Phase 1: Problem Extraction

まず以下の形式で相談内容を整理する。

```text
# Background

問題の背景。

# Goal

達成したい目的。

# Current Situation

現在の状況。

# Current Approach

現在考えている案。

# Options

検討している選択肢。

# Concerns

懸念している点。

# Decision Needed

何を判断してほしいか。

# Questions

GPT-5.6 Sol Proへ確認したい質問。
```

不要な情報は削除する。
特に以下は除外する。

- 関係ないファイル内容
- 長いログ
- 既に確定した情報
- 判断に影響しない背景

## Phase 2: Pro Consultation

整理した内容のみOracleへ渡す。

実行:

```bash
npx -y @steipete/oracle \
  --engine browser \
  --browser-manual-login \
  --browser-model-strategy current \
  --browser-timeout 20m \
  --browser-auto-reattach-delay 5s \
  --browser-auto-reattach-interval 3s \
  --browser-auto-reattach-timeout 120s \
  -p "<CONSULTATION_PROMPT>"
```

### Consultation Prompt

Proには以下の役割を与える。

```text
あなたはシニアアーキテクト・技術レビュアーです。

以下の相談内容をレビューしてください。

単純に肯定するのではなく、
以下を重点的に確認してください。

1. 前提条件の誤り
2. 見落としているリスク
3. 採用しない方が良い理由
4. より良い代替案
5. 最終的な推奨判断

技術的理由を含めて説明してください。
```

## Phase 3: Response

Proの回答をそのまま返すだけではなく、必要に応じて整理する。

出力形式:

```text
## Proの回答

(Pro回答)

## 自分の判断との比較

必要なら違いを説明。

## 推奨アクション

次に行うべきこと。
```

## Rules

- OpenAI APIは使用しない。
- OPENAI_API_KEYは使用しない。
- Responses APIへ送信しない。
- Oracle browser modeのみ利用する。
- ChatGPTブラウザで選択済みのGPT-5.6 Sol Proを利用する。
- モデル変更操作はしない。
- Proが別の判断をした場合、自分の判断を再評価する。

## Usage Examples

```text
# Architecture Review
User:
proに設計レビューして
整理:
Background:
AWS ECS上でPHPアプリを運用。

Current Approach:
Aurora MySQL + Redis構成。

Decision Needed:
RDS Proxy導入判断。

Questions:
導入メリット・デメリットを確認したい。
```

```text
# Decision Review
User:
proに聞いて、この2案どちらが良い？
整理:
Options:

A:
既存構成維持。

B:
新方式へ変更。

Decision Needed:
長期運用ではどちらを選ぶべきか。
```

```text
# Code Review
User:
proに確認して、この変更大丈夫？
整理:
Change:
認証処理変更。

Risk:
セキュリティ影響確認。

Questions:
見落としている脆弱性がないか。
```
