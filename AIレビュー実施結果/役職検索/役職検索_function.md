# 役職検索 レビュー結果

## 指摘事項

### 1. CSV出力のヘッダー順とデータ列順が不一致で、仕様順とも乖離している
- 優先度: High
- 種別: 実装不備
- 対象: functions/organizationPositions/searchOrgPositionsCsv/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md（CSV出力項目順は「テナント役職ID, 有効開始日, 有効終了日, 役職名称, 廃止, 属性」）
  - 実装: functions/organizationPositions/searchOrgPositionsCsv/app.py（ヘッダーは「ID, 名称, 開始日, 終了日, 廃止」、行データは「ID, 開始日, 終了日, 名称, 廃止」でヘッダーと行の並び自体も不一致）
- 指摘内容:
  - CSVの列順が設計書と一致していないうえ、ヘッダー順と行データ順も一致していません。利用側で列解釈を誤り、誤データ取り込みや連携障害を起こすリスクがあります。
- 期待されるテスト:
  - CSVヘッダー順と行データ順が完全一致すること
  - 設計書記載の列順（ID, 開始日, 終了日, 名称, 廃止, 属性）を満たすこと
  - 代表データ1件で列マッピングを厳密比較すること

### 2. CSV出力APIの単体テストが欠落しており、権限・形式・分岐が未検証
- 優先度: High
- 種別: テスト不足
- 対象: tests/functions/organizationPositions/test_organizationPositions.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md（CSV出力の権限チェック、0件時COM01003、UTF-8/CRLF、出力順・出力項目の仕様）
  - 実装: functions/organizationPositions/searchOrgPositionsCsv/app.py（CSV出力ロジック・権限制御・属性展開の分岐あり）
- 指摘内容:
  - 既存テストには searchOrgPositionsCsv を対象とするテストが存在せず、重要分岐が一切保証されていません。仕様逸脱が混入しても検知不能です。
- 期待されるテスト:
  - 権限あり/なし（COM01001）
  - 検索0件（COM01003）
  - UTF-8 BOM・CRLF・QUOTE_ALL の出力形式
  - includeAbolished true/false 分岐
  - 属性型（DATE/DATETIME/その他）変換と空値処理
  - 属性列順（表示順）

### 3. 廃止対象チェックが設計の抽出条件を満たしているか実装上保証されていない
- 優先度: High
- 種別: 実装不備
- 対象: functions/organizationPositions/abolishOrgPosition/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md（廃止対象取得条件は「役職ID一致 かつ 有効終了日=9999/12/31 かつ 廃止=false」）
  - 実装: functions/organizationPositions/abolishOrgPosition/app.py（get_organization_position_by_org_position_id 呼び出し結果の存在確認のみ。対象条件の明示チェックなし）
- 指摘内容:
  - 実装上、廃止対象の厳密条件がコードで確認できません。Repository側の実装依存となっており、仕様条件から外れたデータを更新するリスクがあります。
- 期待されるテスト:
  - 既に廃止済みデータを対象にした場合に更新されないこと
  - 有効終了日が 9999/12/31 以外のデータは廃止対象外となること
  - 更新件数0時のエラー応答（COM01010）を確認すること

### 4. 認証APIが固定ユーザーを返すスタブ実装のまま
- 優先度: High
- 種別: 実装不備
- 対象: functions/auth/getAuthMe/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md（権限に応じた機能制御が前提）
  - 実装: functions/auth/getAuthMe/app.py（固定UUID/loginId/authorities を返却）
- 指摘内容:
  - 実ユーザー文脈に基づく認証結果を返しておらず、権限制御の前提が崩れます。認可逸脱や表示制御不整合のリスクが高い状態です。
- 期待されるテスト:
  - トークン/コンテキストに応じて userId, tenantId, authorities が変わること
  - 認証失敗時のエラーハンドリング（401系）

### 5. 役職検索の並び順が設計指定（序列昇順→テナント役職ID昇順）と不一致
- 優先度: Middle
- 種別: 実装不備
- 対象: shared/repositories/organizationPositionRepository.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md（検索結果エリアのソート順を明示）
  - 実装: shared/repositories/organizationPositionRepository.py（tenant_org_position_id.asc() のみ）
- 指摘内容:
  - 序列を加味しない並び順のため、画面表示順が仕様と異なる可能性があります。
- 期待されるテスト:
  - 序列が異なる複数レコードを用意し、序列優先で並ぶこと
  - 序列同値時にテナント役職ID昇順で並ぶこと

### 6. テナント取得APIでログインユーザー情報の利用が無効化されている
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/tenants/getTenant/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md（テナント設定に基づく表示制御）
  - 実装: functions/tenants/getTenant/app.py（get_current_user と tenantId 取得がコメントアウト）
- 指摘内容:
  - 呼び出しパラメータ tenantId に依存する実装で、ログインテナントとの整合確認が見えません。テナント越境参照の抑止が不明確です。
- 期待されるテスト:
  - ログインテナントと異なる tenantId 指定時の拒否
  - 正常系でログインテナント情報を取得できること

### 7. 役職検索APIのテストで主要分岐（includeAbolished、ページング境界、権限例外）が不足
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/organizationPositions/test_organizationPositions.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md（廃止済含有条件、ページング、検索基準日の仕様）
  - 実装: functions/organizationPositions/searchOrgPositions/app.py（includeAbolished, page, pageSize, UnauthorizedException/ForbiddenException 分岐）
- 指摘内容:
  - 現状は基本正常系と例外1件が中心で、仕様で重要な分岐が不足しています。
- 期待されるテスト:
  - includeAbolished=true/false で repository 呼び出し条件が変わること
  - page/pageSize の境界値（1, 最大, 不正値）
  - UnauthorizedException/ForbiddenException 返却確認

### 8. 廃止処理の境界条件テストが不足（廃止日境界・非更新確認）
- 優先度: Low
- 種別: テスト不足
- 対象: tests/functions/organizationPositions/test_organizationPositions.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/役職検索.md（廃止対象チェック、更新失敗時の取り扱い）
  - 実装: functions/organizationPositions/abolishOrgPosition/app.py（start_date >= abolishDate の分岐、update結果0件分岐）
- 指摘内容:
  - 一部テストはあるものの、境界値（start_date と abolishDate が同日）や他レコード非更新の確認が不足しています。
- 期待されるテスト:
  - start_date == abolishDate の境界ケース
  - 対象外レコードが更新されないこと

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足:
  - searchOrgPositionsCsv の正常系（権限あり、データあり、CSV生成成功）
- 入力値バリエーション不足:
  - searchOrgPositions の page/pageSize/baseDate 組み合わせ
  - searchOrgPositionsCsv の検索条件組み合わせ
- 境界値不足:
  - 廃止日境界（start_date と同日/前日）
  - ページング境界（最小/最大）
- 分岐条件不足:
  - includeAbolished true/false
  - searchOrgPositions の 401/403 分岐
- データ入出力不足:
  - CSVのヘッダー順/列順/文字コード/改行/属性型変換
- 非更新確認不足:
  - 廃止処理で対象外レコードが更新されないこと
- 異常系不足:
  - CSV 0件時COM01003
  - CSV権限なしCOM01001

## 総評
現状の最大リスクは、CSV出力機能に対するテスト欠落と、CSV列順不整合です。設計どおりの出力保証がなく、外部連携に直接影響します。
次に、廃止対象条件と認証・テナント文脈の扱いが実装上不明確で、認可/整合性リスクが残っています。
修正優先順位は、CSV実装整合性とテスト整備を最優先、その後に廃止対象条件・認証/テナント文脈の明確化が妥当です。

## 残留リスク・確認できなかった範囲
- Repositoryメソッド内部の全分岐（特に廃止対象抽出条件）を仕様レベルで完全に保証しているかは、関連SQL/ORM実装の詳細確認が追加で必要です。
- 役職検索画面から呼ばれるフロントエンド側表示制御（CSVボタン表示条件、エラーダイアログ文言）までは本レビュー範囲外です。
- getAuthMe/getTenant の最終仕様（スタブ継続可否）が設計書上で明示されていないため、要件確定が必要です。