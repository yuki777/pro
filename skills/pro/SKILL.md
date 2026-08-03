---
name: pro
description: Consult GPT-5.6 Sol + Pro through Oracle browser mode as an independent senior researcher, architect, and reviewer. Use only when the user explicitly invokes pro with requests such as "proに相談して", "proに調査してもらって", "proに聞いて", "proに確認して", or "proにレビューしてもらって". Do not trigger for ordinary research, reviews, design discussions, or second-opinion requests that do not explicitly name pro.
---

# Pro Consultation

GPT-5.6 Sol + Proへ、技術調査、設計、技術判断、リスクについて独立した調査結果とセカンドオピニオンを依頼する。

Proを実装者として扱わない。実装、ファイル変更、外部への投稿、最終判断は現在のエージェントが担当する。

## Phase 1: Problem Extraction

現在の会話やリポジトリ全体を渡さず、判断に必要な情報だけを抽出する。確認済みの事実と現在のエージェントの見解を区別し、次の形式に整理する。

```text
# Background

問題の背景。

# Goal

達成したい目的。

# Current Situation

確認できている現在の状況と事実。

# Current Approach

現在採用している案、または有力と考えている案。

# Options

比較対象となる選択肢。なければ「なし」とする。

# Constraints

技術、期限、予算、互換性、運用などの制約。

# Concerns

懸念、未知事項、見落としの可能性。

# Decision Needed

Proに調査、判断またはレビューしてほしい論点。

# Questions

Proに答えてほしい具体的な質問。
```

調査または判断に必要な場合だけ、関係するコードやdiffの最小範囲、エラーの該当部分、依存する仕様、比較に必要な数値を含める。

次の情報は除外する。

- 関係ないファイルや会話履歴
- 判断に影響しない長いログと重複情報
- 秘密情報、認証情報、APIキー、顧客情報、個人情報
- Proに先入観を与える断定的な誘導

不足情報が結論を大きく変える場合は、Oracleを呼ぶ前に安全な範囲で確認する。確認できない情報は推測で埋めず、未確認事項として明記する。

## Phase 2: Preflight

固定版Oracleを`npx`でダウンロードして実行し、必要なbrowserオプションを確認する。

```bash
command -v node
node --version
node -e 'if (Number(process.versions.node.split(".")[0]) < 24) { console.error("Node.js 24 or later is required"); process.exit(1); }'
command -v npx
npx -y @steipete/oracle@0.17.0 --version
npx -y @steipete/oracle@0.17.0 --debug-help | rg -- '--browser-chrome-path|--browser-manual-login|--browser-keep-browser'
npx -y @steipete/oracle@0.17.0 --help --verbose | rg -- '--browser-model-strategy'
```

Node.jsが24未満、`npx`が存在しない、Oracleの起動に失敗する、または必要なオプションがない場合は実行を止める。未固定のlatestへ変更せず、未実施であることを報告する。

macOSではGoogle Chromeを明示的に使用する。まず次の実行ファイルが存在することを確認する。

```bash
test -x "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
```

存在しない場合は別のブラウザへ自動フォールバックせず、実行を止めてGoogle Chromeのパスを確認する。

Oracle用Chromeが未起動または未ログインの場合だけ、Phase 3のコマンドを接続確認用の短いプロンプトへ置き換えて一度起動する。接続確認をProへの相談として扱わない。`--browser-keep-browser`でChromeを開いたままにする。

実際の相談を送信する直前に、Oracle用Google Chromeで次を目視確認する。

1. Intelligenceメニューで `Pro` が選択されている。
2. バージョンのサブメニューで `GPT-5.6 Sol` が選択されている。
3. 重要な相談では、両方の選択が分かるスクリーンショットを保存する。

片方でも確認できない、またはOracle用Chromeの画面へアクセスできない場合は、プロンプトを送信しない。必要ならユーザーに選択と確認を依頼する。

`--browser-model-strategy current`は現在の選択を維持するだけであり、GPT-5.6 SolまたはProを選択・検証する指定ではない。

## Phase 3: Oracle Consultation

Phase 1の相談内容を、次の固定指示に続けて1つのプロンプトとして渡す。

```text
あなたは独立したシニアリサーチャー・アーキテクト・技術レビュアーです。
実装者として作業せず、提示された調査、判断、レビュー課題へ批判的に取り組んでください。

次の観点を確認してください。

1. 確認できる事実と根拠、推論、未確認事項の区別
2. 前提条件の誤りや不足情報
3. 見落としているリスクと運用影響
4. より良い代替案とトレードオフ
5. 最終的な調査結果または推奨判断とその根拠

外部情報を調査した場合は出典を示してください。情報が不足して断定できない場合や外部情報へアクセスできない場合は、不足情報、条件分岐、未確認範囲を明示してください。

<PHASE_1_CONSULTATION>
```

組み立て済みの相談プロンプト全体を送信直前に再検査する。秘密情報、認証情報、APIキー、顧客情報、個人情報が残っている場合は削除・匿名化し、判断に必要な形へ直せない場合は送信を止める。

Oracleをbrowser modeで実行する。`<UNIQUE_SESSION_SLUG>`には英数字とハイフンで一意な3〜5語の名前を指定する。

相談プロンプトはshellへ直接展開しない。argv配列を渡せる実行手段があれば、プロンプト全体を1つのargvとして渡す。shellコマンドしか使えない場合は、プロンプト全体をPOSIXの単一引用符で囲み、内部の`'`を`'"'"'`へ置換する。二重引用符内へ未加工の相談内容を埋め込まない。

```bash
env -u OPENAI_API_KEY \
  -u AZURE_OPENAI_API_KEY \
  -u AZURE_API_KEY \
  -u OPENROUTER_API_KEY \
  -u ANTHROPIC_API_KEY \
  -u GEMINI_API_KEY \
  -u XAI_API_KEY \
  -u CODEX_API_KEY \
  npx -y @steipete/oracle@0.17.0 \
    --engine browser \
    --browser-manual-login \
    --browser-chrome-path "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
    --browser-port 19194 \
    --browser-reuse-wait 3s \
    --browser-keep-browser \
    --browser-model-strategy current \
    --browser-timeout 20m \
    --browser-auto-reattach-delay 5s \
    --browser-auto-reattach-interval 3s \
    --browser-auto-reattach-timeout 120s \
    --verbose \
    --slug "<UNIQUE_SESSION_SLUG>" \
    -p '<SHELL_ESCAPED_FIXED_INSTRUCTIONS_AND_PHASE_1_CONSULTATION>'
```

ChatGPTに `Pro thinking` や `Answer now` が表示されても、`Answer now`を押さず最終回答まで待つ。

タイムアウト、不完全取得、接続断が発生した場合は、同じ相談を再実行しない。セッションIDまたはslugを使って既存セッションへreattachする。

```bash
npx -y @steipete/oracle@0.17.0 session <SESSION_ID_OR_SLUG> --render
```

Oracleが利用できない場合もAPI modeやResponses APIへフォールバックしない。

## Phase 4: Verification

回答取得後に対象セッションを確認する。

```bash
npx -y @steipete/oracle@0.17.0 status
```

セッションメタデータを確認する。

```bash
jq '{
  status,
  mode,
  elapsedMs,
  chromePath: .browser.config.chromePath,
  modelSelection: .browser.modelSelection,
  conversationUrl: (.browser.archive.conversationUrl // .browser.runtime.tabUrl)
}' ~/.oracle/sessions/<SESSION_ID_OR_SLUG>/meta.json
```

次をすべて満たした場合だけ、GPT-5.6 Sol + Proの送信時UIを確認して相談したと報告する。

- `status`が`completed`
- `mode`が`browser`
- `chromePath`が`/Applications/Google Chrome.app/Contents/MacOS/Google Chrome`
- `conversationUrl`が`https://chatgpt.com/`のURL
- `modelSelection.strategy`が`current`
- 送信直前のUIで`GPT-5.6 Sol`と`Pro`の両方を確認済み
- `Answer now`を押さず、最終回答を取得済み

`current`実行で`verified=no`または`verified=false`となっても、Proではなかったという判定にはしない。ただしOracle自身によるモデル検証済みとも報告しない。起動ログのモデル名や回答速度、モデル自身の自己申告を選択証拠に使わない。

ChatGPTサーバー内部で実行されたモデルのattestationは取得できない。この限界を必ず報告する。

## Phase 5: Response

Proの調査結果や回答をそのまま転送せず、実際のコンテキストと根拠に照らして整理する。

```text
## 実行確認

実行経路: ChatGPT browser / subscription
ブラウザ: <chromePath>
APIキー環境変数: コマンドに列挙したprovider keyをOracle子プロセスから解除
送信時UI: <GPT-5.6 Sol + Proを確認済み、または未確認>
Oracleセッション: <status> / <mode>
Oracleモデル検証: <verifiedの値>
サーバー側attestation: 取得不可

## Proの調査・回答

調査結果、回答の要点と結論。

## 重要な指摘

前提の誤り、重大なリスク、代替案、未確認事項。

## 自分の判断との比較

一致点、相違点、Proの回答を受けて再評価した点。

## 推奨アクション

ユーザーが次に行うべき具体的な対応。
```

実際にOracleから最終回答を取得していない場合は、Proへ相談したと表現しない。Proの回答を無条件に採用せず、事実誤認や現在の環境と合わない提案を明示し、最終判断は現在のエージェントが行う。

## Prohibitions

- OpenAI API、Responses API、API modeを使用しない。
- `--model gpt-5.6-sol-pro`を使用しない。
- `--reasoning-mode pro`をbrowser modeの代わりに使用しない。
- `--browser-thinking-time heavy`をPro選択の証拠に使用しない。
- Oracle失敗時に別の有料APIへフォールバックしない。
- Proへ実装、ファイル変更、外部投稿を依頼しない。
