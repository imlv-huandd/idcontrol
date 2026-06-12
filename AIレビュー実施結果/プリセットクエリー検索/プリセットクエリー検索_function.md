# プリセットクエリー検索 レビュー結果

## 指摘事項

### 1. 削除時の関連権限データ削除が未実装
- 優先度: High
- 種別: 実装不備
- 対象: プリセットクエリー削除機能
- 根拠:
  - 設計書: [spec/基本設計/機能設計/080.プリセットクエリー/プリセットクエリー検索.md](spec/基本設計/機能設計/080.プリセットクエリー/プリセットクエリー検索.md#L73-L81) では、削除時にグループクエリー使用権限・ユーザークエリー使用権限も削除する要件が明記されている。
  - 実装: [shared/services/presetQuerieService.py](shared/services/presetQuerieService.py#L70-L110) の delete_preset_query は preset_query の削除と操作ログ記録のみ実行。関連メソッドは [shared/repositories/userQueryAuthorityRepository.py](shared/repositories/userQueryAuthorityRepository.py#L10-L21) と [shared/repositories/groupQueryAuthorityRepository.py](shared/repositories/groupQueryAuthorityRepository.py#L50-L60) に存在するが呼び出されていない。
- 指摘内容:
  - 仕様上必要な関連権限レコード削除が抜けており、削除後に孤児データが残る。データ整合性と後続権限制御に影響するため、リリース前修正が必要。
- 期待されるテスト:
  - 削除成功時に preset_query と user_query_authority/group_query_authority が同一 query_id で物理削除されること。
  - 片方削除失敗時にトランザクションがロールバックされること。

### 2. 対象APIの単体テストが未整備
- 優先度: High
- 種別: テスト不足
- 対象: searchPresetQueries API / deletePresetQuery API
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md#L9-L40) で機能単位の単体テスト観点（正常系、入力値バリエーション、境界値、分岐条件、データ入出力、非更新確認、異常系）が要求されている。
  - 実装: API本体は [functions/presetQueries/searchPresetQueries/app.py](functions/presetQueries/searchPresetQueries/app.py#L24-L57) と [functions/presetQueries/deletePresetQuery/app.py](functions/presetQueries/deletePresetQuery/app.py#L24-L57) にあるが、対応テストは未存在。既存は [tests/functions/presetQueries/test_presetQueries.py](tests/functions/presetQueries/test_presetQueries.py#L9-L33) の execute/list 系のみ。
- 指摘内容:
  - 対象機能の主要分岐と異常系が未検証で、仕様逸脱や回帰を検知できない状態。
- 期待されるテスト:
  - search: 条件なし、部分一致、0件、ページング境界、権限エラー、予期せぬ例外。
  - delete: 権限なし(COM01001)、対象なし(COM01010)、times不一致、削除成功時の操作ログ、依存先失敗時の500。

### 3. 削除完了レスポンスが設計のメッセージ仕様と不一致
- 優先度: Middle
- 種別: 実装不備
- 対象: 削除APIレスポンス
- 根拠:
  - 設計書: [spec/基本設計/機能設計/080.プリセットクエリー/プリセットクエリー検索.md](spec/基本設計/機能設計/080.プリセットクエリー/プリセットクエリー検索.md#L91) では完了メッセージ COM01005 の表示を要求。
  - 実装: [shared/services/presetQuerieService.py](shared/services/presetQuerieService.py#L110) は code=204, message=プリセットクエリー削除 success を返却。
- 指摘内容:
  - 画面側がメッセージID前提で共通表示制御している場合に整合しない可能性がある。仕様とAPI契約を合わせる必要がある。
- 期待されるテスト:
  - 削除成功時レスポンスが設計で定義した完了メッセージ仕様に一致すること。

### 4. 入力値検証の異常系テスト観点が不足
- 優先度: Middle
- 種別: テスト不足
- 対象: API入力パラメータ検証
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md#L18-L40) は入力値バリエーション・境界値・異常系の確認を要求。
  - 実装: [functions/presetQueries/searchPresetQueries/app.py](functions/presetQueries/searchPresetQueries/app.py#L30-L31) で page/pageSize を int 変換、[shared/services/presetQuerieService.py](shared/services/presetQuerieService.py#L96-L99) で times を datetime 変換しており、変換失敗や不正値の分岐が存在。
- 指摘内容:
  - page/pageSize 非数値、負数、極大値、times フォーマット不正の振る舞いが未検証。境界条件の不具合を取りこぼすリスクがある。
- 期待されるテスト:
  - page=0, page=-1, pageSize=0, pageSize過大、times不正文字列、queryStringParameters欠落時のレスポンス整形。

### 5. 削除後の再検索・最終ページ補正の責務分界が不明
- 優先度: Low
- 種別: 設計確認事項
- 対象: 削除後画面更新仕様
- 根拠:
  - 設計書: [spec/基本設計/機能設計/080.プリセットクエリー/プリセットクエリー検索.md](spec/基本設計/機能設計/080.プリセットクエリー/プリセットクエリー検索.md#L92-L94) で削除後に現在ページ再検索、0件時は最終ページ表示を要求。
  - 実装: [functions/presetQueries/deletePresetQuery/app.py](functions/presetQueries/deletePresetQuery/app.py#L34-L40) と [shared/services/presetQuerieService.py](shared/services/presetQuerieService.py#L70-L110) は削除結果返却のみで再検索情報を返さない。
- 指摘内容:
  - 本要件をフロント側責務で満たす設計なのか、APIで補助情報を返す設計なのかが不明確。責務境界を明文化しないと実装差異が継続する。
- 期待されるテスト:
  - 画面統合テストで、削除後のページ再検索と最終ページ補正が仕様どおりになること。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足: search成功（条件なし/条件あり）と delete成功（関連権限削除＋操作ログ）のAPIテストが不足。
- 入力値バリエーション不足: presetQueryName 部分一致、未指定、page/pageSize の入力差分検証が不足。
- 境界値不足: page/pageSize 境界、検索0件、最終ページ補正の検証が不足。
- 分岐条件不足: 権限あり/なし、対象データあり/なし、times一致/不一致、依存先例外の分岐確認が不足。
- データ入出力不足: 削除時の関連テーブル削除、操作ログ内容、レスポンス形式の検証が不足。
- 非更新確認不足: 権限不足・times不一致・対象なし時にデータ非更新であることの確認が不足。
- 異常系不足: 不正パラメータ、DB例外、予期しない例外時のエラーコード/メッセージ確認が不足。

## 総評
- High指摘は、削除時の関連権限削除漏れと対象APIテスト未整備の2点で、どちらも本番障害や回帰未検知に直結する。
- 先に実装整合（関連権限削除、レスポンス仕様）を是正し、その後に単体テスト観点を網羅する順で修正するのが妥当。
- 現状のままでは削除機能のデータ整合性と品質保証が不足しており、マージ判断は見送りが妥当。

## 残留リスク・確認できなかった範囲
- 削除後の再検索・最終ページ補正をフロントエンドで担保している可能性は未確認（今回のレビュー対象外）。
- OpenAPI 定義と実装レスポンス整合の最終確認は未実施。
- 既存テストデータ fixture の妥当性（データパターン網羅）は未確認。