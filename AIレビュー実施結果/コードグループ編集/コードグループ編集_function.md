# コードグループ編集 レビュー結果

## 指摘事項

### 1. 認証ユーザーが固定値返却となっており、権限判定の前提を破壊している
- 優先度: High
- 種別: 実装不備
- 対象: [functions/auth/getAuthMe/app.py](functions/auth/getAuthMe/app.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ編集.md](spec/基本設計/機能設計/100.コード定義/コードグループ編集.md) では、ログインユーザー権限に応じて管理連携システム候補および更新可否が変わる前提。
  - 実装: [functions/auth/getAuthMe/app.py](functions/auth/getAuthMe/app.py) でDB取得処理がコメントアウトされ、固定の userId・tenantId・authorities を返却している。
- 指摘内容:
  - 実ユーザーに依存した認可判定が機能せず、コードグループ編集の対象システムチェック・権限チェック結果が実運用と乖離する。
- 期待されるテスト:
  - 認証ユーザーがリクエストごとに異なる場合でも、[functions/users/getUserPermissions/app.py](functions/users/getUserPermissions/app.py) と [functions/codegroups/updateCodeGroup/app.py](functions/codegroups/updateCodeGroup/app.py) の認可分岐が仕様通りになることを検証する統合テスト。

### 2. 更新APIの権限判定が設計条件と不一致で、正当操作を拒否しうる
- 優先度: High
- 種別: 実装不備
- 対象: [functions/codegroups/updateCodeGroup/app.py](functions/codegroups/updateCodeGroup/app.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ編集.md](spec/基本設計/機能設計/100.コード定義/コードグループ編集.md) の権限チェックは「変更前の対象コードグループ管理連携システム」を基準に判定する定義。
  - 実装: [functions/codegroups/updateCodeGroup/app.py](functions/codegroups/updateCodeGroup/app.py) で、変更前の code_admin_system 判定に加えて、更新後 adminSystemId に対する別判定を追加している。
- 指摘内容:
  - 変更前が null のケースでも更新後 adminSystemId を基準に拒否される経路があり、設計上許容される更新をブロックする可能性がある。
- 期待されるテスト:
  - 変更前 null→変更後 systemId の更新に対して、RELATION_SYSTEM_ADMIN 権限あり/なしの両ケースを検証するテスト。
  - 変更前 systemId→変更後 null の更新可否を検証するテスト。

### 3. 取得APIで存在しないコードグループ時の設計エラーコードと動作が一致していない
- 優先度: High
- 種別: 実装不備
- 対象: [functions/codegroups/getCodeGroup/app.py](functions/codegroups/getCodeGroup/app.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ編集.md](spec/基本設計/機能設計/100.コード定義/コードグループ編集.md) の編集モード初期表示では、対象なし時は COM01009 を返す定義。
  - 実装: [functions/codegroups/getCodeGroup/app.py](functions/codegroups/getCodeGroup/app.py) は get_codegroupmaster_by_id の None を明示判定しておらず、後段アクセスで例外化し COM01010 になる経路がある。
- 指摘内容:
  - 設計で要求されるエラー種別と実際の返却が不一致となり、画面側のメッセージ制御と齟齬を生む。
- 期待されるテスト:
  - codeGroupId が存在しない場合に、設計で定義されたエラーコードとレスポンスを返すことを確認するテスト。

### 4. タイムスタンプ比較の型・精度差で正当更新が競合エラー化するリスクがある
- 優先度: High
- 種別: 実装不備
- 対象: [functions/codegroups/getCodeGroup/app.py](functions/codegroups/getCodeGroup/app.py), [functions/codegroups/updateCodeGroup/app.py](functions/codegroups/updateCodeGroup/app.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ編集.md](spec/基本設計/機能設計/100.コード定義/コードグループ編集.md) はタイムスタンプ一致で更新可否を判定する定義。
  - 実装: 取得時は times を文字列化して返却し、更新時は受信値と DB datetime を直接比較している。
- 指摘内容:
  - 文字列化フォーマット差やマイクロ秒精度差で、実際は未競合でも COM01010 相当で弾かれる可能性がある。
- 期待されるテスト:
  - 同値時刻（秒一致・マイクロ秒差あり/なし・タイムゾーン表現差）の更新可否テスト。

### 5. テストが権限分岐の主要ケースを網羅しておらず、設計逸脱を検出できていない
- 優先度: High
- 種別: テスト不足
- 対象: [tests/functions/codegroups/test_codegroups.py](tests/functions/codegroups/test_codegroups.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ編集.md](spec/基本設計/機能設計/100.コード定義/コードグループ編集.md) の更新権限チェック、対象システムチェック。
  - 実装: [functions/codegroups/updateCodeGroup/app.py](functions/codegroups/updateCodeGroup/app.py) の分岐が複数あるが、テストは adminSystem 変更遷移の主要パターンを未検証。
- 指摘内容:
  - 変更前後の管理連携システム組み合わせに対する認可回帰を検知できない。
- 期待されるテスト:
  - null→systemId、systemId→null、systemA→systemB の各遷移で、TENANT_ADMIN/RELATION_SYSTEM_ADMIN/一般権限の判定結果を検証する。

### 6. 更新時のコード更新方式が設計の「全削除後再登録」と異なる
- 優先度: Middle
- 種別: 実装不備
- 対象: [functions/codegroups/updateCodeGroup/app.py](functions/codegroups/updateCodeGroup/app.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ編集.md](spec/基本設計/機能設計/100.コード定義/コードグループ編集.md) は更新時に対象コードを全削除後、入力内容を再登録し codeId 採番する定義。
  - 実装: 既存 codeId は更新、未存在のみ新規作成、最後に差分削除という upsert 方式になっている。
- 指摘内容:
  - 仕様トレース上は設計逸脱であり、codeId のライフサイクル前提を持つ連携先で不整合の原因となる。
- 期待されるテスト:
  - 更新前後で codeId が再採番されること、または設計変更を確定した場合は upsert 方式仕様を明文化し、その仕様テストを追加すること。

### 7. 補助APIの入力フィルタ未適用により画面要件の絞り込み条件を満たせていない
- 優先度: Middle
- 種別: 実装不備
- 対象: [functions/relationSystems/listRelationSystems/app.py](functions/relationSystems/listRelationSystems/app.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ編集.md](spec/基本設計/機能設計/100.コード定義/コードグループ編集.md) は管理連携システム選択を権限条件で扱う前提。
  - 実装: queryStringParameters の systemName・upperOrLower・interfaceName を読み取るが、Repository 呼び出しに渡していない。
- 指摘内容:
  - 画面側で必要となる条件付き一覧の取得要件を満たせず、過剰データ返却や意図しない選択肢表示につながる。
- 期待されるテスト:
  - クエリ条件指定時に返却件数・返却内容が変化することを検証するテスト。

### 8. 権限エラー/業務エラーのHTTPステータスが不統一で、API契約が不安定
- 優先度: Middle
- 種別: 実装不備
- 対象: [functions/codegroups/createCodeGroup/app.py](functions/codegroups/createCodeGroup/app.py), [functions/codegroups/updateCodeGroup/app.py](functions/codegroups/updateCodeGroup/app.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ編集.md](spec/基本設計/機能設計/100.コード定義/コードグループ編集.md) は業務エラーメッセージ定義が中心で、権限不足は COM01001。
  - 実装: create は権限不足を 500 返却、update は同趣旨の分岐で 403 と 500 が混在。
- 指摘内容:
  - クライアント側の共通エラーハンドリングが分岐依存となり、異常時制御が複雑化する。
- 期待されるテスト:
  - 権限不足、入力エラー、競合エラーごとに期待HTTPステータスを固定化し検証するAPIテスト。

### 9. 監査ログ内容の検証テストがなく、要件逸脱を見逃す構成
- 優先度: Middle
- 種別: テスト不足
- 対象: [tests/functions/codegroups/test_codegroups.py](tests/functions/codegroups/test_codegroups.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ編集.md](spec/基本設計/機能設計/100.コード定義/コードグループ編集.md) は操作ログの機能名・操作内容を要求。
  - 実装: [functions/codegroups/createCodeGroup/app.py](functions/codegroups/createCodeGroup/app.py), [functions/codegroups/updateCodeGroup/app.py](functions/codegroups/updateCodeGroup/app.py) で operation を固定文字列にしているが、テストで未検知。
- 指摘内容:
  - 監査証跡の必須情報欠落をCIで検出できない。
- 期待されるテスト:
  - OperationLogRepository への保存引数（function, operation, user_id, tenant_id）を検証するテスト。

### 10. 補助APIテストが異常系を取りこぼしており、画面依存機能の退行検知力が弱い
- 優先度: Low
- 種別: テスト不足
- 対象: [tests/functions/auth/test_getAuthMe.py](tests/functions/auth/test_getAuthMe.py), [tests/functions/relationSystems/test_listRelationSystems.py](tests/functions/relationSystems/test_listRelationSystems.py), [tests/functions/users/test_getUserPermissions.py](tests/functions/users/test_getUserPermissions.py)
- 根拠:
  - 設計書: [spec/基本設計/機能設計/100.コード定義/コードグループ編集.md](spec/基本設計/機能設計/100.コード定義/コードグループ編集.md) の画面初期表示・権限制御は補助APIの安定動作に依存。
  - 実装: getAuthMe の例外系・動的ユーザー取得、listRelationSystems の 401/403 とフィルタ適用の検証が不足。
- 指摘内容:
  - 編集画面の前提APIに回帰が生じても早期検知できない。
- 期待されるテスト:
  - getAuthMe の例外系、listRelationSystems の UnauthorizedException/ForbiddenException、クエリフィルタ適用を追加検証する。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足:
  - 更新時の adminSystemId 遷移パターン（null→値、値→null、値→別値）の成功系検証が不足。
- 入力値バリエーション不足:
  - タイムスタンプ文字列のフォーマット差・精度差（マイクロ秒）の入力バリエーション不足。
- 境界値不足:
  - 権限境界（TENANT_ADMIN/RELATION_SYSTEM_ADMIN/一般）と管理連携システム遷移の組合せ境界不足。
- 分岐条件不足:
  - update の二段権限判定分岐と listRelationSystems のクエリ条件分岐のテスト不足。
- データ入出力不足:
  - 操作ログに記録される operation 文言の内容検証不足。
- 非更新確認不足:
  - 権限不足時に code_group_master/code_master が更新されないことの検証不足。
- 異常系不足:
  - getAuthMe の例外系、listRelationSystems の 401/403、getCodeGroup の未存在ID時の設計エラーコード検証不足。

## 総評
- High 指摘は、認証前提破壊、更新権限判定の設計逸脱、存在しないコードグループ時のエラー不一致、タイムスタンプ競合判定の不安定性であり、いずれも本番障害に直結するためマージ不可レベルです。
- 次点で、更新方式の設計逸脱とステータスコード不統一が API 契約の不安定化を招いています。
- まず High の実装修正と、その分岐を保証するテスト追加を優先し、その後 Middle の仕様整合と契約統一を進める順序が妥当です。

## 残留リスク・確認できなかった範囲
- [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md) の対象機能ごとの厳密な観点割当（どの観点が本機能必須か）の運用基準が文書上で明確でないため、一部は汎用観点として評価。
- [spec/基本設計/機能設計/100.コード定義/コードグループ編集.md](spec/基本設計/機能設計/100.コード定義/コードグループ編集.md) には HTTP ステータスの明示規約が薄く、業務コード優先かステータス優先かは設計確認が必要。
- DB 実挙動（times 桁落ち、flush/commit タイミング）はユニットテスト中心のため、実DB統合観点は未確認。