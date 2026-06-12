# グループ権限編集 レビュー結果

## 指摘事項

### 1. 認証ユーザーを固定値で返却しており、権限制御仕様を満たせない
- 優先度: High
- 種別: 実装不備
- 対象: functions/auth/getAuthMe/app.py, shared/core/common.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/グループ権限編集.md](spec/基本設計/機能設計/050.権限管理/グループ権限編集.md) の「画面初期表示」で、ログインユーザー権限に応じた表示制御を要求。
  - 実装: [functions/auth/getAuthMe/app.py](functions/auth/getAuthMe/app.py#L18) で固定ユーザーを返却。[shared/core/common.py](shared/core/common.py#L31) の get_current_user も固定権限(TENANT_ADMIN)を返却。
- 指摘内容:
  - 実ユーザーの認証・認可情報を利用せず固定権限で処理されるため、画面の権限表示制御・更新可否判定が常に誤る可能性がある。
- 期待されるテスト:
  - TENANT_ADMIN / MASTER_DATA_ADMIN / RELATION_SYSTEM_ADMIN / 権限なし の各ログインユーザーで、表示制御と更新可否判定が仕様どおり分岐するテスト。

### 2. 連携システム権限判定で未定義変数参照が発生しうる
- 優先度: High
- 種別: 実装不備
- 対象: functions/groups/updateGroupPermissions/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/グループ権限編集.md](spec/基本設計/機能設計/050.権限管理/グループ権限編集.md) の「権限設定可能チェック」で、連携システム権限は保有システム範囲で判定する要求。
  - 実装: [functions/groups/updateGroupPermissions/app.py](functions/groups/updateGroupPermissions/app.py#L103) で login_user_system_ids を条件付き定義し、[functions/groups/updateGroupPermissions/app.py](functions/groups/updateGroupPermissions/app.py#L154) で無条件参照している。
- 指摘内容:
  - user_authority レコードが取得できないケースで RELATION_SYSTEM_ADMIN 判定に入ると、UnboundLocalError で 500 応答となるリスクがある。
- 期待されるテスト:
  - ログインユーザーが RELATION_SYSTEM_ADMIN だが user_authority が None のケースで、500 にならず仕様どおり 403 または業務エラーになることを確認するテスト。

### 3. グループ未存在時のエラー処理が設計と不整合で 500 化している
- 優先度: High
- 種別: 実装不備
- 対象: functions/groups/getGroup/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/グループ権限編集.md](spec/基本設計/機能設計/050.権限管理/グループ権限編集.md) の「画面初期表示」で、グループ不存在時は COM01009 を表示して遷移する仕様。
  - 実装: [functions/groups/getGroup/app.py](functions/groups/getGroup/app.py#L38) で ValueError("COM01003",...) を送出し、同関数内の内側 except で [functions/groups/getGroup/app.py](functions/groups/getGroup/app.py#L70) 500 応答化している。
- 指摘内容:
  - 不存在ケースが業務エラーとして扱われず内部エラー化されるため、クライアント側の想定ハンドリングと一致しない。
- 期待されるテスト:
  - groupId 不存在時に業務エラーコード(COM01009)と想定 HTTP ステータスで返却されることを確認するテスト。

### 4. バリデーションエラーコードが設計定義と不一致
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/groups/updateGroupPermissions/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/グループ権限編集.md](spec/基本設計/機能設計/050.権限管理/グループ権限編集.md) の「バリデーション定義」で 権限選択チェック=AUT02001、権限設定可能チェック=AUT02002。
  - 実装: [functions/groups/updateGroupPermissions/app.py](functions/groups/updateGroupPermissions/app.py#L90) で AUT01001、[functions/groups/updateGroupPermissions/app.py](functions/groups/updateGroupPermissions/app.py#L140) などで AUT01002 を返却。
- 指摘内容:
  - 画面メッセージ制御や共通エラー処理がメッセージIDに依存している場合、誤表示・誤遷移の要因になる。
- 期待されるテスト:
  - 各バリデーション失敗時に設計どおりのメッセージIDを返却することを確認する API テスト。

### 5. 操作ログ文言フォーマットが設計記載と一致していない
- 優先度: Middle
- 種別: 設計確認事項
- 対象: functions/groups/updateGroupPermissions/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/グループ権限編集.md](spec/基本設計/機能設計/050.権限管理/グループ権限編集.md) の「更新ボタン押下」で操作内容は「グループ権限編集：[グループ名]  権限：[設定した権限...]」。
  - 実装: [functions/groups/updateGroupPermissions/app.py](functions/groups/updateGroupPermissions/app.py#L195) で「グループ権限編集：{グループ名} - 権限：...」を記録。
- 指摘内容:
  - 監査要件が文言一致を前提にしている場合、検索条件や監査証跡の抽出精度に影響する可能性がある。
- 期待されるテスト:
  - 操作ログの function と operation が仕様フォーマットに一致することを確認するテスト。

### 6. 単体テストが主要仕様分岐を網羅できていない
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/groups/test_updateGroupPermissions.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/グループ権限編集.md](spec/基本設計/機能設計/050.権限管理/グループ権限編集.md) の「サーバーサイドバリデーション」「グループ権限マスター更新」「操作ログ」。
  - 実装: [tests/functions/groups/test_updateGroupPermissions.py](tests/functions/groups/test_updateGroupPermissions.py#L36) から [tests/functions/groups/test_updateGroupPermissions.py](tests/functions/groups/test_updateGroupPermissions.py#L181) まで、404/409/400/403/200 の一部ケースはあるが、RELATION_SYSTEM_ADMIN の systemId 範囲チェック、DOWNLOAD 自動付与、ログ内容、commit 失敗時の挙動が未検証。
- 指摘内容:
  - 現状テストでは仕様上重要な分岐と副作用(DB更新内容、ログ整合)を担保できず、回帰不具合の検知力が不足している。
- 期待されるテスト:
  - RELATION_SYSTEM_ADMIN が保有外 systemId を設定したとき 403。
  - 管理系権限選択時に DOWNLOAD が自動付与されること。
  - 更新成功時に group_authority 更新内容(JSON)と operation log 内容が仕様どおりであること。
  - operation log 作成失敗時に commit されないこと。

### 7. レスポンススキーマ境界値・不正入力の検証が不足
- 優先度: Low
- 種別: テスト不足
- 対象: tests/shared/schemas/test_groupSchemas.py
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md) の単体テスト観点（入力値バリエーション、境界値、異常系）。
  - 実装: [tests/shared/schemas/test_groupSchemas.py](tests/shared/schemas/test_groupSchemas.py#L17) から [tests/shared/schemas/test_groupSchemas.py](tests/shared/schemas/test_groupSchemas.py#L84) は基本生成中心で、times 不正形式、authority 定義外値、relationSystemId 不正形式の異常系検証が不足。
- 指摘内容:
  - スキーマ起因の 400 系不具合を早期検知しにくく、API ハンドラでの例外経路に依存した検知になる。
- 期待されるテスト:
  - times 不正フォーマット、authority 定義外値、relationSystemId 不正UUIDでバリデーションエラーになることのテスト。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足:
  - 更新成功時の DB 更新内容、レスポンス内容、操作ログ内容の完全検証が不足。
- 入力値バリエーション不足:
  - authority の組み合わせ別(管理者系、閲覧者、DOWNLOAD単独)と relationSystemId 保有内外の組み合わせ検証が不足。
- 境界値不足:
  - times の精度差分、permissions 件数上限/0件、UUID 形式境界の検証が不足。
- 分岐条件不足:
  - TENANT_ADMIN / MASTER_DATA_ADMIN / RELATION_SYSTEM_ADMIN / 一般ユーザーの権限分岐網羅が不足。
- データ入出力不足:
  - group_authority.authorities の保存JSON内容と返却レスポンス整合の検証が不足。
- 非更新確認不足:
  - 失敗系(409, 403, 400, ログ記録失敗)で更新されないことの確認が不足。
- 異常系不足:
  - 依存先失敗(Repository例外、OperationLog失敗、DB commit失敗)時の戻り値とロールバック確認が不足。

## 総評
- High 指摘は、認証情報の固定化、未定義変数参照による500化、グループ不存在時のエラー契約不整合の3点で、いずれも機能成立性に直結します。
- 特に権限制御の根幹が固定ユーザー前提になっており、設計の表示制御・設定可能チェックを満たせません。
- 修正優先順位は、1) 認証/権限制御の実ユーザー化、2) updateGroupPermissions の例外経路修正、3) getGroup の業務エラー整合、4) テスト拡充です。

## 残留リスク・確認できなかった範囲
- spec/050.テスト/テスト観点.md の本文全量は参照できず、SubAgent要約に基づく照合を含みます。
- フロントエンド側のメッセージID依存実装(遷移・表示制御)は未照合です。
- DBスキーマ制約(ユニーク制約、JSONB制約、トランザクション分離レベル)は未確認です.
