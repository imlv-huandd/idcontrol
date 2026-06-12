# グループ編集 レビュー結果

## 指摘事項

### 1. 更新時の重複チェックがテナント境界を越えて判定される
- 優先度: High
- 種別: 実装不備
- 対象: shared/repositories/groupmasterRepository.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/040.グループ管理/グループ編集.md（重複チェックは同一テナント内の同名グループを対象に判定する前提）
  - 実装: shared/repositories/groupmasterRepository.py:260 付近の is_group_name_duplicate が user_group_name 条件のみで、tenant_id を WHERE 条件に含めていない
- 指摘内容:
  - updateGroup の重複判定が他テナントの同名グループまで重複扱いする可能性があり、正常更新を誤って拒否する。マルチテナント分離要件に反する。
- 期待されるテスト:
  - 異なる tenant_id で同名グループが存在しても更新が拒否されないこと
  - 同一 tenant_id で同名グループが存在する場合のみ拒否されること

### 2. getGroupPermissions が存在しない属性を参照して 500 になるリスク
- 優先度: High
- 種別: 実装不備
- 対象: functions/groups/getGroupPermissions/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/040.グループ管理/グループ編集.md（編集モード初期表示でグループ情報と所属ユーザーを正常取得する要件）
  - 実装: functions/groups/getGroupPermissions/app.py:45 付近で group.group_id を参照。一方 shared/models/groupMaster.py:16 は user_group_id を定義
- 指摘内容:
  - モデル属性名不整合により実行時例外が発生し、正常系が 500 応答に落ちる可能性が高い。
- 期待されるテスト:
  - getGroupPermissions 正常系で groupId が user_group_id 由来で返ること
  - 属性参照ミスを検知できる失敗テスト（回帰防止）

### 3. getAuthMe が固定ユーザー（TENANT_ADMIN）を返却しており認可検証が成立しない
- 優先度: High
- 種別: 実装不備
- 対象: functions/auth/getAuthMe/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/040.グループ管理/グループ編集.md（権限チェックはログインユーザーの実際の権限に基づく要件）
  - 実装: functions/auth/getAuthMe/app.py:19-29 で DB 参照がコメントアウトされ、固定 UUID・固定 authorities=["TENANT_ADMIN"] を返却
- 指摘内容:
  - 常に管理者相当のユーザー情報を返す実装は、権限制御の前提を崩し、実運用時の認可不備を見逃す。
- 期待されるテスト:
  - 権限別（TENANT_ADMIN / 非管理者 / グループ管理者）のユーザー情報がトークン・DB整合で返ること
  - 不正トークン時の 401 応答

### 4. グループ不存在時のエラー仕様（メッセージID・HTTP）の設計乖離
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/groups/getGroup/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/040.グループ管理/グループ編集.md:126-130（グループ未存在時は COM01009 を表示して遷移）
  - 実装: functions/groups/getGroup/app.py:30 付近で ValueError("COM01003", "該当データがありません。") を送出し、内側 except で 500 化
- 指摘内容:
  - 画面仕様のエラーコードと不一致で、業務エラーがシステムエラーとして扱われる。フロントの分岐制御やメッセージ整合に影響する。
- 期待されるテスト:
  - グループ未存在時に設計どおりのメッセージIDとステータスで返ること

### 5. updateGroup の削除結果判定がリポジトリ戻り値仕様に依存し過ぎている
- 優先度: Middle
- 種別: 設計確認事項
- 対象: functions/groups/updateGroup/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/040.グループ管理/グループ編集.md（所属ユーザー更新の完全性要件）
  - 実装: functions/groups/updateGroup/app.py:193-206 で delete_group_join_user_by_tenant_id_group_id_user_id の戻り値を真偽判定。shared/repositories/groupJoinUserRepository.py:18-25 は明示 return なし
- 指摘内容:
  - 削除成否の判定契約が曖昧で、実装変更時に誤判定しやすい。現状は「戻り値なし」を前提に動くため、失敗検知の堅牢性が低い。
- 期待されるテスト:
  - 削除件数 0 件/1 件/例外時の分岐を明示検証すること
  - リポジトリ契約（戻り値）変更時に破綻しないテスト

### 6. getGroupPermissions の単体テストが存在せず、主要分岐未検証
- 優先度: Middle
- 種別: テスト不足
- 対象: functions/groups/getGroupPermissions/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/040.グループ管理/グループ編集.md（編集モード初期表示要件）
  - 実装: functions/groups/getGroupPermissions/app.py（正常系・未存在系・例外系の分岐あり）
- 指摘内容:
  - 対応テストファイルが確認できず、属性参照ミスやエラー応答仕様の逸脱を検知できない。
- 期待されるテスト:
  - 正常系（200、groupId、members）
  - グループ未存在時の業務エラー応答
  - DB例外時の 500 応答

### 7. listGroupMembers テストが 500 を許容しており仕様検証として弱い
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/groups/test_listGroupMembers.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/040.グループ管理/グループ編集.md（編集モードで所属ユーザー一覧を表示）
  - 実装: tests/functions/groups/test_listGroupMembers.py:24-29 付近で statusCode in (200, 500) を許容
- 指摘内容:
  - 正常系テストでシステムエラーを許容しており、回帰時に不具合を見逃す。期待値が仕様ベースになっていない。
- 期待されるテスト:
  - 正常系は 200 固定アサート
  - groupId と members の内容（件数・主要項目）を具体値で検証

### 8. create/add/update 系のテストでデータ入出力と境界値観点が不足
- 優先度: Low
- 種別: テスト不足
- 対象: tests/functions/groups/test_createGroupWithMembers.py, tests/functions/groups/test_addGroupMember.py, tests/functions/groups/test_updateGroup.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/040.グループ管理/グループ編集.md（登録・更新・所属ユーザー更新・操作ログ要件）
  - 実装: 既存テストは主にステータス検証中心で、複数件・境界値・副作用（登録件数、操作ログ）の検証が限定的
- 指摘内容:
  - C1 的な分岐通過は一部満たしても、仕様観点（データ整合、副作用、境界値）として不足が残る。
- 期待されるテスト:
  - グループ名 1/100/101 文字境界
  - メンバー 0/1/複数件、重複混在、存在しないユーザー混在
  - OperationLog の function/operation 内容検証

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足
  - getGroupPermissions の正常系テストが未整備
  - listGroupMembers 正常系が 500 許容で厳密性不足
- 入力値バリエーション不足
  - create/add/update で複数メンバー混在ケース、重複混在ケースの検証が不足
- 境界値不足
  - グループ名 100/101 文字、メンバー件数境界の検証が不足
- 分岐条件不足
  - getGroupPermissions の未存在分岐・例外分岐の未検証
  - updateGroup の削除成否分岐の契約依存が強く、回帰観点が不足
- データ入出力不足
  - createGroupWithMembers の権限初期登録、OperationLog 記録内容の検証が不足
- 非更新確認不足
  - 他テナントデータに影響しないこと（重複判定の tenant 分離）のテスト不足
- 異常系不足
  - getGroup/getGroupPermissions の業務エラーとシステムエラーの切り分け検証不足

## 総評
High 指摘は、マルチテナント分離違反リスク、属性参照不整合、認可前提を崩す固定認証応答の3点で、いずれも先に解消すべきです。
次に、エラー仕様不整合と getGroupPermissions 未テストを優先して、設計準拠性と回帰検知力を確保する必要があります。
最後に、境界値・副作用検証を補強し、仕様ベースの単体テスト品質へ引き上げるのが妥当です。

## 残留リスク・確認できなかった範囲
- spec/050.テスト/テスト観点.md は観点カテゴリを確認済みだが、機能別の期待テストケース粒度（ケース一覧）までは記載が薄く、詳細なケース網羅の最終判定には追加のテスト設計書が必要
- 外部連携（認証基盤、IDプロバイダ）との実接続を伴う統合試験観点は本レビュー範囲外
