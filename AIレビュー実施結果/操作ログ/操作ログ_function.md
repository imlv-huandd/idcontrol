# 操作ログ レビュー結果

## 指摘事項

### 1. page/pageSize の未指定・不正値で例外化し、仕様のデフォルト動作を満たしていない
- 優先度: High
- 種別: 実装不備
- 対象: functions/auditlogs/searchAuditLogs/app.py
- 根拠:
  - 設計書: spec/詳細設計/openapi/paths/audit-logs.yaml（`page`/`pageSize` は required: false、default はそれぞれ 1/20、minimum 1）
  - 実装: functions/auditlogs/searchAuditLogs/app.py（`page: int = int(path_params.get("page")) or 1`、`page_size: int = int(path_params.get("pageSize")) or 20` が `try` ブロック外）
- 指摘内容:
  - クエリ未指定時に `int(None)` となり例外化するため、OpenAPI の「未指定時デフォルト適用」に反する。
  - 数値変換例外が `try` の外で発生するため、想定レスポンス（エラー整形）にも乗らない。
- 期待されるテスト:
  - `page`/`pageSize` 未指定時に 200 かつ `page=1` `pageSize=20` となること。
  - `page=0` `pageSize=0` `pageSize=101` `page=abc` 等で仕様どおりのエラー応答になること。

### 2. テナント分離条件が検索クエリに反映されておらず、他テナント参照リスクがある
- 優先度: High
- 種別: 実装不備
- 対象: shared/repositories/operationLogRepository.py
- 根拠:
  - 設計書: sql/schema.sql（`operation_log` は `tenant_id` を持つテーブル設計）、shared/models/operationLog.py（`tenant_id` を主キー要素として定義）
  - 実装: shared/core/middleware.py（`tenant_middleware` が `tenant_code` を抽出）、functions/auditlogs/searchAuditLogs/app.py（抽出値を検索条件へ未連携）、shared/repositories/operationLogRepository.py（`tenant_id` 条件なし）
- 指摘内容:
  - テナント識別子を取得している一方、検索時に `tenant_id` で絞り込んでいないため、データ分離要件を満たさない可能性が高い。
- 期待されるテスト:
  - 異なるテナントのログが混在するデータで、呼出テナント以外のレコードが返らないこと。
  - テナント識別子欠落時の拒否動作（401/403）または仕様定義どおりの失敗動作。

### 3. 操作日時の抽出条件が設計の日境界要件（From 00:00:00 / To 23:59:59）と一致していない
- 優先度: Middle
- 種別: 実装不備
- 対象: shared/repositories/operationLogRepository.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/120.管理機能/操作ログ.md（検索ボタン押下: `操作日時（From）00:00:00 <= 操作日時 <= 操作日時（To）23:59:59`）
  - 実装: shared/repositories/operationLogRepository.py（`datetime.fromisoformat(datetime_from)` / `datetime.fromisoformat(datetime_to)` をそのまま `>=` `<=` に使用）
- 指摘内容:
  - 実装では受信値を日境界へ正規化しておらず、設計どおりの「日単位包含」にならないケースが発生する。
- 期待されるテスト:
  - `From=2026-01-01`, `To=2026-01-01` 相当入力で、当日 00:00:00 と 23:59:59 を含むこと。
  - 境界時刻ちょうど（00:00:00 / 23:59:59）の含有確認。

### 4. 主要分岐（部分一致/完全一致・複合条件・ソート・ページング）の単体テストが不足
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/auditlogs/test_searchAuditLogs.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/120.管理機能/操作ログ.md（部分一致/完全一致、降順ソート、ページング）
  - 実装: shared/repositories/operationLogRepository.py（`ilike`、`==`、`order_by desc`、`offset/limit`）
  - テスト観点: spec/050.テスト/テスト観点.md（入力値バリエーション、境界値、分岐条件、データ入出力）
  - テスト: tests/functions/auditlogs/test_searchAuditLogs.py（正常系1件・0件・例外系中心で、上記分岐の検証なし）
- 指摘内容:
  - 現行テストはスモーク中心で、検索仕様の中核である条件分岐と並び順・ページングの正しさを保証できていない。
- 期待されるテスト:
  - `operatorUserName`/`operatorUserNameKana` の部分一致ヒット・非ヒット。
  - `function` 完全一致（一致/不一致）。
  - 複合条件ANDの成立確認。
  - 複数件データでの降順確認。
  - 2ページ目以降の `offset`/`limit` 検証。

### 5. エラー系テストのアサーションが弱く、仕様に対する検証が不十分
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/auditlogs/test_searchAuditLogs.py
- 根拠:
  - 設計書: spec/詳細設計/openapi/components/common.yaml（エラーレスポンス構造）、spec/050.テスト/テスト観点.md（異常系、データ入出力）
  - 実装: functions/auditlogs/searchAuditLogs/app.py（401/403/500 で `ErrorResponse(code, message)` を返却）
  - テスト: tests/functions/auditlogs/test_searchAuditLogs.py（主に `statusCode` のみ検証）
- 指摘内容:
  - 異常系で `code`/`message` の妥当性を検証しておらず、エラー契約逸脱を見逃す。
- 期待されるテスト:
  - 401/403/500 各ケースで `body.code` と `body.message` を明示検証。
  - DB障害・日時フォーマット不正など異常種別ごとの期待応答検証。

### 6. OpenAPI レスポンス定義と実装スキーマの項目不整合（operatorUserNameKana）
- 優先度: Low
- 種別: 設計確認事項
- 対象: spec/詳細設計/openapi/components/audit-logs.yaml
- 根拠:
  - 設計書: spec/基本設計/機能設計/120.管理機能/操作ログ.md（検索結果に「操作ユーザー名（かな）」を表示）
  - 実装: shared/schemas/operationLogSchemas.py（`operatorUserNameKana` 定義あり）、functions/auditlogs/searchAuditLogs/app.py（レスポンスへ詰替えあり）
  - 設計書（OpenAPI）: spec/詳細設計/openapi/components/audit-logs.yaml（`operatorUserNameKana` 定義なし）
- 指摘内容:
  - 基本設計/実装と OpenAPI の契約が一致していない。
- 期待されるテスト:
  - 契約テストで `operatorUserNameKana` の有無・型を検証。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足
  - 複合検索条件（ユーザー名・かな・機能・日時）同時指定時の成立確認が不足。
- 入力値バリエーション不足
  - 部分一致/完全一致の各パターン、未指定/空文字、`page`/`pageSize` の多様入力が不足。
- 境界値不足
  - `page`/`pageSize` 最小・最大、日時境界（00:00:00/23:59:59、From=To）の検証不足。
- 分岐条件不足
  - 各検索条件有無によるクエリ分岐、0件/1件/複数件、ソート分岐の確認不足。
- データ入出力不足
  - レスポンス項目（特に `operationTime` 形式、エラー `code/message`）の厳密検証不足。
- 非更新確認不足
  - 検索APIとして副作用（更新なし）を確認する観点が未整備。
- 異常系不足
  - パラメータ形式不正、境界外値、依存先失敗種別（DB接続失敗等）の個別検証不足。

## 総評
- High 指摘は、デフォルトページング不備とテナント分離欠落で、機能停止およびセキュリティリスクに直結する。
- Middle 指摘は、日時条件の仕様一致性と検索ロジックの検証不足で、誤検索や回帰混入の温床となる。
- まず High を優先修正し、その後に検索条件分岐・境界値のテスト拡充で品質を底上げすべき。

## 残留リスク・確認できなかった範囲
- `spec/詳細設計/openapi/components/common.yaml` の全文照合は未実施（エラーコード体系の網羅一致は未確認）。
- フロントエンド側のクライアントバリデーション実装（日時範囲チェック COM02018）の実コード照合は未実施。
- 他APIを含む横断的なテナント制御方針書（概要/共通設計）の突合は未実施。