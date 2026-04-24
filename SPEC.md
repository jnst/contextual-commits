# Contextual Commits 仕様

**バージョン: v0.1.0**

## 概要

Contextual Commits は、git コミット body に意思決定の記録を埋め込むための規約です。Conventional Commits を拡張し、変更の背後にある意図・決定・制約・学びを捕捉する、型付きでスコープ付きの **action line** を追加します — diff だけでは見えない理由を捉えるためのものです。

この規約が存在する理由は、AI コーディングツールが 1 セッションで 3 つのアウトプット(コード変更、意思決定、理解)を生み出すにもかかわらず、git に生き残るのはコードだけだからです。意思決定と理解は会話ウィンドウが閉じた瞬間に蒸発します。Contextual Commits はその signal (diff では示せない有益な情報) をコミット body に保存します。そこは変更と共に移動でき、任意のツールからクエリ可能な場所です。

この仕様は互換性を明示するためにバージョン番号を使います。v0.1.0 で妥当なコミットは、v0.x.y のどのリリースでも妥当であり続けます。破壊的変更が発生した場合(ある場合)はメジャーバージョンを上げます。

## 用語

この文書におけるキーワード "MUST"、"MUST NOT"、"REQUIRED"、"SHALL"、"SHALL NOT"、"SHOULD"、"SHOULD NOT"、"RECOMMENDED"、"MAY"、"OPTIONAL" は、[RFC 2119](https://www.ietf.org/rfc/rfc2119.txt) に記述された通りに解釈されます。

**subject line** — コミットメッセージの最初の行。妥当な Conventional Commit でなければならない (MUST)。

**commit body** — subject line に続く最初の空行より後の全て。

**action line** — コミット body 内の 1 行で、`action-type(scope): description` の書式に従うもの。1 単位の reasoning を捕捉します。

**action type** — action line のキーワード・プレフィックス。`intent`、`decision`、`rejected`、`constraint`、`learned` のいずれか。

**scope** — action line が関連するドメイン領域・モジュール・概念を識別する、人間が読めるラベル。

## 構造

```
type(scope): subject line          ← Conventional Commit (必須)
                                   ← 空行
action-type(scope): description    ← action line (任意、0 行以上)
action-type(scope): description
                                   ← 自由記述 (任意)
Signed-off-by: ...                 ← Conventional Commits footer (任意)
```

subject line は Conventional Commits 仕様によって統治されます。この仕様は action line と、subject line および commit body との関係を統治します。

## 仕様

### Conventional Commits との互換性

1. subject line は妥当な [Conventional Commit](https://www.conventionalcommits.org/en/v1.0.0/) でなければならない (MUST)。
2. action line は commit body 内、subject line と body を区切る空行より後に配置しなければならない (MUST)。
3. action line を subject line 内に配置してはならない (MUST NOT)。

### action line の書式

1. 各 action line は次の書式に従わなければならない (MUST): `action-type(scope): description`。
2. `action-type` は次のいずれかでなければならない (MUST): `intent`、`decision`、`rejected`、`constraint`、`learned`。
3. `scope` は空であってはならず (MUST NOT)、小文字英数字とハイフンで構成されなければならない (MUST)。ハイフンで始まったり終わったりしてはならない (MUST NOT)。
4. `scope` はプロジェクトの語彙で意味のある、人間が読めるドメイン・モジュール・概念ラベルであるべきである (SHOULD)。
5. 同じ概念を指すコミット間で scope は一貫しているべきである (SHOULD)。あるコミットで `auth` と呼んでいるなら、次のコミットで `authentication` と呼ぶべきではない (SHOULD NOT)。
6. `description` は空であってはならず (MUST NOT)、コロンと単一スペース(`:` )に続かなければならない (MUST)。

### 任意性と signal の質

1. action line は任意 (OPTIONAL)。action line が 0 行のコミットも妥当な contextual commit です。些末な変更(typo 修正、依存関係のバージョン上げ、フォーマット修正)では action line を省略するべきである (SHOULD)。
2. コミットは任意の数の action line を持ってもよい (MAY)。各 action line は diff では示せない signal を運ばなければならない (MUST)。
3. action line はコード変更ですでに見えている情報を再記述してはならない (MUST NOT)。diff が説明しているなら、action line はそれを繰り返すべきではない (SHOULD NOT)。

### 特別なルール

1. `rejected` の action line は却下の理由を含まなければならない (MUST)。理由なき却下は価値がありません — 次のセッションは同じアプローチを再提案してしまいます。
2. action line は作者が根拠を持たない文脈を捏造してはならない (MUST NOT)。会話の文脈が得られない場合(前のセッションの出力、外部からの変更など)、`decision` のみを、明確な技術的選択が diff から読み取れるときに限り推論して記述してもよい (MAY)。`intent`、`rejected`、`constraint`、`learned` は根拠なしに捏造してはならない (MUST NOT)。

### Conventional Commits との相互作用

1. action line 内の scope と subject line 内の scope は独立した名前空間であり、異なっていてもよい (MAY)。subject line の scope は Conventional Commits の規約に従い、action line の scope はこの仕様に従います。
2. 自由記述テキストは action line と並んで commit body 内に現れてもよい (MAY)。各 action line はちょうど 1 行を占めなければならない (MUST)。
3. Conventional Commits の footer (例: `Signed-off-by`、`BREAKING CHANGE`) は action line の後に現れてもよい (MAY)。Conventional Commits 仕様の trailer に関する規定に従います。

## ABNF 文法

以下の文法は補足です。上記の散文ルールと衝突する場合は、散文ルールが優先されます。

```abnf
contextual-body = *( action-line / free-text / blank-line ) [ CC-footer ]

action-line     = action-type "(" scope ")" ":" SP description LF
action-type     = "intent" / "decision" / "rejected" / "constraint" / "learned"
scope           = scope-char *( scope-char / "-" scope-char )
scope-char      = %x61-7A / DIGIT   ; lowercase a-z, 0-9
description     = 1*( %x20-7E / UTF8-non-ascii )

free-text       = 1*( %x20-7E / UTF8-non-ascii ) LF
blank-line      = LF
CC-footer       = <as defined by Conventional Commits v1.0.0>
```

## 例

### 1. 最小 — action line なし (ルール 10)

```
fix(button): モバイルビューポートでの位置ずれを修正
```

Conventional Commit の subject line で十分です。action line は不要です。

### 2. 単一の action line

```
feat(notifications): 週次サマリのメールダイジェストを追加

intent(notifications): イベント単位のメールではなく、バッチ通知を望む声が多い
```

1 行の intent が、subject と diff では示せない動機を捕捉しています。

### 3. 複数の action type (ルール 4-5、11)

```
feat(auth): Google OAuth プロバイダーを実装

intent(auth): Google を皮切りにソーシャルログインを導入。続いて GitHub、Apple を予定
decision(oauth-library): multi-provider 対応の柔軟性のため auth0-sdk ではなく passport.js を採用
rejected(oauth-library): auth0-sdk — 独自のセッションモデルに縛られ、redis ストアと両立しない
constraint(callback-routes): 既存規約に従い /api/auth/callback/:provider パターンに準拠する必要がある
learned(passport-google): refresh token取得には明示的に offline_access スコープが必要
```

5 種類の action type それぞれが diff で示せない signal を運んでいます。action line の scope (`auth`、`oauth-library`、`callback-routes`、`passport-google`) は subject line の scope (`auth`) と異なります。これはルール 15 に従ったものです。

### 4. 実装途中での方針転換 (ルール 11、13)

```
refactor(auth): セッションベースから JWT トークンに切り替え

intent(auth): 元のセッション方式は redis cluster 構成と両立しない
rejected(auth-sessions): redis cluster は passport が必要とするsession stickinessに未対応
decision(auth-tokens): JWT + 短い有効期限 + refresh tokenのパターン
learned(redis-cluster): session affinityは LB レベルでの sticky session を要求する — 侵襲的すぎる
```

`rejected` 行には理由が含まれています(ルール 13)。`intent` 行は方針転換を捕捉しています — 元のアプローチが失敗したことが、この refactor の動機です。

### 5. diff のみからの推論 — 会話の文脈なし (ルール 14)

```
refactor(http-client): axios をネイティブ fetch に置き換え

decision(http-client): 依存関係ゼロの HTTP 通信のため axios からネイティブ fetch に移行
```

この変更について作者は会話の文脈を持っていません。技術的選択が diff から読み取れるため、`decision` のみが記述されています。`intent`、`rejected`、`constraint`、`learned` は捏造されていません。

## 設計原則

これらの原則は規約の設計を導くものであり、実装を導くものでもあるべきである (SHOULD)。

1. **拡張する、壊さない。** subject line は標準的な Conventional Commit でなければならない (MUST)。既存の全ツール(commitlint、semantic-release、チェンジログ生成ツール)はそのまま動作しなければならない (MUST)。
2. **エージェントネイティブで、人間にも読める。** 書式はエージェントがプログラムでパース・クエリできるように設計されています。人間への恩恵は副産物です — `git log` は即座に有益な文脈を表示しますが、書式は機械消費向けに最適化されています。
3. **diff が示せない signal。** 全ての action line はコード変更ですでに見えていない情報を運ばなければならない (MUST)。これが核となる品質ルールです。
4. **追加するだけ、強制しない。** 実装は当てはまる action type のみを使うべきである (SHOULD)。typo 修正には action line は 0 行で十分です。大きなリファクタリングには 10 行入ることもあります。
5. **ゼロインフラ。** この規約はデータベース、外部サービス、設定ファイル、CI ステップを要求してはならない (MUST NOT)。規約は git コミット body に住みます — ソフトウェア開発において最も普遍的、可搬的、永続的なストレージです。
6. **既定でクエリ可能。** 書式はシンプルなテキストツールでパース可能でなければならない (MUST)。`git log --all --grep="rejected(auth"` は、履歴全体から却下された認証アプローチを発見できなければならない (MUST)。
7. **捕捉するだけで、規定しない。** action line は時点のイベントです — 特定のセッション中に発見・決定・却下されたことの記録です。将来のコミットに対する立ち位置のルールや指令ではありません。蓄積された歴史から規約を導出したり、パターンを強制したり、ガイダンスを合成したりするのは、利用側のツールの責任です。規約は忠実に捕捉するだけ、解釈と強制は下流の関心事です。

