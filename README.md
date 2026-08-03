# pro

AIエージェントからGPT-5.6 Sol + Proへ、技術調査、設計判断、レビューを依頼するためのSkillです。OpenAI APIではなく、[Oracle](https://github.com/steipete/oracle)のbrowser modeとChatGPT subscriptionを使用します。

このSkillは、ユーザーが「proに相談して」「proに調査してもらって」「proにレビューしてもらって」のように明示した場合だけ起動します。Proは独立したシニアリサーチャー・アーキテクト・技術レビュアーとして調査結果や意見を返し、実装と最終判断は呼び出し元のAIエージェントが担当します。

## 背景

難しい調査や設計判断では、実装を担当しているエージェントとは独立した視点が役立ちます。一方、ブラウザ経由でProへ相談する処理には、次の問題があります。

- 会話やリポジトリ全体を送ると、無関係な情報や秘密情報まで共有するおそれがある
- CLIのモデル指定だけでは、ChatGPT UIでGPT-5.6 SolとProが選ばれていることを保証できない
- browser modeの失敗時にAPIへ切り替えると、subscription経由という前提や料金条件が変わる
- 途中で失敗した実行を「Proへ相談済み」と扱うと、存在しないレビュー結果に基づいて判断してしまう

`pro` Skillは、必要な情報だけを抽出し、送信前のUI確認と実行後のセッション確認を必須にするfail-closedな相談手順を提供します。

## 解決する問題

- **意図しない外部送信を防ぐ**: `pro`を明示した依頼でのみ起動する
- **共有する情報を最小化する**: 背景、目的、制約、懸念、質問など9項目へ整理してから送る
- **subscription経由に限定する**: Oracleのbrowser modeだけを使い、APIへフォールバックしない
- **通常版Google Chromeを固定する**: Chrome Canaryなどへ自動で切り替えない
- **モデル選択を誤認しない**: 送信直前にChatGPT UIで`GPT-5.6 Sol`と`Pro`の両方を確認する
- **未完了を成功扱いしない**: Oracleの最終回答とセッション情報を確認できた場合だけ相談結果として報告する
- **判断責任を分離する**: Proは調査、助言、レビューを担当し、AIエージェントが事実確認、実装、最終判断を行う

## 処理フロー

1. AIエージェントが現在の状況から、調査や判断に必要な情報だけを9項目へ整理する
2. Oracle、必要なbrowserオプション、Google Chromeの実行ファイルを確認する
3. Oracle用Chromeで`GPT-5.6 Sol`と`Pro`の選択を目視確認する
4. APIキーを子プロセスから外し、Oracleのbrowser modeで相談する
5. セッション完了、browser mode、Chromeパス、ChatGPTの会話URLを確認する
6. Proの回答を現在の根拠と比較し、採用判断と具体的な次の対応をAIエージェントがまとめる

詳細な手順と停止条件は[`skills/pro/SKILL.md`](skills/pro/SKILL.md)にあります。

## 依存関係

| 依存 | 要件 |
| --- | --- |
| AIエージェント | Agent Skillsを利用でき、`gh skill`が対応している環境 |
| OS | macOS |
| Google Chrome | `/Applications/Google Chrome.app/Contents/MacOS/Google Chrome`にインストール済み |
| ChatGPT | GPT-5.6 SolとProを利用できるsubscription、およびOracle用Chromeでのログイン |
| GitHub CLI | `gh skill`を利用できるバージョン |
| Node.js | 24以上で、`npx`を利用できる環境 |
| Oracle | `npx -y @steipete/oracle@0.17.0`で取得・実行 |
| CLI | `rg`と`jq` |

Oracleは、検証済みの0.17.0を`npx`で必要時にダウンロードして実行します。再現性を保つためバージョンを固定し、preflightで必要なbrowserオプションを確認します。

OpenAI APIキーは不要です。実行時には、主要なAI providerのAPIキー環境変数をOracleの子プロセスから外します。

## インストール

GitHub CLIで、利用するAIエージェントのユーザースコープへインストールします。

```bash
gh skill install yuki777/pro pro --scope user
```

対話中にインストール先のAIエージェントを選択します。非対話で実行する場合は、`--agent claude-code`、`--agent codex`、`--agent gemini-cli`、`--agent pi`などを追加してください。利用可能な値は`gh skill install --help`で確認できます。

インストール後、AIエージェントを新しいセッションで起動し、Skill一覧に`pro`が表示されることを確認します。

## 事前確認

初回利用前に、OracleとGoogle Chromeを確認します。

```bash
command -v node
node --version
node -e 'if (Number(process.versions.node.split(".")[0]) < 24) { console.error("Node.js 24 or later is required"); process.exit(1); }'
command -v npx
npx -y @steipete/oracle@0.17.0 --version
npx -y @steipete/oracle@0.17.0 --debug-help | rg -- '--browser-chrome-path|--browser-manual-login|--browser-keep-browser'
npx -y @steipete/oracle@0.17.0 --help --verbose | rg -- '--browser-model-strategy'
test -x "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
```

初回のpreflightでは、固定版Oracleが`npx`でダウンロードされます。Oracle用ChromeでChatGPTへログインしていない場合は、初回実行時に短い接続確認が行われます。開いたGoogle Chromeで手動ログインし、そのブラウザを閉じずに再利用してください。

## 使い方

AIエージェントへ、`pro`を含む依頼を明示的に送ります。

```text
proに相談して。このAPIの認証方式をJWTに変更すべきか判断したい。
```

```text
proに調査してもらって。このライブラリの公式仕様と既知の制約を整理してほしい。
```

```text
proにこの設計をレビューしてもらって。特に障害時の復旧手順と運用負荷を確認したい。
```

```text
実装前にproへ確認して。現在案と代替案のトレードオフを比較してほしい。
```

通常の「調査して」「レビューして」「設計を相談したい」といった依頼では自動起動しません。Proへ共有する内容に不足がある場合や、送信前のUIをAIエージェントから確認できない場合は、Oracleを実行する前に確認を求めます。

相談中にChatGPTへ`Pro thinking`や`Answer now`が表示されても、`Answer now`は押さずに最終回答を待ちます。タイムアウトや接続断が起きた場合は、同じ相談を新規送信せず、既存のOracleセッションへ再接続します。

## 安全性

- 秘密情報、認証情報、APIキー、顧客情報、個人情報を相談内容へ含めない
- リポジトリ全体や会話履歴全体ではなく、判断に必要な最小範囲だけを送る
- Google Chrome、Oracle、必要なオプションのいずれかを確認できなければ実行を止める
- GPT-5.6 SolとProの両方を送信直前に確認できなければプロンプトを送らない
- Oracleが失敗してもOpenAI APIや別の有料APIへ切り替えない
- Proへファイル変更、実装、外部投稿を依頼しない

## 検証できる範囲と限界

このSkillは、Oracleがbrowser modeで動作したこと、通常版Google Chromeを使用したこと、ChatGPTの会話URLを取得したこと、送信時UIでGPT-5.6 SolとProが選択されていたことを確認します。

`--browser-model-strategy current`は、UIの現在の選択を維持する指定です。Oracleによるモデル選択の証明ではありません。そのため`verified=no`は直ちに「Proではなかった」ことを意味しませんが、Oracleがモデルを検証した証拠としても扱いません。

ChatGPTサーバー内部で実際に動作したモデルのattestationは取得できません。起動ログ、応答速度、モデル自身の自己申告もモデル選択の証拠には使用しません。この限界は相談結果と一緒に報告します。

## 開発と検証

Skillを変更した場合は、frontmatterと構造を検証します。

```bash
gh skill publish --dry-run
git diff --check
```

検証コマンドの成功はSkillファイルの形式を確認するものであり、ChatGPTへのログインや実際のPro相談が成功したことまでは保証しません。
