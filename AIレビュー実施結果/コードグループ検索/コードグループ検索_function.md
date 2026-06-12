# コードグループ検索 レビュー結果

## 指摘事項

### 1. 検索結果のタイムスタンプ形式と削除API比較形式が不整合で、正当な削除が競合扱いになる
- 優先度: High
- 種別: 実装不備
- 対象: コードグループ検索API/コードグループ削除API
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ検索.md](spec/基本設計/機能設計/100.コード定義/コードグループ検索.md) の「削除条件」にて、検索時のタイムスタンプ一致で削除判定する仕様。
  - 実装: 検索側は [functions/codegroups/searchCodeGroups/app.py](functions/codegroups/searchCodeGroups/app.py#L73) でレスポンスに `times` を設定し、実データは [functions/codegroups/searchCodeGroups/app.py](functions/codegroups/searchCodeGroups/app.py#L58) で `isoformat()` 変換。削除側は [functions/codegroups/deleteCodeGroup/app.py](functions/codegroups/deleteCodeGroup/app.py#L54) で文字列受領し、[functions/codegroups/deleteCodeGroup/app.py](functions/codegroups/deleteCodeGroup/app.py#L78) で `str(codegroupmaster.times)` と直接比較。
- 指摘内容:
  - 検索レスポンス形式 (ISO 8601) と削除時比較形式 (`str(datetime)`) が一致保証されておらず、同一レコードでも `times` 不一致で COM01010 を返すリスクがある。
- 期待されるテスト:
  - 検索APIで返却された `times` をそのまま削除APIに渡して削除成功する連携テスト。
  - `times` のフォーマット差 (`T` 区切り/タイムゾーン有無/マイクロ秒有無) を含む比較テスト。

### 2. 削除操作ログの操作内容に実際のコードグループ名が記録されない
- 優先度: High
- 種別: 実装不備
- 対象: functions/codegroups/deleteCodeGroup/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ検索.md](spec/基本設計/機能設計/100.コード定義/コードグループ検索.md) の「操作ログの登録」にて、操作内容は「コードグループ削除：[コードグループ名]」を記録する仕様。
  - 実装: [functions/codegroups/deleteCodeGroup/app.py](functions/codegroups/deleteCodeGroup/app.py#L164) で `operation="コードグループ削除：[コードグループ名]"` の固定文字列を登録しており、実データ埋め込みがない。
- 指摘内容:
  - 監査ログが削除対象を特定できず、監査証跡として要件未達。
- 期待されるテスト:
  - `create_operationLog` 呼び出し時の `operation` 引数が実際の `code_group_name` を含むことを検証するテスト。

### 3. page/pageSize の入力バリデーション不足により不正入力が500系で落ちる
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/codegroups/searchCodeGroups/app.py
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md#L28) の単体テスト観点に「入力値バリエーション」「境界値」「異常系」が定義。
  - 実装: [functions/codegroups/searchCodeGroups/app.py](functions/codegroups/searchCodeGroups/app.py#L23) と [functions/codegroups/searchCodeGroups/app.py](functions/codegroups/searchCodeGroups/app.py#L24) で `int()` へ直接変換し、数値変換失敗時は最終的に [functions/codegroups/searchCodeGroups/app.py](functions/codegroups/searchCodeGroups/app.py#L99) の汎用500返却に流れる構造。
- 指摘内容:
  - `page`/`pageSize` に非数値・0・負数が来た場合の入力エラー制御が不足。利用者入力不備がサーバ内部エラー扱いとなる。
- 期待されるテスト:
  - `page=abc`, `page=0`, `page=-1`, `pageSize=0`, `pageSize=-1` の入力バリエーションテスト。
  - 期待ステータス/エラーコード (400系) の明示テスト。

### 4. 監査ログ内容を検証する単体テストが存在しない
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/codegroups/test_codegroups.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ検索.md](spec/基本設計/機能設計/100.コード定義/コードグループ検索.md) の「操作ログの登録」仕様。
  - 実装: [functions/codegroups/deleteCodeGroup/app.py](functions/codegroups/deleteCodeGroup/app.py#L159) から [functions/codegroups/deleteCodeGroup/app.py](functions/codegroups/deleteCodeGroup/app.py#L166) で操作ログ登録を実施。SubAgent確認結果として、同引数の値検証を行うテストは該当なし。
- 指摘内容:
  - ログ登録APIが呼ばれるだけで値が誤っていても検知できない構成になっている。
- 期待されるテスト:
  - `OperationLogRepository.create_operationLog` 呼び出し引数の `function` と `operation` を具体値でアサートするテスト。

### 5. 検索API返却のtimesを削除APIへ受け渡す仕様連携テストがない
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/codegroups/test_searchCodeGroups.py / tests/functions/codegroups/test_codegroups.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ検索.md](spec/基本設計/機能設計/100.コード定義/コードグループ検索.md) の削除条件で「検索時タイムスタンプ一致」が要求。
  - 実装: 検索返却は [functions/codegroups/searchCodeGroups/app.py](functions/codegroups/searchCodeGroups/app.py#L58) の `isoformat()`、削除比較は [functions/codegroups/deleteCodeGroup/app.py](functions/codegroups/deleteCodeGroup/app.py#L78) の文字列比較。SubAgent確認結果として両API連携テストは該当なし。
- 指摘内容:
  - API単体では見えないフォーマット差異不具合を捕捉できない。
- 期待されるテスト:
  - 検索レスポンスの `times` をそのまま delete の `times` に渡すE2E相当テスト。

### 6. 検索0件時のCOM01003表示責務がAPI仕様として不明確
- 優先度: Low
- 種別: 設計確認事項
- 対象: コードグループ検索機能
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ検索.md](spec/基本設計/機能設計/100.コード定義/コードグループ検索.md) の検索仕様に「検索結果0件の場合、COM01003表示」。
  - 実装: [functions/codegroups/searchCodeGroups/app.py](functions/codegroups/searchCodeGroups/app.py#L73) から [functions/codegroups/searchCodeGroups/app.py](functions/codegroups/searchCodeGroups/app.py#L76) で0件でも通常200レスポンス返却のみ。
- 指摘内容:
  - 0件メッセージをAPIで返す前提か、フロントで `totalCount==0` を解釈して表示する前提かが設計上分離されていない。
- 期待されるテスト:
  - API責務を明確化した上で、0件時のUI表示/メッセージ判定の結合テストまたはフロント単体テスト。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足
  - 検索API返却 `times` を削除APIに受け渡して削除成功する連携正常系が不足。
- 入力値バリエーション不足
  - `page`/`pageSize` の非数値・0・負数、`times` フォーマット差の入力検証ケースが不足。
- 境界値不足
  - `page=1` 以外の境界 (0, 負数, 大きい値) と `pageSize` 境界の検証が不足。
- 分岐条件不足
  - 監査ログ生成時の値内容分岐 (対象名の埋め込み) を検証していない。
- データ入出力不足
  - 操作ログTBLへ記録される `operation` の内容整合性検証が不足。
- 非更新確認不足
  - `times` 不一致時に削除/ログ登録が行われないことを副作用まで含めて確認する検証が不足。
- 異常系不足
  - 入力不正を400系で返す設計を担保する異常系ケースが不足。

## 総評
- Highは、削除可否判定に直結する `times` 比較形式不整合と、監査証跡不備（操作ログ固定文言）の2点です。
- いずれも本番での誤拒否・追跡不能につながるため、リリース前修正が必要です。
- 次点で、入力値バリデーションとAPI間連携テスト不足が品質リスクとして残っています。

## 残留リスク・確認できなかった範囲
- [spec/詳細設計](spec/詳細設計) 配下で、`times` の厳密フォーマット（ISO 8601固定か、DB文字列表現許容か）を規定した文書を今回確認できていません。
- [spec/基本設計/機能設計/システム共通/ページネーション.md](spec/基本設計/機能設計/システム共通/ページネーション.md) の詳細仕様本文を未確認のため、`page`/`pageSize` エラーコードの最終確定は要確認です。
- [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md) は観点レベルまで確認済みで、機能別ケース定義（別紙）がある場合は追加突合余地があります。
