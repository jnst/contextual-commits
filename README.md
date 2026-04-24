[Spec](SPEC.md)
[License](LICENSE)

# Contextual Commits

> これは [berserkdisruptors/contextual-commits](https://github.com/berserkdisruptors/contextual-commits) の日本語訳 fork です。本家の仕様・ドキュメントを日本語化したものであり、規約本体は同一です。

**Conventional Commits は WHAT(何が変わったか)を標準化しました。Contextual Commits は WHY(なぜ)を加えます。**

git コミット body に意思決定の記録を埋め込むための規約です。コミットはコード変更だけでなく、その形を決めた意図と理由を、どのツールからもパース可能・どのエージェントからもクエリ可能な、型付きの action line として運びます。

新しいツールは要りません。インフラも要りません。ただ、より良いコミットを残すだけです。

---

**一般的な AI によるコミット:**

```
feat(auth): Google OAuth プロバイダーを実装

passport.js との統合により GoogleAuthProvider クラスを追加。
/api/auth/callback/google に callback route handler を作成。
offline アクセススコープでrefresh tokenのロジックを追加。
複数プロバイダーをサポートするよう認証ミドルウェアを更新。
```

これは diff を見れば分かることを言い換えているだけです。signal (diff では示せない有益な情報) は何も増えていません。

**Contextual commit:**

```
feat(auth): Google OAuth プロバイダーを実装

intent(auth): Google を皮切りにソーシャルログインを導入。続いて GitHub、Apple を予定
decision(oauth-library): multi-provider 対応の柔軟性のため auth0-sdk ではなく passport.js を採用
rejected(oauth-library): auth0-sdk — 独自のセッションモデルに縛られ、redis ストアと両立しない
constraint(callback-routes): 既存規約に従い /api/auth/callback/:provider パターンに準拠する必要がある
constraint(session-store): redis の TTL は 24 時間のため、トークンはその範囲内でリフレッシュする必要がある
learned(passport-google): refresh token取得には明示的に offline_access スコープが必要
```

subject line は **何を** したかを伝えます。body は **なぜ** そうしたかを伝えます。

---

## 動機

AI コーディングツールは至るところにあります。その出力への信頼はそうではありません。このギャップの正体は文脈(context)です。

AI コーディングのセッションは毎回 3 つの成果を生み出します:

1. **コード変更** — git にコミットされ、保存されます。
2. **意思決定** — どのアプローチが選ばれ、何が却下され、どんな制約が見つかったか。**失われます。**
3. **理解** — システムがどう動いているか、なぜそうなっているかについての深い把握。**失われます。**

会話ウィンドウが閉じた瞬間、そのセッションが生んだ価値の 2/3 は蒸発します。コミット履歴は、あらゆる AI コーディングツールが最初から参照できる唯一の文脈ソースです。それにも関わらず、標準的な AI 生成のコミット body は diff に既に書いてあることを繰り返しているに過ぎません。エージェントが diff から読み取れないのは、なぜそのアプローチが選ばれたか、どんな制約がそれを形作ったか、何が試されて却下されたかです。

git はブランチ、diff、履歴を追跡します。git が追跡しないただ一つのもの、それが「理由」です。コミット body はずっと前からそのために空けられていた場所です。

### 消えていく文脈

エージェントは、あなたが前のセッションで試して却下したアプローチを提案してきます。しかし、それを却下した理由は会話ウィンドウとともに死にました。エージェントは、存在を知りようがない制約に反するきれいな実装を書き、失敗して初めて気付きます。3 ヶ月後、別のセッションが恣意的に見えるパターンをコード内に見つけます。恣意的ではなかったのですが、その理由は、もう存在しない会話の中にあったのです。

同じ問題の 3 つの形です。AI コーディングのセッションはコードと並んで意思決定と理解を生み出しますが、git には生き残るのはコードだけです。

### エージェントが復元できるもの、できないもの

AI コーディングの品質を左右する文脈にはいくつかのカテゴリがあります。その大半はコードベースからリバースエンジニアリングできます。アーキテクチャ、コードパターン、テスト戦略、命名規約。コードを読めるエージェントなら、これらは把握できます。

しかしリバースエンジニアリングできないカテゴリが 2 つあります。**何をしようとしていたか(intent)** と **すでに何を試したか(history)** です。意図と歴史的文脈 — なされた意思決定、却下された代替案、発見された制約、得られた学び — は、人間の記憶と消えていく会話の中にしか存在しません。

Contextual commits はまさにこの 2 つを捕捉します。最も興味深い情報だからではなく、放っておけば永久に失われる情報だからです。

### 複利

最初の contextual commit は将来の再探索を 1 回分節約します。100 回目になれば、新しいセッションを開始するエージェントは、全ての寄稿者による全ての過去セッションから、全ての意思決定・却下・制約・学びを継承できます。

これはあなたがメンテナンスするドキュメントではありません。コードをコミットする副産物として蓄積されていく追記専用(append-only)の歴史です。更新しなければならないファイルも、書き換え続ける wiki ページも、マージコンフリクトもありません。あるのは git だけです。

---

## 規約

### 書式

contextual commit はコミット body に構造化された文脈を持たせます。subject line は標準的な [Conventional Commit](https://www.conventionalcommits.org/en/v1.0.0/) です。body はそれを型付きの action line で拡張します:

```
<type>(<scope>): <description>

<action-type>(<scope>): <content>
<action-type>(<scope>): <content>
```

**scope** は人間が読める形のラベル — ドメイン領域、モジュール、概念です。`auth`、`payment-flow`、`api-contracts`、`session-store` など、プロジェクトで自然に使われている語彙をそのまま使ってください。

### Action type


| Type                | 捕捉する情報            | 例                                                             |
| ------------------- | ----------------- | ------------------------------------------------------------- |
| `intent(scope)`     | ユーザーが何を求めたか・なぜか   | `intent(notifications): イベント単位ではなくメールをバッチ配信する`                |
| `decision(scope)`   | 代替案がある中で何を選んだか    | `decision(queue): マネージドなスケーリングのため RabbitMQ ではなく SQS を採用`      |
| `rejected(scope)`   | 検討されて捨てられたもの、理由つき | `rejected(queue): RabbitMQ — 自前運用のインフラが必要`                    |
| `constraint(scope)` | アプローチを形作ったハードリミット | `constraint(api): ペイロード最大 5MB、タイムアウト 30 秒`                    |
| `learned(scope)`    | 将来のミスを防ぐ、発見された事実  | `learned(stripe): presentment currency ≠ settlement currency` |


5 つの型。それぞれが他の型ではカバーできない signal を捕捉します。それぞれが、新しいセッションを開始するエージェントにとってすぐに役立ちます。

- `intent` — ユーザーが達成しようとしていること。これがないと、エージェントはコードから目的をリバースエンジニアリングするはめになります。
- `decision` — どのアプローチを選んだか。これがないと、エージェントはパターンが意図的なのか偶然なのかを判別できません。
- `rejected` — 何を試して捨てたか。これがないと、エージェントは同じ袋小路を再探索します。最も価値の高い型です。
- `constraint` — 実装上のハードリミット。これがないと、エージェントは失敗することで制約を発見することになります。
- `learned` — API の癖、非自明な挙動、ドキュメントの抜け。これがないと、エージェントは同じ落とし穴を再発見するのに時間を浪費します。constraint とは別物です。constraint は守るべき境界であり、learned は避けるべき罠です。

### 設計原則

- **拡張する、壊さない。** subject line は標準的な Conventional Commit です。既存の全ツール(commitlint、semantic-release、チェンジログ生成ツール)はそのまま動きます。
- **エージェントネイティブで、人間にも読める。** エージェントがプログラムでパース・クエリできるように設計されています。人間への恩恵は副産物です。`git log` は即座に有益な文脈を表示しますが、書式は機械消費向けに最適化されています。
- **diff が示せない signal。** 全ての action line は、コード変更にはまだ見えていない情報を運ばなければなりません。diff が説明していることを繰り返してはいけません。これが品質面の核となるルールです。
- **追加するだけ、強制しない。** 当てはまる action type だけ使ってください。typo 修正には 0 行の action line で十分です。大きなリファクタリングには 10 行入ることもあります。
- **ゼロインフラ。** データベース、外部サービス、設定ファイル、CI ステップは一切不要です。規約は git コミット body に住みます — ソフトウェア開発において最も普遍的、可搬的、永続的なストレージです。
- **既定でクエリ可能。** `git log --all --grep="rejected(auth"` で、履歴全体から却下された認証アプローチを即座に見つけられます。シンプルな正規表現で任意のコミット範囲から action line を抽出できます。
- **捕捉するだけで、規定しない。** action line は時点のイベントです — 特定のセッション中に発見・決定・却下されたことの記録です。将来のコミットに対する立ち位置のルールではありません。蓄積された歴史から規約を導出したり、パターンを強制したりするのは、利用側のツールの仕事です。規約は忠実に捕捉するだけ、解釈は下流の関心事です。

番号付きルールと ABNF 文法を含む公式仕様は [SPEC.md](SPEC.md) を参照してください。

---

## 例

### rejected 行が仕事をするとき

```
feat(search): 商品カタログに全文検索を追加

intent(search): LIKE クエリを置き換える — 5 万件を超えると遅すぎる
decision(search-engine): Elasticsearch ではなく Postgres の全文検索を採用
rejected(search-engine): Elasticsearch — 現状の規模には運用負荷が重すぎる。50 万件到達時に再検討
rejected(search-engine): Algolia — スケール時のコストが過大、ベンダーロックインの懸念
constraint(search-index): 全カタログのインデックス再構築に約 8 分。ピーク外で実行する必要あり
```

将来の再探索を 2 件防止。「なぜ Elasticsearch じゃないの?」と誰に聞かれても、ミーティングなしで答えが返ります。

### learned 行が仕事をするとき

```
feat(exports): ダッシュボードデータから PDF レポートを生成

decision(pdf-engine): pdfkit ではなく Puppeteer を採用 — デザインチームはピクセルパーフェクトな HTML/CSS レンダリングを要求
learned(puppeteer): Docker では --no-sandbox が必要。sandbox モードだとコンテナが黙って落ちる
learned(puppeteer): load ではなく networkidle0 で待つこと。遅延 JS のあるページで load だとタイムアウトする
constraint(exports): PDF 生成は約 3 秒ブロックする。インラインではなくバックグラウンドジョブとして実行すること
```

`learned` 一行ごとに、誰かが再発見に費やさなくて済む 2 時間があります。

### 一行で十分なとき

```
fix(checkout): ダブルクリック送信による注文重複を防止

rejected(checkout): debounce — 300ms の遅延は低速回線で壊れているように感じる
```

修正内容は diff から自明です。記録する価値があるのは、自明な代替案が却下された理由だけです。

---

些末なコミット — 依存関係のバージョン上げ、typo 修正、フォーマット修正 — には action line は 1 行も不要です。クリーンな Conventional Commits の subject line は、捏造された文脈よりも常に優れています。

---

## 現実的な普及の道筋

この規約は AI コーディングツールがネイティブに実装するためのスペックです。

目指すのは、コーディングエージェント(Claude Code、OpenCode、Codex、Gemini CLI など)と IDE(Cursor、Windsurf など)がデフォルトでこの規約に従うことです。現在生成されている noise (diff を読めば分かる冗長な情報) は、signal に置き換わります。エージェントが diff を言い換えるのをやめて理由の保存を始めた瞬間、全ての開発者が恩恵を受けます。問題は普遍的で、解決は個人の規律ではなくツールを通じてスケールします。

以下のリファレンス実装は、今日使うための橋渡しです — ネイティブな採用が進む間、誰でもインストールできるエージェントスキルです。本来の普及の道筋は、ツールメーカーがこれをビルトインで実装することです。

採用は漸進的です — 新しいコミットは action line を持ち、古いコミットはそのままです。文脈は前に向かって蓄積していきます。

---

## リファレンス実装

このリポジトリには [Agent Skills](https://agentskills.io) オープン標準に準拠した、エージェント非依存のファイルが 2 つ含まれます。互換性のあるどのエージェント — Claude Code、GitHub Copilot、Cursor、Gemini CLI、その他 [26+](https://agentskills.io) — でも動作します。

### クイックスタート

```bash
npx skills add jnst/contextual-commits
```

エージェントを自動検出し、正しいディレクトリにスキルをインストールします。これだけです。

### 含まれるもの


| File                                                                     | 内容                                                                                      |
| ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------- |
| `[skills/contextual-commit/SKILL.md](skills/contextual-commit/SKILL.md)` | contextual commit 書式をエージェントに教えます。コミット時に自動起動し、セッションで起きたことに基づいて構造化された action line を生成します。 |
| `[skills/recall/SKILL.md](skills/recall/SKILL.md)`                       | `/recall` — contextual commit 履歴から開発文脈を再構成します。                                          |


### 使い方

**Contextual commit を書く** — 普通にコミットするだけです。エージェントがコミットを書こうとしたときにスキルが自動で発動し、セッションの会話を元に action line を生成します。

**文脈を呼び出す:**

```
/recall                         現在のブランチの完全なセッションブリーフィング
/recall <scope>                 scope にマッチする全 action line (前方一致)
/recall <action>(<scope>)       scope 内の特定 action type
```

`/recall` を明示的に呼び出すこともできますが、エージェントに任せても構いません — スキルなので、行動前に文脈を読むエージェントなら自然に手を伸ばします。

## License

MIT