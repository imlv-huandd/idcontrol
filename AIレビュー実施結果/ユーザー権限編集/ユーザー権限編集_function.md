# ユーザー権限編集機能 レビュー結果

## 指摘事項

### 1. 認証ユーザーを固定値で返却しており、権限制御の前提を満たしていない
- 優先度: High
- 種別: 実装不備
- 対象: functions/auth/getAuthMe/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md の「画面初期表示」「権限設定可能チェック」では、ログインユーザー権限に応じた表示制御・設定可否判定が必須。
  - 実装: functions/auth/getAuthMe/app.py で AuthenticatedUser を固定 UUID/固定 authorities(TENANT_ADMIN) で生成し返却している。
- 指摘内容:
  - 実ユーザー文脈を参照せず固定管理者を返すため、画面側・API側の権限制御前提が崩れ、実運用で過剰権限判定となるリスクがある。

### 2. 連携システムIDの正規化未実装により、権限判定誤りと不正データ保存のリスクがある
- 優先度: High
- 種別: 実装不備
- 対象: functions/users/updateUserPermissions/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md の「更新ボタン押下」では、連携システム管理者/操作者権限は relation_system_id を正しく設定することが要求される。
  - 実装: functions/users/updateUserPermissions/app.py の権限構築処理で relationSystemId を str() のまま systemId に保存しており、コメントで示す「~~~ 以降除去」が実装されていない。
- 指摘内容:
  - relationSystemId の入力形式によっては、ログインユーザーの保有 systemId との比較で偽陰性となり 403 を返す、または DB に期待外の systemId を保存する不整合が起きる。
- 期待されるテスト:
  - relationSystemId が「<id>~~~<name>」形式の場合に <id> のみ保存されること。
  - 同形式入力時でも systemId 比較が正しく通過/拒否されること。

### 3. 権限設定可能チェックの主要分岐が単体テストで未検証
- 優先度: High
- 種別: テスト不足
- 対象: tests/functions/users/test_updateUserPermissions.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md の「権限設定可能チェック（AUT01002）」で、ログインユーザー権限ごとに設定可能範囲が定義されている。
  - 実装: functions/users/updateUserPermissions/app.py には TENANT_ADMIN/MASTER_DATA_ADMIN/RELATION_SYSTEM_ADMIN および systemId 一致判定の複数分岐が存在。
- 指摘内容:
  - 既存テストは正常系・自己編集禁止・ロック競合・ダウンロードのみ拒否が中心で、権限別の許可/拒否分岐（特に systemId 不一致、MASTER_DATA_ADMIN の許可境界）が不足している。
- 期待されるテスト:
  - RELATION_SYSTEM_ADMIN が未保有 systemId を付与しようとして 403 になるケース。
  - MASTER_DATA_ADMIN が許可権限のみ更新でき、TENANT_ADMIN 付与は拒否されるケース。
  - TENANT_ADMIN が全権限を設定可能であるケース。

### 4. 自己編集禁止チェックのエラーコードが設計定義と不一致
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/users/updateUserPermissions/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md の「バリデーション定義」で自己編集禁止チェックは COM01001。
  - 実装: functions/users/updateUserPermissions/app.py では自己編集禁止時に AUT01002 を返却。
- 指摘内容:
  - 仕様メッセージIDと実装の応答コードが不一致であり、フロントのメッセージ制御・運用問い合わせ時の切り分けに影響する。
- 期待されるテスト:
  - 自己編集禁止時に code=COM01001 を返すことを検証するテスト。

### 5. 連携システム一覧APIのテストが仕様観点に対して不足
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/relationSystems/test_listRelationSystems.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/050.権限管理/ユーザー権限編集.md の「画面初期表示」で、連携システム一覧を利用した動的チェックボックス生成が前提。
  - 実装: functions/relationSystems/listRelationSystems/app.py は UnauthorizedException/ForbiddenException を分岐処理しているが、既存テストは 200/500 中心で権限系異常の検証が薄い。
- 指摘内容:
  - 画面初期表示の重要依存APIに対し、認証/認可異常時の振る舞いが未検証で、障害時に画面側の異常系ハンドリング不整合を見逃す可能性がある。
- 期待されるテスト:
  - UnauthorizedException 発生時に 401 を返すケース。
  - ForbiddenException 発生時に 403 を返すケース。

### 6. 一部APIの異常系テストが依存先障害を十分に網羅していない
- 優先度: Low
- 種別: テスト不足
- 対象: tests/functions/users/test_getUserPermissions.py, tests/functions/users/test_getUser.py, tests/functions/auth/test_getAuthMe.py
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md の「異常系」「データ入出力」では、依存先失敗時の挙動確認が求められる。
  - 実装: 各 API は DB/認証関連例外の分岐を持つが、テストは代表ケース中心で、依存先故障の粒度が限定的。
- 指摘内容:
  - 例外分岐が存在しても、実運用で起こる依存先障害（接続失敗、想定外例外）を仕様観点で検証できていない。
- 期待されるテスト:
  - DB接続取得時例外、Repository呼び出し時例外で期待ステータス/エラーコードを返すケース。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足:
  - updateUserPermissions で権限別（TENANT_ADMIN/MASTER_DATA_ADMIN/RELATION_SYSTEM_ADMIN）の正常更新網羅が不足。
- 入力値バリエーション不足:
  - relationSystemId の形式差分（UUIDのみ/「id~~~name」）の検証不足。
- 境界値不足:
  - systemId 一致/不一致境界、複数権限同時指定時の境界分岐不足。
- 分岐条件不足:
  - 権限設定可能チェックの主要分岐（権限種別×対象 systemId）の網羅不足。
- データ入出力不足:
  - systemId の保存値正規化確認、操作ログ内容の妥当性確認が不足。
- 非更新確認不足:
  - 権限拒否/競合発生時に更新・ログ記録が行われないことの確認不足。
- 異常系不足:
  - listRelationSystems の 401/403、各 API の依存先障害系（DB接続/Repository例外）検証不足。

## 総評
High 指摘は、認証文脈の固定化と relationSystemId 正規化未実装であり、権限制御の正当性に直結するため最優先で修正が必要です。
次点として、自己編集禁止のエラーコード不一致を是正し、仕様準拠の応答契約を揃えるべきです。
テストは主要正常系はあるものの、権限分岐と依存先障害の観点が不足しており、回帰防止力が不十分です。

## 残留リスク・確認できなかった範囲
- spec/詳細設計 および spec/概要 配下の当該機能に直接対応する記述は確認対象として追加抽出したが、ユーザー指定設計書以外の必須要件の明示範囲は限定的で、I/F定義（クエリパラメータ名、エラーコード体系）との完全突合までは未完了。
