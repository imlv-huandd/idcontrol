# コード変換グループ編集 レビュー結果

## 指摘事項

### 1. 変換値重複時のエラーコード・メッセージが設計不一致
- 優先度: High
- 種別: 実装不備
- 対象: functions/codeConversionGroups/createCodeConversionGroup/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md - バリデーション定義「変換元値重複チェック」は COD04002
  - 実装: functions/codeConversionGroups/createCodeConversionGroup/app.py - beforeValue/afterValue 重複で COD04001 と「コード変換グループ名称」向け文言を返却
- 指摘内容:
  - 変換値重複の業務エラーで、グループ名重複用コード・文言を返しており、利用者向けエラー内容と設計定義が一致していない。
- 期待されるテスト:
  - beforeValue 重複時に COD04002 が返ること。
  - afterValue 重複時の期待仕様（対象外なら許容、対象内なら専用コード）を明示した上でコード値を検証すること。

### 2. 更新時の楽観ロック比較が型不一致で誤判定リスク
- 優先度: High
- 種別: 実装不備
- 対象: functions/codeConversionGroups/updateCodeConversionGroup/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md - 更新時は取得時タイムスタンプ一致を更新条件にする
  - 実装: functions/codeConversionGroups/updateCodeConversionGroup/app.py - リクエスト times（文字列）と DB の times（datetime）を直接比較
- 指摘内容:
  - 型・精度差により、実データが未更新でも不一致扱いになる可能性がある。仕様上の排他制御が不安定。
- 期待されるテスト:
  - 同一時刻（ISO 8601）で更新成功するケース。
  - マイクロ秒差・タイムゾーン差で不一致になるケース。
  - 不正な times 形式での入力エラーケース。

### 3. 認証 API が固定ユーザーを返却しており本番仕様逸脱
- 優先度: High
- 種別: 実装不備
- 対象: functions/auth/getAuthMe/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md - 権限に応じた UI 制御（管理連携システム候補・更新可否）を前提
  - 実装: functions/auth/getAuthMe/app.py - DB 参照処理がコメントアウトされ、固定 UUID/loginId/authorities を返却
- 指摘内容:
  - 画面側の権限制御が実ユーザーに追従せず、認可設計の検証不能。誤った権限で編集操作が可能になるリスクがある。
- 期待されるテスト:
  - 実ユーザーの authority 差分（TENANT_ADMIN / RELATION_SYSTEM_ADMIN / 一般）でレスポンスが切り替わること。

### 4. 操作ログがプレースホルダ固定で監査要件を満たさない
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/codeConversionGroups/createCodeConversionGroup/app.py, functions/codeConversionGroups/updateCodeConversionGroup/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md - 操作内容「...：[コード変換グループ名]」を記録
  - 実装: 両 API の operationLog で文字列 "[コード変換グループ名]" をそのまま保存
- 指摘内容:
  - 実際の対象名が監査ログに残らず、追跡性が不足している。
- 期待されるテスト:
  - 登録/更新後のログ operation に実入力のグループ名が含まれること。

### 5. 更新 API の権限制御が複雑で仕様トレースが不十分
- 優先度: Middle
- 種別: 設計確認事項
- 対象: functions/codeConversionGroups/updateCodeConversionGroup/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md - 権限チェックは「変更前対象」と「変更後対象」の両観点
  - 実装: 変更前 convert_admin_system と変更後 admin_system_id を別条件で判定し、同等の拒否処理が重複
- 指摘内容:
  - 条件の重複で境界ケース（変更前は許可、変更後は不許可等）の期待動作が読み取りにくく、仕様逸脱を埋め込みやすい構造。
- 期待されるテスト:
  - 変更前許可/変更後不許可、変更前不許可/変更後許可、両方不許可の組み合わせを網羅。

### 6. 取得 API の入力必須チェックが不足
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/codeConversionGroups/getCodeConversionGroup/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md - 編集モードはコード変換グループID指定で取得
  - 実装: queryStringParameters から codeConversionGroupId を取得するが、空値・形式チェックなしで repository 呼び出し
- 指摘内容:
  - パラメータ欠落時の応答が 400 ではなく 500 側に寄る可能性があり、API 契約が曖昧。
- 期待されるテスト:
  - codeConversionGroupId 未指定、空文字、UUID 不正形式の入力で期待エラーを返すこと。

### 7. テストがエラー詳細ではなくステータス中心で仕様検証が弱い
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/codeConversionGroups/test_codeConversionGroups.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/100.コード定義/コード変換グループ編集.md - メッセージ ID 単位でバリデーション仕様を定義
  - 実装: create/update で多数の業務コードを返却
- 指摘内容:
  - 一部ケースで statusCode のみ確認し、code/message の妥当性を十分に検証していないため、誤コード返却を見逃す。
- 期待されるテスト:
  - COM/COD 系エラーごとに code と message を厳密検証。

### 8. テスト観点に対する不足（境界値・分岐・非更新確認）
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/codeConversionGroups/test_codeConversionGroups.py
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md - 単体テスト観点（境界値、分岐条件、非更新確認、依存先失敗）
  - 実装: update で convert master 差分削除・権限分岐・排他制御・ログ記録など複数分岐あり
- 指摘内容:
  - 権限分岐の組み合わせ、times の厳密比較、削除を伴う更新時の非更新確認、DB 失敗時ロールバック確認のテストが不足。
- 期待されるテスト:
  - 正常系不足: 権限別（TENANT_ADMIN/RELATION_SYSTEM_ADMIN）成功パターン。
  - 境界値不足: 100文字ちょうど、101文字、空文字。
  - 分岐条件不足: 管理連携システム変更の前後組み合わせ。
  - 非更新確認不足: エラー時に convert_group_master / convert_master / operation_log が更新されないこと。
  - 異常系不足: repository 例外時のレスポンスコード・ロールバック。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足
  - 権限パターン別の create/update 成功ケース（管理連携システム指定有無を含む）が不足。
- 入力値バリエーション不足
  - codeConversionGroupId/times の形式異常、adminSystemId の不正値ケースが不足。
- 境界値不足
  - 100/101 文字境界、変換行数 1 件/多数件の境界が不足。
- 分岐条件不足
  - 変更前権限と変更後権限の組み合わせ分岐が不足。
- データ入出力不足
  - 更新後レスポンスの conversions 内容・順序・ID 反映の検証が不足。
- 非更新確認不足
  - バリデーション/権限/排他エラー時に DB 更新・ログ記録が発生しないことの検証が不足。
- 異常系不足
  - Repository 例外、DB 接続失敗、操作ログ失敗時の扱い（ロールバック含む）の検証が不足。

## 総評
High 指摘は、エラーコード不整合・排他制御の型不一致・認証スタブ固定の3点で、いずれも本番品質と監査性に直接影響する。
特に update の排他制御と create のバリデーション応答不整合は、ユーザー操作上の誤判定や誤誘導につながるため優先修正が必要。
次点として、操作ログの具体値記録と権限分岐テストの補強を進めることで、設計適合性と回帰耐性が向上する。

## 残留リスク・確認できなかった範囲
- spec/050.テスト/テスト観点.md の機能別マッピング粒度が粗く、コード変換グループ編集専用の期待ケース一覧は別資料依存の可能性がある。
- tests/functions/codeConversionGroups/test_codeConversionGroups.py 以外の統合/E2E テスト有無は本レビュー範囲外。
- getAuthMe の正式仕様（スタブ継続か本実装化か）が設計書単体では明確でないため、要件確認が必要。