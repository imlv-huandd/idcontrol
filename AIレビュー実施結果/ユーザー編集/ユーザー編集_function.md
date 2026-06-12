# ユーザー編集 レビュー結果

## 指摘事項

### 1. 更新時の楽観的ロック（タイムスタンプ一致条件）が未実装
- 優先度: High
- 種別: 実装不備
- 対象: functions/users/updateUser/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md](spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md#L133-L137) 更新条件は「ユーザーID + 取得時タイムスタンプ一致」、不一致時 COM01010。
  - 実装: [functions/users/updateUser/app.py](functions/users/updateUser/app.py#L148-L151) で `times` を現在時刻で上書きして更新しており、取得時タイムスタンプ一致判定がない。更新処理は [shared/repositories/usermasterRepository.py](shared/repositories/usermasterRepository.py#L314) の `merge` 実行のみ。
- 指摘内容:
  - 同時更新競合を検知できず、後勝ち上書きで誤更新が発生し得る。設計上の COM01010 分岐も到達しない。
- 期待されるテスト:
  - 取得時タイムスタンプと異なる値で更新要求した場合に更新されず、所定エラー（COM01010）になること。
  - 競合時に対象レコード以外へ副作用が発生しないこと（非更新確認）。

### 2. createUser が途中コミットしており整合性を壊す可能性がある
- 優先度: High
- 種別: 実装不備
- 対象: functions/users/createUser/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md](spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md#L110-L117) 登録時はユーザーマスター登録、権限登録、操作ログ記録を一連で実施。
  - 実装: [functions/users/createUser/app.py](functions/users/createUser/app.py#L104) で user_master 登録直後に `db.commit()` してから、権限登録・操作ログ登録を続行。
- 指摘内容:
  - 権限登録や操作ログで失敗した場合に user_master だけ確定する部分成功状態となり、業務整合性と監査整合性を崩す。
- 期待されるテスト:
  - user_authority 登録失敗時に user_master もロールバックされること。
  - operation_log 登録失敗時に関連更新がロールバックされること。

### 3. API層で権限制御が実質未実装
- 優先度: High
- 種別: 実装不備
- 対象: functions/users/getUser/app.py, functions/users/updateUser/app.py, functions/users/createUser/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md](spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md) にロール別の詳細記載はないが、ユーザー編集機能として認可不備は重大リスク。
  - 実装: [functions/users/updateUser/app.py](functions/users/updateUser/app.py#L39-L40) / [functions/users/getUser/app.py](functions/users/getUser/app.py#L24-L25) でログインユーザー取得後、権限判定分岐が存在しない。共通処理 [shared/core/common.py](shared/core/common.py#L40-L45) も認証情報抽出がモック状態。
- 指摘内容:
  - 実運用時に認可漏れがあると、編集権限のないユーザーによる参照・更新を許容する可能性がある。
- 期待されるテスト:
  - 権限なしユーザーでの get/update/create が 403 となること。
  - 他ユーザー編集可否（本人のみ可/管理者可など）の仕様に沿った分岐テスト。

### 4. createUser の入力不正・重複時ステータスコードが不適切
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/users/createUser/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md](spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md#L105-L109) 入力チェック・重複チェックのサーバーサイド検証。
  - 実装: [functions/users/createUser/app.py](functions/users/createUser/app.py#L57-L69) 入力検証失敗で 500、[functions/users/createUser/app.py](functions/users/createUser/app.py#L108-L126) 重複で 401 を返却。
- 指摘内容:
  - クライアント実装や監視の観点で、クライアント起因エラー（4xx）とサーバー障害（5xx）が混在し、誤判定を招く。
- 期待されるテスト:
  - 入力不正時に 400、重複時に 409 を返すこと。
  - updateUser/getUser とエラー分類が統一されること。

### 5. 更新・取得時の業務エラーコードが設計と不整合
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/users/getUser/app.py, functions/users/updateUser/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md](spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md#L83-L86) 編集モードでユーザー未存在時は COM01009、[spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md](spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md#L157-L158) 重複チェックメッセージIDは USR02001。
  - 実装: [functions/users/getUser/app.py](functions/users/getUser/app.py#L52) は NOT_FOUND、[functions/users/updateUser/app.py](functions/users/updateUser/app.py#L105-L110) は USR02001、[functions/users/updateUser/app.py](functions/users/updateUser/app.py#L122-L138) は USR02002/USR02003。
- 指摘内容:
  - フロント側メッセージマッピングと不一致となり、仕様通りのユーザー通知にならないリスク。
- 期待されるテスト:
  - 未存在時、重複時のエラーコードが設計書のIDに一致すること。
  - フロントメッセージ定義との結合観点テスト（コード→文言解決）。

### 6. 更新時の操作ログ内容が設計仕様と一致していない
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/users/updateUser/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md](spec/基本設計/機能設計/030.ユーザー管理/ユーザー編集.md#L139-L142) 操作内容は「ユーザー更新：[ログインID]」。
  - 実装: [functions/users/updateUser/app.py](functions/users/updateUser/app.py#L167) は `operation=f"ユーザー更新：{user_id}"` で user_id を記録。
- 指摘内容:
  - 監査ログの検索性・仕様整合性が低下し、運用時のトレーサビリティが落ちる。
- 期待されるテスト:
  - 更新成功時の operation_log.operation がログインIDを含むこと。

### 7. 単体テストが spec/050 の主要観点を十分に満たしていない
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/users/test_getUser.py, tests/functions/users/test_updateUser.py, tests/functions/users/test_createUser.py
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md#L20-L39) に正常系/入力値バリエーション/境界値/分岐条件/データ入出力/非更新確認/異常系が定義。
  - 実装: [tests/functions/users/test_updateUser.py](tests/functions/users/test_updateUser.py#L20-L193) と [tests/functions/users/test_createUser.py](tests/functions/users/test_createUser.py#L28-L168) は境界値・非更新確認・操作ログ副作用検証が不足。
- 指摘内容:
  - 重要分岐（競合更新、部分失敗時ロールバック、ログ内容）を見逃す構成で、仕様逸脱の早期検知が困難。
- 期待されるテスト:
  - 文字数上限/下限・メール形式境界の境界値テスト。
  - 競合更新時の非更新確認テスト。
  - createUser で権限・操作ログまで含む一連副作用検証。

### 8. モック中心で仕様検証が弱く、副作用の欠陥を見逃しやすい
- 優先度: Low
- 種別: テスト不足
- 対象: tests/functions/users/test_createUser.py, tests/functions/users/test_updateUser.py
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md#L32-L36) データ入出力・非更新確認の観点。
  - 実装: [tests/functions/users/test_createUser.py](tests/functions/users/test_createUser.py#L28-L45) は repository 初期化と一部戻り値中心で、`create_user_authority` や `create_operationLog` の引数妥当性・呼び出し順序を厳密に検証していない。
- 指摘内容:
  - 実装依存の浅いアサーションになり、監査ログ不整合や部分コミット問題がテストで検知されにくい。
- 期待されるテスト:
  - 各 repository 呼び出し引数（tenant_id, user_id, operation文言）の厳密検証。
  - 失敗注入で呼び出し順序・ロールバック有無を確認するテスト。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足
  - createUser/updateUser 成功時に user_authority・operation_log まで含む完了条件の検証が不足。
- 入力値バリエーション不足
  - `None`、空文字、形式不正（UUID/メール）に対する組合せ検証が不足。
- 境界値不足
  - loginId/userName/userNameKana の 100 文字境界、mailAddress の 2048 文字境界が未検証。
- 分岐条件不足
  - 競合更新（タイムスタンプ不一致）、重複チェック優先順などの分岐が未検証。
- データ入出力不足
  - OperationLog の function/operation 値、UserAuthority 初期値（閲覧者）の検証が不足。
- 非更新確認不足
  - 更新失敗時に user_master が不変であること、他ユーザーへ副作用がないことの検証が不足。
- 異常系不足
  - createUser の途中失敗（権限登録失敗、操作ログ失敗）時のロールバックとエラーレスポンス検証が不足。

## 総評
High 指摘として、更新時の楽観的ロック未実装と createUser の部分コミット可能性、認可チェック欠落を確認しました。
いずれも誤更新・データ不整合・権限不備に直結するため、最優先で実装修正とテスト追加が必要です。
次点で、エラーコード/ステータス不整合と監査ログ不整合を是正し、フロント連携の仕様整合性を確保してください。

## 残留リスク・確認できなかった範囲
- 認可要件（どのロールがどの操作可能か）の詳細は設計書上で明示が弱く、最終仕様確認が必要。
- get_current_user がモック状態のため、実運用の認証連携（JWT検証）時挙動は本レビュー範囲で未確認。
