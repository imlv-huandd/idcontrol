# 組織ツリー レビュー結果

## 指摘事項

### 1. 適用開始日の範囲チェック（ORG03001）が未実装
- 優先度: High
- 種別: 実装不備
- 対象: functions/organizations/updateOrganizationsTree/app.py、shared/services/organizationService.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md「バリデーション定義 No.1 適用開始日入力チェック」  
    → 「適用開始日: 必須、日付形式、**選択中の組織構成履歴の有効開始日以上かつ有効終了日以下**（ORG03001）」
  - 実装: app.py は `applyStartDate` の必須チェックのみ。organizationService.py の `update_organizations_tree` は必須チェック + 日付形式チェック（COM02006）まで。有効期間範囲のチェック（ORG03001）がどちらにも存在しない。
- 指摘内容:
  - 設計書で定義されている「選択中の組織構成履歴の有効開始日以上かつ有効終了日以下」という適用開始日の範囲バリデーション（エラーコード: ORG03001）が、app.py・サービス層ともに一切実装されていない。有効期間外の日付を指定されても 400 エラーが返らず、意図しないデータが登録される可能性がある。
- 期待されるテスト:
  - 適用開始日が組織構成履歴の有効開始日と同じ日（境界値・正常系）
  - 適用開始日が組織構成履歴の有効開始日より1日前（境界値・異常系）→ ORG03001
  - 適用開始日が組織構成履歴の有効終了日と同じ日（境界値・正常系）
  - 適用開始日が組織構成履歴の有効終了日より1日後（境界値・異常系）→ ORG03001
  - 有効終了日=9999-12-31 の最新履歴に対して範囲内の日付を指定（正常系）

---

### 2. ツリー構造チェックで「複数ルート」を検出できない
- 優先度: High
- 種別: 実装不備
- 対象: functions/organizations/updateOrganizationsTree/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md「バリデーション定義 No.3 ツリー構造チェック」  
    → 「循環参照がないこと。かつ**ルートを持つ木構造として成立すること**（ORG03002）」
  - 実装: app.py の判定条件 `if has_cycle or len(roots) == 0:` はルートが「0件」の場合のみエラー。`len(roots) > 1`（複数ルートが存在する）場合は 409 を返さずそのまま処理を続行する。
- 指摘内容:
  - ルートが2件以上存在する場合（森状の構造、木構造として非成立）はエラーにすべきだが、現状は通過してしまう。組織ツリーが意図しない複数ツリー構造で登録される可能性がある。
- 期待されるテスト:
  - 親組織IDが None のノードが2件存在するリクエスト → 409 ORG03002
  - 全ノードの親が設定されている（ルート0件）→ 409 ORG03002
  - 循環参照を含むノード群 → 409 ORG03002
  - ルートが1件かつ循環なし（正常系）→ 200

---

### 3. 操作ログの変更組織名が空文字になっている
- 優先度: High
- 種別: 実装不備
- 対象: shared/services/organizationService.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md「保存ボタン押下 → 操作ログの登録」  
    → 「操作内容：「組織ツリー編集：適用開始日=[適用開始日] 変更組織=**[更新対象組織の組織名称を"、"区切りにした文字列]**」」
  - 実装: organizationService.py 操作ログ登録箇所  
    → `operation=f"組織ツリー編集：適用開始日={applyStartDate} 変更組織="`  
    変更組織名が末尾に連結されておらず、空文字のまま登録される。
- 指摘内容:
  - 変更対象となった組織の名称（"、"区切り）を操作ログの `operation` フィールドに設定する処理が未実装。`changed_orgs` リストの org_name を結合して末尾に追加する必要がある。操作ログの内容が不完全なため、監査・調査時に変更組織を特定できない。
- 期待されるテスト:
  - 変更組織が1件の場合、操作ログの operation フィールドに組織名が含まれること
  - 変更組織が複数件の場合、"、"区切りで全組織名が含まれること
  - 操作ログに applyStartDate が正しく設定されること

---

### 4. getOrganizationsTree・updateOrganizationsTree の単体テストコードが存在しない
- 優先度: High
- 種別: テスト不足
- 対象: tests/functions/organizations/test_organizations.py
- 根拠:
  - 設計書: spec/050.テスト/テスト観点.md「単体テスト観点」  
    → 「正常系、入力値バリエーション、境界値、分岐条件、データ入出力、非更新確認、異常系」が必須観点
  - 実装: test_organizations.py のコメントで getOrganizationsTree/updateOrganizationsTree のテストが「スキップ（関数未実装）」とされており、両 API ハンドラのテストケースが一切存在しない。
- 指摘内容:
  - getOrganizationsTree ハンドラ（権限不要・baseDate 取得）および updateOrganizationsTree ハンドラ（権限チェック・複数バリデーション・楽観的ロック・履歴更新）に対する単体テストが皆無。C1 カバレッジ目標を全く満たせていない。
- 期待されるテスト:
  - 【getOrganizationsTree】
    - baseDate パラメータなし → デフォルト 9999-12-31 で取得
    - baseDate パラメータあり（有効な日付）→ 指定日で取得
    - サービス層が空リストを返す場合 → 200 空配列
    - ServiceException 発生時 → 500
    - 汎用例外 → 500
  - 【updateOrganizationsTree】
    - 正常系（親組織変更1件）→ 200
    - 権限なし（一般ユーザー）→ 403
    - MASTER_DATA_ADMIN 権限で正常更新 → 200
    - applyStartDate 未指定 → 400 COM02001
    - treeTimes 未指定 → 400 COM02001
    - nodes 空リスト → 400 COM02001
    - orgIdPath が空の nodes → 400 COM02001
    - 循環参照のある nodes → 409 ORG03002
    - ルートが0件の nodes → 409 ORG03002
    - ルートが2件の nodes → 409 ORG03002
    - DB と nodes の組織ID が不一致 → 409 ORG03003
    - 親組織IDに変更がない → 409 ORG03004
    - 楽観的ロック競合（COM01010）→ 409
    - ServiceException 発生時 → 500
    - UnauthorizedException → 401
    - ForbiddenException → 403
    - 汎用例外 → 500

---

### 5. 変更有無チェックのエラーコードがサービス層と設計書で不一致
- 優先度: Middle
- 種別: 実装不備
- 対象: shared/services/organizationService.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md「バリデーション定義 No.5 変更有無チェック」  
    → メッセージID: `ORG03004`
  - 実装: organizationService.py `update_organizations_tree` L207-210  
    → `raise ServiceException(code="COM01010", message="...先に更新されています...")`  
    変更なし時に `COM01010`（楽観的ロック競合のコード）を raise しており、意味が異なる。  
    app.py 側では `parent_diff_count < 1` で `ORG03004` を正しく返しているが、サービス層の二重チェックで別コードが混在している。
- 指摘内容:
  - サービス層の変更なし判定（`len(changed_orgs) == 0`）で `COM01010` を raise しているため、app.py での `ORG03004` による事前チェックをすり抜けたケース（理論的には発生しないが）で誤ったエラーコードが返る。また、コードの意図が不明確になり保守性が低下する。サービス層でも `ORG03004` を使用すべき。
- 期待されるテスト:
  - サービス層（organizationService）の `update_organizations_tree` に変更なし状態を渡した場合 → `ORG03004` の ServiceException

---

### 6. COM01010 が異なる2つの意味（楽観的ロック競合 / 変更なし）に使われている
- 優先度: Middle
- 種別: 実装不備
- 対象: shared/services/organizationService.py、functions/organizations/updateOrganizationsTree/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/070.組織情報/組織ツリー編集.md「保存ボタン押下」  
    → `タイムスタンプ更新` でロック競合時のみ COM01010（メッセージ: COM01010）を使用  
    → `変更有無チェック` は ORG03004 を使用
  - 実装: organizationService.py で楽観的ロック失敗（L127-129）・変更なし（L207-210）・更新件数0件（L248-250, L260-262）の4か所すべてで `COM01010` を使用。app.py 側の `status_code = 409 if e.code == "COM01010" else 500` により全て同一 HTTP ステータスで返却される。
- 指摘内容:
  - 楽観的ロック競合と変更なしエラーが同一コードを使用しているため、クライアント側でエラー種別を区別できない。設計書の意図に反した実装になっている。
- 期待されるテスト:
  - 楽観的ロック競合 → 409 + COM01010
  - 変更なし → 409 + ORG03004（修正後）

---

### 7. baseDate クエリパラメータの日付形式バリデーション欠如
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/organizations/getOrganizationsTree/app.py
- 根拠:
  - 設計書: 入力値の形式に関する暗黙の要件（日付パラメータは日付形式であるべき）
  - 実装: `baseDate: str = path_params.get("baseDate") or "9999-12-31"` と取得後、形式チェックなしにサービス層へ渡している。
- 指摘内容:
  - `baseDate` に不正な文字列（例: `"abc"`, `"2024-13-01"`）が渡された場合、サービス層または DB クエリで予期しないエラーが発生する可能性がある。入力バリデーション（`is_date()` 等）を追加すべき。
- 期待されるテスト:
  - baseDate に不正な形式の文字列を渡した場合 → 400
  - baseDate に実在しない日付（例: 2024-02-30）を渡した場合 → 400
  - baseDate を省略した場合 → 200（デフォルト動作）

---

### 8. getOrganizationsTree で OrganizationTreeNode が未使用インポートになっている
- 優先度: Low
- 種別: 実装不備
- 対象: functions/organizations/getOrganizationsTree/app.py
- 根拠:
  - 設計書: なし（実装上の問題）
  - 実装: `from shared.services.organizationTreeService import OrganizationTreeNode, get_organizations_tree`  
    → `OrganizationTreeNode` はハンドラ内で一切使用されていない。
- 指摘内容:
  - 未使用インポートが残っており、可読性・保守性が低下する。削除すべき。
- 期待されるテスト:
  - なし（コード品質の問題）

---

### 9. app.py とサービス層のバリデーション二重実装による整合性リスク
- 優先度: Low
- 種別: 実装不備
- 対象: functions/organizations/updateOrganizationsTree/app.py、shared/services/organizationService.py
- 根拠:
  - 設計書: 特定の記述なし
  - 実装: `applyStartDate` の必須チェックが app.py（L48-49）とサービス層（L107-108）の両方に存在する。ツリー構造チェック（循環・ルート判定）も app.py のみに存在し、サービス層には存在しない。
- 指摘内容:
  - バリデーションの責務がハンドラとサービス層に分散しており、一方を修正した際に他方との整合性が取れなくなるリスクがある。また、サービス層を別の呼び出し元から使う場合にバリデーションが抜ける。設計上どちらに責務を置くか方針を統一すべき。
- 期待されるテスト:
  - なし（アーキテクチャ上の改善点）

---

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）

### 正常系不足
- getOrganizationsTree: 正常系テスト（baseDate あり/なし）が存在しない
- updateOrganizationsTree: 正常系テスト（1件変更、複数件変更）が存在しない

### 入力値バリエーション不足
- updateOrganizationsTree: applyStartDate の形式バリエーション（不正形式、実在しない日付、YYYY-MM-DD形式、YYYY/MM/DD形式）テストが存在しない
- updateOrganizationsTree: treeTimes の不正形式テストが存在しない
- updateOrganizationsTree: nodes の各フィールド（organizationId, orgIdPath など）の不正値テストが存在しない
- getOrganizationsTree: baseDate の形式バリエーションテストが存在しない

### 境界値不足
- updateOrganizationsTree: 適用開始日 = 有効開始日（境界値）、適用開始日 = 有効終了日（境界値）のテストが存在しない
- updateOrganizationsTree: nodes が1件（最小）のテストが存在しない

### 分岐条件不足
- updateOrganizationsTree: 権限 TENANT_ADMIN / MASTER_DATA_ADMIN / 権限なし の分岐テストが存在しない
- updateOrganizationsTree: 循環参照あり/なしの分岐テストが存在しない
- updateOrganizationsTree: ルートが0件/1件/複数件の分岐テストが存在しない
- updateOrganizationsTree: applyStartDate = 既存 start_date（同一履歴更新）vs applyStartDate > 既存 start_date（新規履歴追加）の分岐テストが存在しない

### データ入出力不足
- updateOrganizationsTree: DB の状態（現在のツリー）と入力 nodes が一致しない場合（ORG03003）のテストが存在しない
- updateOrganizationsTree: 楽観的ロック競合時（treeTimes が古い）のテストが存在しない
- updateOrganizationsTree: 操作ログに変更組織名が正しく設定されることの確認テストが存在しない

### 非更新確認不足
- updateOrganizationsTree: 変更がない nodes を渡した場合（ORG03004）に DB が更新されないことのテストが存在しない

### 異常系不足
- getOrganizationsTree: ServiceException 発生時 → 500 のテストが存在しない
- getOrganizationsTree: 汎用例外 → 500 のテストが存在しない
- updateOrganizationsTree: ServiceException（COM01010）発生時 → 409 のテストが存在しない
- updateOrganizationsTree: UnauthorizedException → 401 のテストが存在しない
- updateOrganizationsTree: ForbiddenException → 403 のテストが存在しない
- updateOrganizationsTree: 汎用例外 → 500 のテストが存在しない

---

## 総評

High 指摘が4件あり、うち3件（ORG03001 未実装・複数ルート判定欠如・操作ログ変更組織名未設定）は設計書に明示された仕様の実装漏れで、リリース前に修正が必須。最も影響が大きいのは単体テストコードが皆無である点で、API ハンドラ・サービス層ともにテスト未実施のためデグレ検知が不可能な状態。Middle 以下の指摘もエラーコード不一致・バリデーション欠如が含まれており、保守・運用フェーズでの混乱リスクがある。まずテスト実装と ORG03001・複数ルート判定・操作ログの修正を優先すること。

---

## 残留リスク・確認できなかった範囲

- `update_organizations_tree_version` 関数の実装内容（楽観的ロック処理の詳細）は未確認。タイムスタンプ比較ロジックの正確性については本レビュー範囲外。
- `OrganizationsTreeRepository` の各メソッド実装（insert/update の SQL・ORM 操作）は未確認。DB 側での整合性制約が正しく機能しているかは結合テストで確認が必要。
- getOrganizationsTree の `organizationTreeService.get_organizations_tree` におけるデータ取得ロジック（組織ツリーのノード構築処理）の詳細は探索結果から概要のみ把握。詳細な分岐・境界値については別途確認推奨。
- メッセージ定義ファイル（spec/基本設計/機能設計/メッセージ/メッセージ定義.md）は今回未参照。ORG03001 等のメッセージ文言の正確性は確認できていない。
- spec/詳細設計/ 配下に組織ツリー関連の詳細設計書が存在する場合、そちらとの照合は未実施。
