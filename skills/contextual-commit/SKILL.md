---

## name: contextual-commit

description: >-
  コード変更と並んで、意図・意思決定・制約を捕捉する contextual commit を書く。
  コミット時、タスク完了時、あるいはユーザーからコミットを依頼されたときに使う。
  Conventional Commits を拡張し、構造化された action line を commit body に
  追加することで、「何が」変わったかだけでなく「なぜ」そのコードが書かれたかを
  保存する。
license: MIT

# Contextual Commits

あなたは、diff だけでは示せない開発上の理由 — 意図、意思決定、制約、学び — をコミット body に持たせるコミットを書きます。

## あなたが解決する問題

通常のコミットは WHAT(何が変わったか)を保存します。diff もそれを示します。失われるのは WHY — ユーザーが何を依頼したか、どの代替案が検討されたか、どんな制約が実装を形作ったか、途中で何が学ばれたか、です。この文脈はセッションが終わると蒸発します。あなたはそれを防ぎます。

## コミット書式

subject line は標準的な Conventional Commit です。body には **action line** — 型付きでスコープ付きの、理由を捕捉するエントリ — が入ります。

```
type(scope): subject line (標準的な Conventional Commit)

action-type(scope): 理由や文脈の記述
action-type(scope): もう 1 行
```

### Subject Line

Conventional Commits に厳密に従ってください。ここは何も変わりません:

- `feat(auth): Google OAuth プロバイダーを実装`
- `fix(payments): 通貨の丸め端数ケースに対処`
- `refactor(notifications): ダイジェストスケジュールのロジックを抽出`

### Action Line

body の各行は次の書式に従います: `action-type(scope): description`

**scope** は人間が読める概念ラベル — ドメイン領域、モジュール、関心事です。例: `auth`、`payment-flow`、`oauth-library`、`session-store`、`api-contracts`。プロジェクトの語彙で意味のあるものを使ってください。同じ概念を指すなら、コミット間で scope を一貫させてください。

## Action Type

当てはまる型のみ使ってください。大半のコミットは 1〜3 行の action line で十分です。noise (diff を読めば分かる冗長な情報) で埋めないでください。

### `intent(scope): ...`

ユーザーが達成したかったことと、その理由。あなたの解釈ではなく、人間の声を捕捉します。

- `intent(auth): Google を皮切りにソーシャルログインを導入。続いて GitHub、Apple を予定`
- `intent(notifications): イベント単位のメールではなくバッチ通知を望む声が多い`
- `intent(payment-flow): エンタープライズ顧客向けに USD と並んで EUR、GBP をサポートする必要がある`

**When to use:** ほとんどの機能開発、目的のあるリファクタリング、subject line から動機が自明でない変更全般。

### `decision(scope): ...`

代替案がある中で何のアプローチを選んだか。簡潔に理由を添えて。

- `decision(oauth-library): multi-provider 対応の柔軟性のため auth0-sdk ではなく passport.js を採用`
- `decision(digest-schedule): 毎日ではなく月曜 9 時週次 — ユーザーリサーチに合致`
- `decision(currency-handling): アカウント単位のデフォルトではなくトランザクション単位の通貨指定`

**When to use:** 選択肢を評価したとき。代替案の無い自明な選択はスキップ。

### `rejected(scope): ...`

検討して明示的に捨てたもの。理由つき。これが最も価値の高い action type です — 将来のセッションが同じ提案を繰り返すのを防ぎます。

- `rejected(oauth-library): auth0-sdk — 独自のセッションモデルに縛られ、redis ストアと両立しない`
- `rejected(currency-handling): アカウント単位のデフォルト — マーケットプレイス出品者には制約が強すぎる`
- `rejected(money-library): accounting.js — サブ単位(セント)の演算をサポートしない`

**When to use:** あなたまたはユーザーが意味のある代替案を検討した上で採用を見送ったときは、毎回。必ず理由を添えること。

### `constraint(scope): ...`

アプローチを形作った、実装中に発見したハードリミット・依存関係・境界。

- `constraint(callback-routes): 既存規約に従い /api/auth/callback/:provider パターンに準拠する必要がある`
- `constraint(stripe-integration): PaymentIntent 作成時に通貨を指定する必要があり、後から変更不可`
- `constraint(session-store): redis の TTL は 24 時間のため、トークンはその範囲内でリフレッシュが必要`

**When to use:** 自明でない制約が実装に影響したとき。次にここを触る人が知る必要のあること。

### `learned(scope): ...`

実装中に発見したもので、将来のセッションで時間を節約できる事実。API の癖、ドキュメント外の挙動、パフォーマンス特性。

- `learned(passport-google): refresh token取得には offline_access スコープが明示的に必要。クイックスタートに記述なし`
- `learned(stripe-multicurrency): presentment currency と settlement currency は別概念`
- `learned(exchange-rates): Stripe が変換を処理する — 独自のレートを保持してはならない`

**When to use:** 「これ、最初に知っていたかった」という瞬間。ライブラリの罠、API の想定外の振る舞い、非自明な挙動。

## コミットを書く前に

コミットの範囲を決めた上で、action line を組み立てます:

1. **まずステージ済みの変更を確認する** — `git diff --cached --stat` を実行。
  - **ステージ済み変更がある場合:** それがコミット範囲です。未ステージやトラック外のファイルは考慮しません — ユーザーはステージングという形でこのコミットに何を含めるかをすでに表明しています。
  - **何もステージされていない場合:** 全ての未ステージ変更とトラック外ファイルを候補とします。セッションの文脈と diff から、何をステージしてコミットするか判断します。
2. **自分が文脈を持っている変更を特定する** — 今回の会話中にあなたが生成した、議論した、あるいは理由を観察した変更。
3. **文脈を持たない変更を特定する** — 前のセッション、別のエージェント、会話外の手動編集によるファイルや変更。
4. **それに応じて action line を書く:**
  - 文脈を持つ変更: セッションの知識から完全な action line を書く。
  - 文脈を持たない変更: 下の「会話の文脈がないとき」のルールを適用し、diff が示すことだけを書く。

コミットメッセージはコミット範囲内の**全ての**変更を説明しなければなりません。自分が書いた変更だけに限りません。自分が生成しなかった変更を無視するのは、薄い action line を書くよりも悪い選択です。

## 例

### 単純な修正 — action line は不要

```
fix(button): モバイルビューポートでの位置ずれを修正
```

Conventional Commit の subject line で十分です。noise を足さないこと。

### 中程度の機能

```
feat(notifications): 週次サマリのメールダイジェストを追加

intent(notifications): イベント単位のメールではなくバッチ通知を望む声が多い
decision(digest-schedule): 月曜 9 時週次 — ユーザーリサーチのフィードバックに合致
constraint(email-provider): SendGrid のバッチ API は 1 コール 1000 宛先まで
```

### 複雑なアーキテクチャ変更

```
refactor(payments): 単一通貨からmulti-currency 対応に移行

intent(payments): エンタープライズ顧客が USD と並んで EUR、GBP を必要としている
intent(payment-architecture): 既存の USD フローを変えず、後方互換性を維持する必要がある
decision(currency-handling): アカウント単位のデフォルトではなくトランザクション単位の通貨指定
rejected(currency-handling): アカウント単位のデフォルト — マーケットプレイス出品者には制約が強すぎる
rejected(money-library): accounting.js — サブ単位演算をサポートしない。代わりに currency.js を採用
constraint(stripe-integration): Stripe は PaymentIntent 作成時に通貨を確定し、後から変更不可
constraint(database-migration): 既存の amount カラムは置き換えではなく、通貨カラムを併設する必要がある
learned(stripe-multicurrency): presentment currency と settlement currency は別の Stripe 概念
learned(exchange-rates): Stripe が変換を処理する。独自のレートを保持してはならない
```

### 実装途中の方針転換

intent が作業中に変わった場合、その転換が起きたコミットで捕捉します:

```
refactor(auth): セッションベースから JWT トークンに切り替え

intent(auth): 元のセッション方式は redis cluster 構成と両立しない
rejected(auth-sessions): redis cluster は passport のセッションで必要なsession stickinessに未対応
decision(auth-tokens): JWT + 短い有効期限 + refresh tokenのパターン
learned(redis-cluster): session affinityは LB レベルでの sticky session を要求する — 侵襲的すぎる
```

## 会話の文脈がないとき

ステージ済みの変更に、あなたがこのセッションで生成していないもの — 前のセッション出力、別のエージェントの変更、貼り付けコード、外部生成ファイル、手動編集 — が含まれることがあります。理由のトレイルを持たない変更については:

**diff で明確に証拠立てられることについてのみ action line を書いてください。** 観察できない意図や制約を推測してはいけません。

diff 単体から推論できること:

- `decision(scope)` — 明確な技術的選択が読み取れる場合(新規依存の追加、パターンの採用、ライブラリの切り替えなど)。例: `decision(http-client): axios からネイティブ fetch に切り替え` は diff から読み取れます。

推論できない — 捏造しないこと:

- `intent(scope)` — なぜ変更したかは diff にはありません。diff が示すことを言い換えるだけにならないように。
- `rejected(scope)` — 選ばれなかったものは、選ばれてコミットされたものからは見えません。
- `constraint(scope)` — ハードリミットはコード変更にほぼ現れません。
- `learned(scope)` — 学びはアウトプットではなくプロセスから生まれます。

**クリーンな Conventional Commit の subject line だけで action line 無しのほうが、捏造された文脈より常に優れています。**

## Git ワークフロー

Contextual commit は標準的な git ワークフロー全てとそのまま動作します。特別な扱いは不要です。

- **通常のマージ:** コミット body はそのまま保存されます。
- **スカッシュマージ:** 全コミット body がスカッシュコミット body に連結されます。結果として時系列順の型付き・スコープ付き action line のトレイルが得られ、エージェントは問題なくパース・フィルタ・グループ化できます。
- **リベース・チェリーピック:** コミット body はそのまま保存されます。

## ルール

1. **subject line は Conventional Commit。** 既存規約やツールを壊さない。
2. **action line は body のみ。** subject line には絶対入れない。
3. **signal (diff では示せない有益な情報) を運ぶ action line のみ書く。** diff が既に説明しているなら繰り返さない。決めること・却下すること・発見することが何もなかったなら、action line は 0 行。
4. **簡潔かつ完全に。** 各 action line は明確な一文。長さの人工的な制限はないが、エッセイを書かないこと。
5. **プロジェクト内で scope を一貫させる。** あるコミットで `auth` と呼んだなら、次で `authentication` と呼ばない。
6. `**intent` 行はユーザーの言葉で意図を捕捉する。** あなたの実装要約ではなく、人間が依頼したそのものを反映する。
7. `**rejected` 行は必ず理由を説明する。** 理由のない却下は無意味 — 次のエージェントがまた提案してきます。
8. **些末なコミットには action line を発明しない。** typo 修正、依存関係のバージョン上げ、フォーマット修正 — Conventional Commit の subject line で十分。
9. **持っていない文脈を捏造しない。** 推論プロセスに居合わせなかったなら、居合わせたふりをしない。上の「会話の文脈がないとき」を参照。

