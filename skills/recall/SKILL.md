---

## name: recall

description: >-
  contextual commit から現在の開発文脈を再構成してナラティブとして提示する。
  セッション開始時、作業を再開するとき、ブランチを切り替えたときに実行する。
  現状を簡潔かつ会話的に要約する。
license: MIT

# Context Recall

contextual commit の履歴から開発の物語を再構成し、自然なブリーフィングとして提示します。

## 引数の検出

`recall` がどう呼び出されたかを確認します:

- **引数なし** — デフォルトモードを実行(下記の完全ブランチ/セッションブリーフィング)。
- **単語のみ**(例: `recall auth`) — scope クエリとして扱う。**Scope Query** にジャンプ。
- `**word(word)` パターン**(例: `recall rejected(auth)`) — action+scope クエリとして扱う。**Action+Scope Query** にジャンプ。

---

## デフォルトモード(引数なし)

### ステップ 1: ブランチ状態の検出

作業状態を判定します:

```bash
CURRENT_BRANCH=$(git branch --show-current)
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "main")

# 単にリポジトリのデフォルトブランチではなく、実際の親ブランチを見つける。
# まず upstream tracking ブランチを試す(高速経路)。
BASE_BRANCH=$(git rev-parse --abbrev-ref @{upstream} 2>/dev/null | sed 's|^origin/||')

# upstream がない場合、コミット距離で最も近いローカルブランチを探す。
if [ -z "$BASE_BRANCH" ]; then
    BASE_BRANCH=$(git for-each-ref --format='%(refname:short)' refs/heads/ | while read branch; do
        [ "$branch" = "$CURRENT_BRANCH" ] && continue
        echo "$(git log --oneline "$branch..$CURRENT_BRANCH" 2>/dev/null | wc -l | tr -d ' ') $branch"
    done | sort -n | head -1 | awk '{print $2}')
fi

# 最終フォールバックはデフォルトブランチ。
BASE_BRANCH=${BASE_BRANCH:-$DEFAULT_BRANCH}

UNSTAGED=$(git diff --stat)
STAGED=$(git diff --cached --stat)
BRANCH_COMMITS=$(git log ${BASE_BRANCH}..HEAD --oneline 2>/dev/null | wc -l | tr -d ' ')
```

`DEFAULT_BRANCH` はリポジトリの主要ブランチを識別します(シナリオ C/D の検出用)。`BASE_BRANCH` は最も近い祖先ブランチを識別します — feature ブランチがデフォルト以外のブランチから切られている場合(例: `develop` から切られた feature ブランチ)には両者が異なります。ブランチ相対のクエリは全て `BASE_BRANCH` を使います。

### ステップ 2: 素材の収集

#### シナリオ A — コミットのある feature ブランチ上

最も情報量の多いシナリオです。以下を収集します:

```bash
# 全ブランチコミットから contextual action line を取得
git log ${BASE_BRANCH}..HEAD --format="%H%n%s%n%b%n---COMMIT_END---"

# 未ステージの変更(今まさに進行中のもの)
git diff --stat
git diff  # 重要な変更については実際の diff を読む

# ステージ済みの変更
git diff --cached --stat
```

#### シナリオ B — まだコミットのない feature ブランチ上

```bash
# 未ステージとステージ済みの変更のみ
git diff --stat
git diff --cached --stat

# 親ブランチの直近数コミットをプロジェクト文脈として取得
git log ${BASE_BRANCH} -10 --format="%H%n%s%n%b%n---COMMIT_END---"
```

#### シナリオ C — デフォルトブランチ上

```bash
# contextual action line を含む直近のコミット履歴
git log -20 --format="%H%n%s%n%b%n---COMMIT_END---"
```

#### シナリオ D — デフォルトブランチ上で未コミットの変更あり

```bash
# C と同じ。加えて未コミットの変更
git log -20 --format="%H%n%s%n%b%n---COMMIT_END---"
git diff --stat
git diff --cached --stat
```

### ステップ 3: action line の抽出

収集したコミット body から、以下にマッチする行を抽出します:

```
^(intent|decision|rejected|constraint|learned)\(
```

コミット単位(時系列を保持)と型単位(合成用)でグループ化します。

### ステップ 4: 出力の合成

**ナラティブの流れよりも signal (diff では示せない有益な情報) の密度を優先。** 出力はコンパクトでスキャン可能、そして全てがコミットと diff が示す内容に根拠を持つものでなければなりません。全ての行が行動可能な情報であるべきです。冗長な装飾も、会話的なパディングも無しで。

#### シナリオ A の場合(コミットのあるブランチ):

ブランチ状態を出力し、作業継続にとって最も重要な順に contextual action line を密度の高いブリーフィングに合成します。

出力例:

```
Branch: feat/google-oauth (4 commits ahead of main, unstaged changes in tests/)

Active intent: Google を皮切りにソーシャルログインを導入。GitHub、Apple が後続予定。
Approach: passport.js + /api/auth/callback/:provider 規約。
Rejected: auth0-sdk — セッションモデルが redis ストアと両立しない。
Constraints:
  - Redis セッション TTL 24h。トークンはこの範囲内でリフレッシュ必須
  - callback routeは既存の :provider パターンに従う必要あり
Learned: passport-google はrefresh token取得に offline_access スコープが明示的に必要。
In progress: callback handlerの結合テスト(未ステージ)。
```

デフォルト以外のブランチから切られている場合:

```
Branch: feat/oauth-refresh (2 commits ahead of develop, no uncommitted changes)

Active intent: OAuth プロバイダーのトークンリフレッシュを実装。
Constraint: develop 上のセッションストア変更と互換を保つ必要がある。
```

優先順:

1. Active intent(何を・なぜ作っているか)
2. 現在のアプローチ(下された decision)
3. 却下されたアプローチ(再探索すべきでないもの — 最重要)
4. 制約(ハード境界)
5. 学び(時間を節約する事実)
6. 進行中の作業(未ステージ/ステージ済みの変更)

ブランチの途中で intent が進化した(方針転換)場合は、転換が見えるように元の intent と現在の intent の両方を示します。

#### シナリオ B の場合(コミットなしのブランチ):

```
Branch: feat/new-feature (0 commits ahead of develop)

このブランチには contextual 履歴がまだありません。
Staged: 2 files (src/auth/provider.ts, src/auth/types.ts)
Unstaged: なし

直近のプロジェクト動向(develop より):
  - Auth: OAuth プロバイダフレームワークがマージ済み、Google 動作中
  - Payments: multi-currency 対応が出荷済み(USD と並んで EUR、GBP)
```

#### シナリオ C の場合(デフォルトブランチ、変更なし):

直近 20 コミットからマージ済み作業を合成します。活動領域ごとにグループ化。広く適用される現行の制約や学びを浮上させます。

```
直近のプロジェクト動向:

Auth: OAuth プロバイダフレームワークがマージ済み。Google 動作中、GitHub と Apple を予定。
  - auth0-sdk を却下(セッションモデルが redis ストアと両立しない)
  - Constraint: redis セッション TTL 24h。トークンはこの範囲内でリフレッシュ必須
Payments: multi-currency 対応が出荷済み(USD、EUR、GBP)。
  - アカウント単位ではなくトランザクション単位の通貨指定
  - Constraint: Stripe は PaymentIntent 作成時に通貨を確定する
  - Learned: Stripe では presentment と settlement は別通貨概念

何に取り組みますか?
```

#### シナリオ D の場合(デフォルトブランチで未コミット変更あり):

シナリオ C と同じ。先頭に未コミット変更を記す。

#### contextual commit が全く存在しない場合

履歴に conventional commit はあるが contextual action line が全くない場合、それでも**存在する情報から有用な出力を出します**:

```
Branch: main (直近履歴に contextual commit が見つかりません)

直近の活動(コミットの subject より):
  - auth: 6 commits (OAuth 実装、ミドルウェア更新) — 2 日前
  - payments: 4 commits (通貨処理、Stripe 統合) — 先週
  - tests: 3 commits — 先週
  - config: 2 commits

最新: feat(auth): Google OAuth プロバイダーを実装

コミット履歴には明示的な intent、decision、constraint が記録されていません。
今後のコミットで contextual-commit skill を使うと、
subject だけでは見えない変更の理由を浮かび上がらせることができます。
```

これは signal の質について正直でありながら、有用な道案内を提供します。skill の採用の提案は自然な次の一歩であり、売り込みではありません。

### ガイドライン

- **会話的よりも密度高く。** 全ての行が情報を運ぶこと。「これまでの経緯は」「お話ししましょう」といった前置きは不要。
- **データに根拠を置く。** action line、コミット subject、diff が実際に示すことだけを報告する。推論・推測・欠落の補完はしない。
- **却下を目立たせる。** 却下されたアプローチは最高価値の signal — 無駄な探索を防ぎます。
- **複数 scope がある場合は scope でグループ化。** 広い履歴を持つデフォルトブランチでは、時系列よりもドメイン領域で整理する。
- **プロンプトで締める。** 「何に取り組みますか?」のような短い問いで終える。簡潔に。
- **データ量に合わせて拡縮する。** contextual commit が 2 件なら出力は 3〜4 行。20 件なら段落にグループ化。薄いデータを長い出力に水増ししない。

---

## Scope Query (`recall <scope>`)

与えられた scope について、リポジトリ履歴全体からターゲットを絞ったクエリを行います。前方一致: `auth` は `auth`、`auth-tokens`、`auth-library` などにマッチします。

### ステップ 1: クエリ

```bash
git log --all --grep="(${SCOPE}" --format="%H%n%s%n%b%n---COMMIT_END---"
```

### ステップ 2: 抽出

収集したコミット body から、以下にマッチする行を抽出します:

```bash
grep -E "^(intent|decision|rejected|constraint|learned)\(${SCOPE}"
```

これにより、scope がクエリ語で始まる全 action line が拾えます。

### ステップ 3: 出力

action type でグループ化し、各グループ内は時系列順に並べます。どのサブ scope が見つかったかを示します。

例(`recall auth`):

```
Scope: auth (also found: auth-tokens, auth-library)

intent:
  - Google を皮切りにソーシャルログインを導入。続いて GitHub、Apple
  - 元のセッション方式は redis cluster 構成と両立しない

decision:
  - multi-provider 対応の柔軟性のため auth0-sdk ではなく passport.js を採用
  - JWT + 短い有効期限 + refresh tokenのパターン

rejected:
  - auth0-sdk — 独自のセッションモデルに縛られ、redis ストアと両立しない
  - auth-sessions — redis cluster は passport のセッションで必要なsession stickinessに未対応

constraint:
  - callback routeは既存規約に従い /api/auth/callback/:provider パターンに準拠する必要がある
  - redis の TTL は 24 時間のため、トークンはその範囲内でリフレッシュする必要がある

learned:
  - passport-google はrefresh token取得に offline_access スコープが明示的に必要
  - redis-cluster のsession affinityは LB レベルでの sticky session を要求する — 侵襲的すぎる
```

マッチが見つからない場合は端的にそう伝え、scope 名の確認を促します。

---

## Action+Scope Query (`recall <action>(<scope>)`)

リポジトリ履歴全体から、特定の scope に対する特定の action type をクエリします。

### ステップ 1: クエリ

```bash
git log --all --grep="${ACTION}(${SCOPE}" --format="%H%n%s%n%b%n---COMMIT_END---"
```

### ステップ 2: 抽出

```bash
grep "^${ACTION}(${SCOPE}"
```

### ステップ 3: 出力

来歴が分かるようコミット subject を添え、時系列のフラットリストとして出力します。

例(`recall rejected(auth)`):

```
rejected(auth) across history:

  - auth0-sdk — 独自のセッションモデルに縛られ、redis ストアと両立しない
    from: feat(auth): Google OAuth プロバイダーを実装

  - auth-sessions — redis cluster は passport のセッションで必要なsession stickinessに未対応
    from: refactor(auth): セッションベースから JWT トークンに切り替え
```

マッチが見つからない場合は端的にそう伝えます。

---

## プロアクティブな使い方

任意の scope 領域で重要な意思決定を下す前に、その scope の `rejected` 行と `constraint` 行をまず確認します:

```bash
git log --all --grep="rejected(${SCOPE}" --format="%b" | grep "^rejected(${SCOPE}"
git log --all --grep="constraint(${SCOPE}" --format="%b" | grep "^constraint(${SCOPE}"
```

あなたが提案しようとしているアプローチについて、過去のコミットが `rejected` 行を記録していた場合、**提案する前に**却下理由をユーザーに提示してください。その却下は今も有効かもしれませんし、状況が変わっているかもしれません — しかしユーザーには完全な文脈の上で判断してもらうべきで、同じ袋小路を再発見させてはいけません。

同様に、既知の境界を侵害しそうなアプローチを提案する前に `constraint` 行を確認します。

このチェックは軽量であり、あなたが作業中のどの scope についても習慣になるべきです。