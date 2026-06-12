# 就業者情報詳細 レビュー結果

## 指摘事項

### 1. 更新API未実装により設計イベント「更新ボタン押下」を満たせていない
- 優先度: High
- 種別: 実装不備
- 対象: functions/persons/updatePerson/app.py（未存在） / frontend/src/pages/workers/WorkerDetail.jsx
- 根拠:
  - 設計書: [spec/基本設計/機能設計/060.就業者情報/就業者情報詳細.md](spec/基本設計/機能設計/060.就業者情報/就業者情報詳細.md#L109) に「更新ボタン押下」イベントが定義されている。
  - 実装: [frontend/src/pages/workers/WorkerDetail.jsx](frontend/src/pages/workers/WorkerDetail.jsx#L271) の更新処理は「準備中」アラートで終了し、[frontend/src/pages/workers/WorkerDetail.jsx](frontend/src/pages/workers/WorkerDetail.jsx#L31) で参照する updatePerson のバックエンド実装が存在しない。
- 指摘内容:
  - 画面仕様で必須の更新機能が未実装で、就業者情報詳細の主要ユースケースが成立していない。
- 期待されるテスト:
  - 更新正常系（履歴編集モード）
  - 変更不可属性を更新しようとした場合の拒否
  - 権限不足時の403
  - 依存先DB障害時のエラー応答

### 2. 共通認証取得がハードコード化され、権限制御が実質無効
- 優先度: High
- 種別: 実装不備
- 対象: shared/core/common.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/060.就業者情報/就業者情報詳細.md](spec/基本設計/機能設計/060.就業者情報/就業者情報詳細.md#L174) ではマスターデータ管理者のみ履歴操作可など、権限制御が前提。
  - 実装: [shared/core/common.py](shared/core/common.py#L33) の get_current_user が固定ユーザーを返却し、実トークン解析がコメントアウトされている。
- 指摘内容:
  - API全体で認証・認可の前提が崩れており、権限判定の信頼性が担保できない。
- 期待されるテスト:
  - 有効/無効トークンの認証テスト
  - 権限別（MASTER_DATA_ADMIN/TENANT_ADMIN/NORMAL）アクセス制御テスト

### 3. deleteUserのリクエストボディ解析がAPI Gateway標準形式に非対応
- 優先度: High
- 種別: 実装不備
- 対象: functions/users/deleteUser/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/060.就業者情報/就業者情報詳細.md](spec/基本設計/機能設計/060.就業者情報/就業者情報詳細.md#L203) のサーバーサイド処理は要求パラメータを正しく受け取り検証する前提。
  - 実装: [functions/users/deleteUser/app.py](functions/users/deleteUser/app.py#L49) で body が dict の場合のみ読む実装になっており、文字列JSONの場合は空扱いになり userId 必須エラーとなる。
- 指摘内容:
  - 実運用イベントで body が文字列の場合、正当な削除要求でも 400 が返る可能性が高い。
- 期待されるテスト:
  - body が文字列JSONの正常削除
  - body が dict の正常削除
  - 不正JSON時の400系エラー

### 4. 就業者詳細APIの単体テストが未整備（getPersonはスキップ、create/getPersonAttributeは未作成）
- 優先度: High
- 種別: テスト不足
- 対象: tests/functions/persons/test_persons.py / tests/functions/persons 配下
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md#L26) の単体テスト観点では正常系・境界値・異常系などの網羅が必要。
  - 実装: [tests/functions/persons/test_persons.py](tests/functions/persons/test_persons.py#L76) で getPerson のテストが skipif(True) で無効、createPerson/getPersonAttribute のハンドラテストが見当たらない。
- 指摘内容:
  - 就業者情報詳細の主要APIに対する回帰防止が機能しておらず、仕様逸脱やリグレッションを検知できない。
- 期待されるテスト:
  - getPerson/createPerson/getPersonAttribute の正常系・異常系
  - 必須項目不足、入力形式不正、権限不足、依存先失敗（DB例外）

### 5. getTenantが認証ユーザー文脈を使わずテナントIDを直接受ける実装
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/tenants/getTenant/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/060.就業者情報/就業者情報詳細.md](spec/基本設計/機能設計/060.就業者情報/就業者情報詳細.md#L120) ではテナント設定は画面表示制御に利用され、権限と整合した取得が前提。
  - 実装: [functions/tenants/getTenant/app.py](functions/tenants/getTenant/app.py#L20) で get_current_user がコメントアウトされ、queryString の tenantId をそのままサービスに渡している。
- 指摘内容:
  - 呼び出し元の tenantId を信頼する実装で、将来的な越境参照リスクがある。
- 期待されるテスト:
  - ログインユーザーtenantIdと要求tenantId不一致時の拒否
  - tenantId未指定/不正形式時のバリデーション

### 6. searchUsersのページング入力異常時に500へフォールバックする可能性
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/users/searchUsers/app.py
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md#L30) では入力値バリエーションと異常系の明確化が必要。
  - 実装: [functions/users/searchUsers/app.py](functions/users/searchUsers/app.py#L38) で page/pageSize を int 変換しており、非数値入力時は汎用例外で500になる。
- 指摘内容:
  - クライアント入力不正をサーバー内部エラーとして返すため、障害判別と利用者体験が悪化する。
- 期待されるテスト:
  - page/pageSize に非数値、負数、0、極大値を与えた場合の応答検証

### 7. persons系APIのテスト実装品質が弱く、仕様検証ではなくモック動作検証に偏る
- 優先度: Low
- 種別: テスト不足
- 対象: tests/functions/users/test_searchUsers.py / tests/functions/users/test_deleteUser.py / tests/functions/tenants/test_tenants.py
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md#L39) ではデータ入出力の仕様一致確認が求められる。
  - 実装: 各テストが主にステータスコード中心で、レスポンス項目内容・副作用（削除連鎖、操作ログ、非更新確認）の具体検証が薄い。
- 指摘内容:
  - 実装変更時に仕様逸脱を見逃すリスクが残る。
- 期待されるテスト:
  - レスポンス項目単位アサーション
  - 削除時の関連データ削除・操作ログ作成確認
  - 条件不一致時に更新されないことの検証

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足:
  - getPerson/createPerson/getPersonAttribute のハンドラ単体テストが不足。
- 入力値バリエーション不足:
  - page/pageSize の非数値・負数・0、body文字列JSONなど入力形態差の検証が不足。
- 境界値不足:
  - 文字列長上限、件数上限、日付境界のテストが不足。
- 分岐条件不足:
  - 権限別分岐（MASTER_DATA_ADMIN/TENANT_ADMIN/NORMAL）と tenant 不一致時分岐の検証不足。
- データ入出力不足:
  - 返却フィールドの値検証、削除連鎖・操作ログなど副作用検証が不足。
- 非更新確認不足:
  - バリデーションエラー時・権限不足時にDB更新されないことの検証不足。
- 異常系不足:
  - 依存先障害、不正入力、対象なし、重複・競合条件の網羅が不足。

## 総評
- High 指摘として、更新API未実装と認証文脈の不備はリリース前に必須修正です。
- 就業者情報詳細の主要APIに対する単体テストが不足しており、仕様準拠性を継続担保できない状態です。
- 優先順位は「認証基盤是正」→「updatePerson実装」→「persons系APIテスト整備」の順が妥当です。

## 残留リスク・確認できなかった範囲
- users/searchUsers, users/deleteUser は就業者情報詳細の直接APIではないため、機能関連性の最終判断は要確認。
- spec/詳細設計 配下で本機能に紐づくAPI詳細設計書の明示参照は今回対象外入力のため未確認。
- E2E（画面-API-DB）統合観点の実行結果は未確認（静的レビューのみ）。