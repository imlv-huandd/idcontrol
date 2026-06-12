# 連携システム画面 API レビュー結果

## 指摘事項

### 1. 一覧取得で権限範囲の絞り込みが未実装
- 優先度: High
- 種別: 実装不備
- 対象: functions/relationSystems/listRelationSystems/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/090.連携システム/連携システム.md（初期表示: テナント管理者は全システム、連携システム管理者/操作者は権限を持つシステムのみ表示）
  - 実装: functions/relationSystems/listRelationSystems/app.py#L33-L37 では tenant_id での取得のみを実施。shared/repositories/systemMasterRepository.py#L67-L83 でも権限マスターを用いた絞り込みが確認できない。
- 指摘内容:
  - 一覧APIがログインユーザーのシステム権限を考慮せず、同一テナントの全システムを返却する実装になっている。設計上の参照可能範囲制御に対して逸脱しており、過剰参照のリスクがある。
- 期待されるテスト:
  - テナント管理者、連携システム管理者、連携システム操作者の3パターンで返却件数・返却対象が設計通りに分岐することを確認する。

### 2. システム削除時に権限マスタ削除が未実装
- 優先度: High
- 種別: 実装不備
- 対象: shared/services/relationSystemService.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/090.連携システム/連携システム.md（システム削除時にグループ権限マスター、ユーザー権限マスターを削除対象とする）
  - 実装: shared/services/relationSystemService.py#L82-L84 に TODO コメントが残存し、該当削除処理が未実装。
- 指摘内容:
  - 設計で必須の削除対象が漏れているため、システム削除後に不要権限データが残存し、データ整合性・権限制御の不整合を招く。
- 期待されるテスト:
  - システム削除時に権限マスタ（グループ/ユーザー）の対象レコードが削除されること、対象外データが削除されないことを検証する。

### 3. 対象6APIのうち5APIで単体テストが未作成
- 優先度: Middle
- 種別: テスト不足
- 対象: functions/relationSystems/deleteRelationSystem/app.py, functions/upperInterfaces/deleteRelationSystemUpperInterface/app.py, functions/lowerInterfaces/deleteRelationSystemLowerInterface/app.py, functions/relationSystems/createRelationSystem/app.py, functions/relationSystems/updateRelationSystem/app.py
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md（正常系、入力値バリエーション、境界値、分岐条件、データ入出力、非更新確認、異常系を網羅）
  - 実装: tests/functions/relationSystems/test_listRelationSystems.py と tests/functions/relationSystems/test_relationSystems.py のみ存在。削除/作成/更新/upper削除/lower削除の各APIに対応するテストファイルが確認できない。
- 指摘内容:
  - 重要処理（削除バリデーション、権限判定、重複チェック、更新競合、依存削除）の回帰防止ができない状態で、仕様逸脱や不具合を検知しにくい。
- 期待されるテスト:
  - 各APIごとに正常系・異常系（401/403/業務エラー/予期せぬ例外）・分岐条件（参照あり、権限不足、重複、更新0件）・データ入出力（DB反映、操作ログ記録）を追加する。

### 4. 一覧APIの既存テストが仕様観点を十分に検証できていない
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/relationSystems/test_listRelationSystems.py, tests/functions/relationSystems/test_relationSystems.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/090.連携システム/連携システム.md（初期表示時の権限別表示制御、一覧表示順、例外時制御）
  - 実装: tests/functions/relationSystems/test_listRelationSystems.py#L91-L102 は汎用Exceptionの500のみ検証。401/403分岐、権限別返却、表示順（システムID昇順、連携元→連携先→ID昇順）の検証が不足。tests/functions/relationSystems/test_relationSystems.py は list の重複検証が中心で仕様観点が薄い。
- 指摘内容:
  - 仕様に対する期待値（権限・並び順・エラー分類）より、実装の一部経路のみを通す内容に偏っており、仕様準拠の保証が弱い。
- 期待されるテスト:
  - 権限別表示、並び順、Unauthorized/Forbidden/ServiceException の応答コードとメッセージ、クエリ条件の組み合わせ時の結果整合を検証する。

### 5. システム更新APIの操作ログ要件が設計書上で不明確
- 優先度: Low
- 種別: 設計確認事項
- 対象: functions/relationSystems/updateRelationSystem/app.py, shared/services/relationSystemService.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/090.連携システム/連携システム.md（システム削除・インターフェイス削除では操作ログ登録が明示）
  - 実装: shared/services/relationSystemService.py では削除系の操作ログ登録は確認できる一方、システム更新で同等の操作ログ要件が明示的に読めない。
- 指摘内容:
  - 監査要件として更新操作ログが必要かどうかの判断が設計書から一意に読めないため、運用監査要件と齟齬の可能性がある。
- 期待されるテスト:
  - 該当なし（要件確定後に、更新時ログ登録有無を検証するテストを追加）。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足:
  - create/update/delete/upper削除/lower削除の正常系が未整備。
- 入力値バリエーション不足:
  - systemName 必須・100文字境界・重複名・times 競合入力などの入力差分テストが不足。
- 境界値不足:
  - systemName 長さ境界、件数境界、空文字・null系の境界条件が不足。
- 分岐条件不足:
  - 権限分岐（テナント管理者/連携システム管理者/操作者）、参照チェック分岐（前段参照・統合参照）の検証が不足。
- データ入出力不足:
  - 削除系の関連テーブル削除網羅、操作ログ登録、返却スキーマ整合の検証が不足。
- 非更新確認不足:
  - 参照チェックで削除拒否時にデータ未更新であること、更新0件時に副作用がないことの検証が不足。
- 異常系不足:
  - 401/403/ServiceException/予期しない例外の分類検証、メッセージID相当の応答検証が不足。

## 総評
High指摘は、一覧APIの権限範囲逸脱と、システム削除時の削除漏れ（権限マスタ）の2点です。いずれも権限・データ整合性に直結し、リリース前の修正が必要です。
加えて、対象6APIのうち5APIが未テストで、既存テストも仕様網羅が不足しています。
修正優先順位は、1) 実装不備の解消、2) 削除/更新系を中心に異常系・分岐系テスト追加、の順が妥当です。

## 残留リスク・確認できなかった範囲
- spec/基本設計/インターフェイス定義 配下のAPIパラメータ定義（path/queryの厳密仕様）まで本レビューでは未参照のため、URLパラメータ実装整合の最終判断は追加確認が必要。
- 連携システム.md 以外の詳細設計（例: エラーコード厳密マッピング）が別紙管理の場合、その差分は未確認。