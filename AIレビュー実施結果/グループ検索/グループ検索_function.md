# グループ検索 レビュー結果

## 指摘事項

### 1. 500エラー応答で内部例外詳細をそのまま返却している
- 優先度: High
- 種別: 実装不備
- 対象: functions/groups/searchGroups/app.py, functions/groups/deleteGroup/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/040.グループ管理/グループ検索.md のバリデーション定義・メッセージ運用では業務メッセージIDでの通知を前提としており、内部実装詳細の返却は想定されていない。
  - 実装: functions/groups/searchGroups/app.py では内部例外時に message="グループの取得に失敗しました。: " + str(e) を返却。functions/groups/deleteGroup/app.py でも message="グループの削除に失敗しました。: " + str(e) を返却している。
- 指摘内容:
  - 例外文字列をそのままAPIレスポンスへ含めるため、SQL/内部構造/スタック由来情報の漏えいリスクがある。セキュリティ観点でリリース前に修正が必要。
- 期待されるテスト:
  - 依存先例外発生時に、レスポンス本文へ内部例外文字列が含まれないことを検証するテスト。

### 2. 入力パラメータの必須・形式チェック不足により不正入力が500扱いになる
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/groups/deleteGroup/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/040.グループ管理/グループ検索.md の削除ボタン押下では、削除対象の識別情報を用いたサーバーサイド処理を前提にしており、入力不備での異常系ハンドリング方針が必要。
  - 実装: functions/groups/deleteGroup/app.py で group_id = uuid.UUID(group_id_raw), times = datetime.fromisoformat(...) を try ブロック外で実行しているため、欠損/不正形式時は例外が500へ集約される。
- 指摘内容:
  - groupId 未指定・UUID不正・times不正形式の入力が、業務上の入力エラーではなくサーバーエラーとして扱われる。利用者視点で誤ったエラー分類となる。
- 期待されるテスト:
  - groupId未指定、groupId形式不正、times形式不正で想定ステータス（通常は400系）と業務メッセージが返ること。

### 3. 検索APIのページング境界値バリデーションがなく、不正値がそのままSQL条件に流れる
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/groups/searchGroups/app.py, shared/repositories/groupmasterRepository.py
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md の単体テスト観点に「境界値」が定義されており、数値範囲の妥当性確認が必要。
  - 実装: functions/groups/searchGroups/app.py で page/pageSize を int 変換するのみで下限チェックなし。shared/repositories/groupmasterRepository.py では offset=(page-1)*page_size を直接適用。
- 指摘内容:
  - page=0 や負数、pageSize=0/負数が許容されると、意図しないSQL実行やページング結果不整合を引き起こす可能性がある。
- 期待されるテスト:
  - page/pageSize の 0、負数、極端な大値に対する入力バリデーション・応答確認テスト。

### 4. searchGroups テストで単体テスト観点の網羅が不足している
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/groups/test_searchGroups.py, tests/functions/groups/test_searchGroups_additional.py
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md の「入力値バリエーション」「境界値」「分岐条件」「異常系」。
  - 実装: tests/functions/groups/test_searchGroups.py / test_searchGroups_additional.py では正常系、空結果、デフォルト値、withUserNames true/false、内部例外の一部のみ確認。
- 指摘内容:
  - groupName 部分一致のバリエーション、page/pageSize の境界値、文字種/空白系、不正型入力時の挙動が未検証で、仕様起点の担保が弱い。
- 期待されるテスト:
  - groupName の先頭一致/中間一致/後方一致。
  - page/pageSize の境界値（1、0、負数、上限相当値）。
  - queryStringParameters に不正型や空文字が来た場合の分岐。

### 5. deleteGroup テストで削除処理の実行妥当性検証が不足している
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/groups/test_deleteGroup.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/040.グループ管理/グループ検索.md の削除ボタン押下では、グループ所属・グループ権限マスター・グループクエリー使用権限・グループマスターの削除、および操作ログ記録を要求。
  - 実装: functions/groups/deleteGroup/app.py は複数Repositoryを順次呼び出して削除。tests/functions/groups/test_deleteGroup.py の成功系は主に statusCode 検証で、各delete呼び出しの assert が不足。
- 指摘内容:
  - 複数テーブル削除の呼び出し漏れや、将来改修での処理順/呼び出し条件崩れをテストで検出できない。
- 期待されるテスト:
  - 各 repository の delete メソッド呼び出し有無・引数検証。
  - groupmaster 削除時の操作ログ記録（function/operation）の検証。
  - 途中失敗時に rollback され、後続削除が実行されないことの検証。

### 6. グループ削除の独立設計書が見当たらず、機能単位のトレーサビリティが弱い
- 優先度: Low
- 種別: 設計確認事項
- 対象: spec/基本設計/機能設計/040.グループ管理
- 根拠:
  - 設計書: spec/基本設計/機能設計/040.グループ管理/グループ削除.md は確認できず（not found）。削除仕様は spec/基本設計/機能設計/040.グループ管理/グループ検索.md に内包されている。
  - 実装: functions/groups/deleteGroup/app.py が独立APIとして存在する。
- 指摘内容:
  - 実装が独立している一方で設計書が機能分離されていないため、レビュー/保守時に仕様追跡が難しくなる。
- 期待されるテスト:
  - なし（設計整理事項）。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足:
  - deleteGroup 成功時の副作用（4テーブル削除呼び出し、操作ログ記録）確認が不足。
- 入力値バリエーション不足:
  - searchGroups の groupName 部分一致パターン（前方/中間/後方、一致複数件）未検証。
  - deleteGroup の groupId/times 不正形式ケース未検証。
- 境界値不足:
  - searchGroups の page/pageSize で 0、負数、大値の検証不足。
- 分岐条件不足:
  - deleteGroup で中間処理失敗時の後続処理非実行分岐の検証不足。
- データ入出力不足:
  - deleteGroup で各Repository呼び出し引数の妥当性検証不足。
- 非更新確認不足:
  - deleteGroup 途中失敗時の rollback 後、対象データが削除されていないことの確認不足。
- 異常系不足:
  - searchGroups/deleteGroup ともに入力不正時のエラー分類（4xx/5xx）検証が不足。

## 総評
- High 指摘として、内部例外情報のレスポンス露出はセキュリティリスクが高く、最優先で是正が必要です。
- 次点で、入力不備のエラー分類とページング境界値の実装・テスト不足が品質リスクになっています。
- 修正優先順位は「情報漏えい防止」→「入力バリデーション明確化」→「仕様起点テスト拡充」を推奨します。

## 残留リスク・確認できなかった範囲
- spec/基本設計/機能設計/040.グループ管理/グループ削除.md が確認できず、delete API の独立設計との完全突合は未実施。
- API外の画面挙動（確認ダイアログ表示、0件メッセージ表示、削除後の最終ページ再表示）は本レビュー対象外のため未確認。