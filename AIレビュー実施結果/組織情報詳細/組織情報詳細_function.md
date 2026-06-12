# 組織情報詳細 レビュー結果

## 指摘事項

### 1. 変更有無チェックの条件式が設計意図と不整合
- 優先度: High
- 種別: 実装不備
- 対象: shared/services/organizationService.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織情報詳細.md](spec/基本設計/機能設計/070.組織情報/組織情報詳細.md) の「変更有無チェック」は「更新前後で有効開始日以外の項目に差分があること」。
  - 実装: [shared/services/organizationService.py](shared/services/organizationService.py#L760) 付近の changesCheck が有効開始日比較と他項目比較を混在した条件で判定しており、設計要件と一致しないケースが発生する。
- 指摘内容:
  - 現状ロジックでは、履歴追加時や有効開始日のみ変更時の判定が仕様意図とずれる可能性があり、誤って更新可否を判定するリスクがある。
- 期待されるテスト:
  - 有効開始日のみ変更、有効開始日以外のみ変更、両方変更、どちらも未変更の4パターンを changesCheck 単体で検証する。

### 2. getTenant APIでユーザー取得処理が無効化されており認可要件が不明瞭
- 優先度: High
- 種別: 実装不備
- 対象: functions/tenants/getTenant/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織情報詳細.md](spec/基本設計/機能設計/070.組織情報/組織情報詳細.md) は権限統制を前提に記載（マスターデータ管理者チェックなど）。
  - 実装: [functions/tenants/getTenant/app.py](functions/tenants/getTenant/app.py#L20) 付近で get_current_user 呼び出しがコメントアウトされ、呼び出し主体に応じた明示的な制御がない。
- 指摘内容:
  - tenant_middleware のみで十分か、API単位での認可要件が必要かが曖昧で、想定外利用を招くリスクがある。
- 期待されるテスト:
  - 認証済みかつ許可ユーザー、認証済みだが非許可ユーザー、未認証の3条件で getTenant のレスポンスコードを検証する。

### 3. サービス層の業務ロジックがほぼ未検証
- 優先度: High
- 種別: テスト不足
- 対象: tests/functions/organizations/test_organizations.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織情報詳細.md](spec/基本設計/機能設計/070.組織情報/組織情報詳細.md) には、重複チェック、権限チェック、履歴追加チェック、操作ログ記録などサービス層の業務仕様が定義されている。
  - 実装: [tests/functions/organizations/test_organizations.py](tests/functions/organizations/test_organizations.py) はハンドラーでサービス関数をモック化する構成が中心で、organizationService 実ロジックの分岐が通っていない。
- 指摘内容:
  - ハンドラーの配線検証に偏っており、仕様の核であるバリデーション・履歴更新・競合判定・権限判定の欠陥を見逃す可能性が高い。
- 期待されるテスト:
  - create_organization、update_organization、abolish_organization のサービス層単体テストを追加し、設計記載のサーバーサイドバリデーション分岐を直接検証する。

### 4. 廃止処理の操作ログ機能名が設計と不一致
- 優先度: Middle
- 種別: 実装不備
- 対象: shared/services/organizationService.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織情報詳細.md](spec/基本設計/機能設計/070.組織情報/組織情報詳細.md) の操作ログは対象機能「組織情報詳細」。
  - 実装: [shared/services/organizationService.py](shared/services/organizationService.py#L593) で abolish_organization の operation log に function="組織検索" を設定している。
- 指摘内容:
  - 操作ログの機能分類が不一致となり、監査時の追跡性と分析精度を低下させる。
- 期待されるテスト:
  - 廃止API実行時に operation log の function/action が設計値になることを検証する。

### 5. update_organization のエラー返却形式が混在
- 優先度: Middle
- 種別: 実装不備
- 対象: shared/services/organizationService.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織情報詳細.md](spec/基本設計/機能設計/070.組織情報/組織情報詳細.md) はサーバーサイドバリデーションでエラー時に終了する振る舞いを定義。
  - 実装: [shared/services/organizationService.py](shared/services/organizationService.py#L380) と [shared/services/organizationService.py](shared/services/organizationService.py#L423) で一部は (500, ErrorResponse) を返し、他分岐は ServiceException を送出している。
- 指摘内容:
  - 呼び出し側の前提が揺れ、将来的な保守時に例外処理漏れやレスポンス不整合の原因となる。
- 期待されるテスト:
  - changesCheck/addHistoryCheck 失敗時にハンドラーが設計どおりのエラーコード・メッセージで応答することを検証する。

### 6. 入力チェックの境界値テストが不足
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/shared/schemas/test_organizationSchemas.py, tests/functions/organizations/test_organizations.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織情報詳細.md](spec/基本設計/機能設計/070.組織情報/組織情報詳細.md) のバリデーション定義に、100/200文字上限や日付形式などの境界要件が明記。
  - 実装: [tests/shared/schemas/test_organizationSchemas.py](tests/shared/schemas/test_organizationSchemas.py) と [tests/functions/organizations/test_organizations.py](tests/functions/organizations/test_organizations.py) ではサービス層バリデーション境界の網羅が不足。
- 指摘内容:
  - 仕様で明示された上限超過・形式不正・必須未入力の境界が十分に担保されていない。
- 期待されるテスト:
  - tenantOrgId/orgName/orgNameAll/orgNameDisplay の上限ちょうど・1超過、startDate の形式不正、必須空値を網羅する。

### 7. 追加情報の入力タイプ別検証テストが不足
- 優先度: Middle
- 種別: テスト不足
- 対象: shared/services/organizationService.py, tests/functions/organizationAttribute/test_organizationAttribute.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織情報詳細.md](spec/基本設計/機能設計/070.組織情報/組織情報詳細.md) の「追加情報入力チェック内容」で入力タイプ別制約を定義。
  - 実装: [shared/services/organizationService.py](shared/services/organizationService.py) の additionalInformationInputCheck 相当ロジックに対し、[tests/functions/organizationAttribute/test_organizationAttribute.py](tests/functions/organizationAttribute/test_organizationAttribute.py) はハンドラー中心でタイプ別異常系が不足。
- 指摘内容:
  - 動的属性の型・桁・必須制約違反が本番で顕在化するリスクがある。
- 期待されるテスト:
  - TEXT_ALPHA、TEXT_NUMERIC、INTEGER、DATE、コードグループ参照の各タイプで正常/異常値を最小1件以上追加する。

### 8. 例外時ログ出力方針の一貫性が低い
- 優先度: Low
- 種別: 実装不備
- 対象: functions/organizations/createOrganization/app.py, functions/organizations/updateOrganization/app.py, functions/organizations/abolishOrganization/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/070.組織情報/組織情報詳細.md](spec/基本設計/機能設計/070.組織情報/組織情報詳細.md) の運用上、監査・障害解析に利用するログは一貫した方針が必要。
  - 実装: [functions/organizations/createOrganization/app.py](functions/organizations/createOrganization/app.py#L55) と [functions/organizations/updateOrganization/app.py](functions/organizations/updateOrganization/app.py#L53) で print と logger を混在、[functions/organizations/abolishOrganization/app.py](functions/organizations/abolishOrganization/app.py#L25) で通常イベントを error レベル記録。
- 指摘内容:
  - 運用時のノイズ増加と障害解析効率低下につながる。
- 期待されるテスト:
  - なし

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足
  - サービス層 create/update/abolish の正常系（権限・履歴分岐・操作ログ記録完了）を直接検証するテストが不足。
- 入力値バリエーション不足
  - 追加情報の入力タイプ別（英字、数字、日付、コード存在）で正常/異常のバリエーションが不足。
- 境界値不足
  - 基本情報の文字数上限（100/200）ちょうど値、超過値、空値の境界テストが不足。
- 分岐条件不足
  - update の changesCheck/addHistoryCheck、abolish の権限分岐（TENANT_ADMIN/MASTER_DATA_ADMIN）の分岐網羅が不足。
- データ入出力不足
  - 操作ログの function/action 値、履歴更新時の start_date/end_date 更新結果の確認が不足。
- 非更新確認不足
  - 変更なし更新時に適切なエラー（変更有無チェック）となることの確認が不足。
- 異常系不足
  - 認可不備、入力不正、重複、競合（タイムスタンプ不一致）のサービス層異常系が不足。

## 総評
- High 指摘として、変更有無チェックの判定ロジック不整合と getTenant の認可処理曖昧化は、誤判定や認可境界の不備に直結するため優先修正が必要。
- 併せて、現行テストはハンドラー中心でサービス層仕様の検証が不足しており、重要な業務不具合を取り逃すリスクが高い。
- 修正優先順位は、1) 判定ロジックと認可要件の明確化、2) サービス層テストの拡充、3) ログ整合性の改善。

## 残留リスク・確認できなかった範囲
- [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md) の全観点に対し、E2E/結合テストレベルでの実施状況までは確認対象外。
- DB実データを用いた競合再現（同時更新）や性能劣化観点は今回レビュー範囲外。
