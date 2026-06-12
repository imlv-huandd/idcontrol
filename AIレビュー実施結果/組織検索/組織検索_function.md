# 組織検索 レビュー結果

> レビュー実施日: 2026-06-09  
> レビュー対象: 組織検索機能（searchOrganizations / exportOrganizations / abolishOrganization / getTenant / getOrganizationsTreeVersion）  
> 参照設計書: spec/基本設計/機能設計/070.組織情報/組織検索.md

---

## 指摘事項

### 1. CSV出力の有効終了日 9999/12/31 を空文字出力していない
- 優先度: **High**
- 種別: 実装不備
- 対象: `shared/services/organizationService.py`（export_organizations 関数）
- 根拠:
  - 設計書: `spec/基本設計/機能設計/070.組織情報/組織検索.md` 検索結果エリア No.12「有効終了日：9999/12/31の場合は出力しない」および CSV出力サンプル（9999-12-31 は空白出力）
  - 実装: `organizationService.py` 内で `format_date(organization.end_date)` を使用しており、9999/12/31 でも "9999/12/31" がそのまま出力される。空文字への変換処理なし。
- 指摘内容:
  - 有効終了日が 9999/12/31 の場合、設計書は空文字出力を要求しているが、実装は "9999/12/31" を出力する。CSV出力サンプルでも `"9999/12/31"` ではなく空文字が期待されている。利用者が参照する出力データが設計仕様と異なる。
- 期待されるテスト:
  - 有効終了日 = 9999/12/31 のデータに対して、CSVの当該列が空文字で出力されることを確認するテスト
  - 有効終了日 ≠ 9999/12/31 の場合は日付が正しく出力されることを確認するテスト

---

### 2. CSVヘッダーのテナント組織ID列名がハードコード（テナント設定から動的取得していない）
- 優先度: **High**
- 種別: 実装不備
- 対象: `shared/services/organizationService.py`（export_organizations 関数）
- 根拠:
  - 設計書: `spec/基本設計/機能設計/070.組織情報/組織検索.md` CSV出力ボタン押下 > 出力項目「`組織情報`.`テナント組織ID`(列名は`テナントマスター`の`組織コードカラム名`)」
  - 実装: `header = ["部門コード(基軸コード)", ...]` と固定文字列がハードコードされており、テナントマスターの `orgColumnName` を参照していない。
- 指摘内容:
  - テナントごとに異なる組織コードカラム名が設定可能な設計であるにもかかわらず、CSV の第1カラム名が "部門コード(基軸コード)" に固定されている。テナントごとに列名が変わる要件を満たせない。
- 期待されるテスト:
  - テナント設定の orgColumnName が "部署コード" の場合、CSVヘッダーの1列目が "部署コード" となることを確認するテスト
  - orgColumnName が異なる複数テナントでそれぞれ正しい列名で出力されることを確認するテスト

---

### 3. getTenant/app.py で認証処理がコメントアウトされており、認証・認可なしにテナント情報が取得できる
- 優先度: **High**
- 種別: 実装不備
- 対象: `functions/tenants/getTenant/app.py`
- 根拠:
  - 設計書: 詳細設計 OpenAPI 定義 `spec/詳細設計/openapi/paths/tenants.yaml` にて認証要件が定義。`spec/基本設計/概要資料/権限について.md` の権限体系にてテナント情報取得には認証が必要。
  - 実装: `getTenant/app.py` L16-17 にて `login_user = get_current_user(event)` と `login_tenant_id = login_user.tenantId` がコメントアウトされており、未認証状態でハンドラーが呼び出せる。
- 指摘内容:
  - 認証処理が欠落しているため、認証なしで任意の `tenantId` を指定してテナント情報（設定情報・カラム名等）を取得できる状態。テナント情報の不正取得（情報漏洩）のリスクがある。コメントアウトが意図的であれば設計書への記載および理由の明示が必要。
- 期待されるテスト:
  - 未認証リクエストに対して 401 を返すことを確認するテスト（現在は認証チェックが機能していないため pass してしまう）

---

### 4. 操作ログの「テナント組織ID」にテナントIDが誤記録される
- 優先度: **High**
- 種別: 実装不備
- 対象: `shared/services/organizationService.py`（abolish_organization 関数）
- 根拠:
  - 設計書: `spec/基本設計/機能設計/070.組織情報/組織検索.md` 廃止実行ボタン押下 > 操作ログ「操作内容：`組織情報廃止：テナント組織ID 組織名称 廃止日`」
  - 実装: `operation=f"組織情報廃止：{loginUser.tenantId} {organization.org_name if organization else ''} {abolishDate}"` にて `loginUser.tenantId`（UUIDのテナントID）を使用しており、設計書が要求する `organization.tenant_org_id`（テナント組織ID＝部門コード等）とは異なる値が記録される。
- 指摘内容:
  - 操作ログに記録されるべき「テナント組織ID（部門コード）」の代わりにシステム内部のテナントUUIDが出力されている。運用時に監査ログを参照しても、どの組織が廃止されたかを人間が読み取れない。
- 期待されるテスト:
  - 廃止実行後に操作ログの operation 列に `"組織情報廃止：{tenant_org_id} {org_name} {abolish_date}"` 形式で記録されていることを確認するテスト

---

### 5. CSV出力（search_organizations_for_csv）で include_abolished=False 時に廃止済みデータが混入する可能性
- 優先度: **High**
- 種別: 実装不備
- 対象: `shared/repositories/organizationRepository.py`（search_organizations_for_csv 関数）
- 根拠:
  - 設計書: `spec/基本設計/機能設計/070.組織情報/組織検索.md` CSV出力「廃止済を含む：trueの場合に条件追加」→ false の場合は廃止済みを除外する要件
  - 実装: `search_organizations_for_csv` の `include_abolished=False` 時、`Organization.is_abolished.is_(False)` フィルターが存在しない。`search_organizations`（検索用）では同条件に `abolished_condition = and_(Organization.is_abolished.is_(False))` を追加している。CSV版のみこのフィルターが欠落している。
- 指摘内容:
  - `include_abolished=False` でもアクティブ期間内（end_date >= search_date）の廃止済みレコード（is_abolished=True）がCSV出力に含まれてしまう。廃止済みを除外すべき条件下でも廃止済みデータが出力されるデータ品質の問題がある。また `end_date IS NULL` の条件も設計書・エンティティ定義に根拠が見当たらず、要確認事項。
- 期待されるテスト:
  - includeAbolished=False のとき、is_abolished=True のレコードがCSVに含まれないことを確認するテスト
  - includeAbolished=True のとき、廃止済みレコードが正しく含まれることを確認するテスト

---

### 6. AbolishOrganizationHandler のテストが正常系・例外のみで設計仕様の大半が未検証
- 優先度: **High**
- 種別: テスト不足
- 対象: `tests/functions/organizations/test_organizations.py`（TestAbolishOrganizationHandler）
- 根拠:
  - 設計書: `spec/基本設計/機能設計/070.組織情報/組織検索.md` 廃止実行ボタン押下に定義されたバリデーション3種（権限・入力・廃止対象）、楽観的ロック、操作ログ、配下組織廃止
  - 実装: `TestAbolishOrganizationHandler` に `test_success`, `test_service_exception`, `test_generic_exception` の3ケースのみ存在。
- 指摘内容:
  - 以下のテストケースが全て欠落している：権限チェック（UnauthorizedException / ForbiddenException）、廃止日未入力エラー、廃止日不正形式エラー、廃止対象チェックエラー（有効開始日 >= 廃止日）、楽観的ロック失敗（COM01010）、配下組織も含めた廃止処理の確認、操作ログへの記録確認。
- 期待されるテスト:
  - テナント管理者・マスターデータ管理者以外のユーザーで呼び出した場合に 403 が返ることを確認するテスト
  - abolishDate が未入力の場合にバリデーションエラーになることを確認するテスト
  - abolishDate が不正形式（例: "2026-13-01"）の場合にバリデーションエラーになることを確認するテスト
  - 廃止対象組織の有効開始日 >= 廃止日の場合に ORG01001 エラーになることを確認するテスト
  - organizationTreeTimes が不一致（楽観的ロック失敗）の場合に COM01010 エラーになることを確認するテスト
  - 廃止実行後、対象組織および配下組織の is_abolished=True かつ end_date=廃止日 に更新されることを確認するテスト
  - 廃止実行後、操作ログが登録されることを確認するテスト

---

### 7. ExportOrganizationsHandler のテストが正常系・例外のみで設計仕様の大半が未検証
- 優先度: **High**
- 種別: テスト不足
- 対象: `tests/functions/organizations/test_organizations.py`（TestExportOrganizationsHandler）
- 根拠:
  - 設計書: `spec/基本設計/機能設計/070.組織情報/組織検索.md` CSV出力ボタン押下（ダウンロード権限チェック、0件エラー、CSV出力内容）
  - 実装: `TestExportOrganizationsHandler` に `test_success`, `test_service_exception`, `test_generic_exception` の3ケースのみ存在。
- 指摘内容:
  - 以下のテストケースが欠落している：ダウンロード権限なしユーザーで 403 を返すことの確認、検索結果0件時のエラーメッセージ（COM01003）確認、CSV出力内容（ヘッダー列名、有効終了日9999/12/31の空文字出力、廃止フラグ"廃止"出力、組織属性項目のソート順）の確認、CRLF改行・UTF-8-SIGの確認。
- 期待されるテスト:
  - ダウンロード権限なしユーザーが呼び出した場合に 403 が返ることを確認するテスト
  - 該当データ0件の場合に COM01003 のエラーレスポンスが返ることを確認するテスト
  - CSVのヘッダー行にテナントマスターの orgColumnName が1列目に設定されていることを確認するテスト
  - 有効終了日 = 9999/12/31 のデータのCSV出力が空文字であることを確認するテスト
  - is_abolished=True のデータのCSV廃止列が "廃止" であることを確認するテスト
  - 組織属性項目が表示順（display_order）の昇順で出力されることを確認するテスト

---

### 8. search_organizations の page / pageSize が空文字の場合に ValueError が発生する
- 優先度: **Middle**
- 種別: 実装不備
- 対象: `shared/services/organizationService.py`（search_organizations 関数）
- 根拠:
  - 設計書: 特記なし（ただし app.py で `page: str = path_params.get("page") or ""` というデフォルト値処理が実装されており、空文字が渡り得る）
  - 実装: `search_organizations` 内で事前チェックなしに `int(page)` / `int(pageSize)` を呼び出しており、空文字またはページング未指定のリクエストで `ValueError` が発生し、500 エラーとなる。
- 指摘内容:
  - app.py が `page` と `pageSize` の未指定時に空文字 `""` を設定しているため、ページネーションパラメータなしのリクエストで必ず ValueError が発生する。ページングパラメータのデフォルト値設定またはバリデーションが欠落している。
- 期待されるテスト:
  - page / pageSize 未指定のリクエストで正常なレスポンスが返ること（または適切なバリデーションエラーが返ること）を確認するテスト

---

### 9. SearchOrganizationsHandler のテストに検索条件の分岐確認が欠落
- 優先度: **Middle**
- 種別: テスト不足
- 対象: `tests/functions/organizations/test_organizations.py`（TestSearchOrganizationsHandler）
- 根拠:
  - 設計書: `spec/基本設計/機能設計/070.組織情報/組織検索.md` 検索ボタン押下（廃止済を含む条件分岐、各検索条件の部分一致）
  - `spec/050.テスト/テスト観点.md` 単体テスト観点「分岐条件」「入力値バリエーション」「異常系」
  - 実装: `test_success`, `test_no_params`, `test_service_exception`, `test_generic_exception` の4ケースのみ。
- 指摘内容:
  - 以下のテストケースが欠落している：includeAbolished=true / false の検索結果の差異、各検索条件（tenantOrgId / orgName / orgNameAll / orgNameDisplay）による絞り込み確認、baseDate 指定時と未指定時の差異、権限チェック（Unauthorized / Forbidden）、検索結果0件の確認、ページング動作確認。
- 期待されるテスト:
  - includeAbolished=true と false で返却レコード件数が変わることを確認するテスト
  - 各検索条件で部分一致絞り込みが機能することを確認するテスト
  - baseDate 未指定のデフォルト（9999-12-31相当）で最新データが取得されることを確認するテスト
  - page / pageSize を指定した場合に正しいページのデータが返ることを確認するテスト
  - 未認証リクエストで 401、権限不足で 403 が返ることを確認するテスト

---

### 10. abolishOrganization/app.py で logger.error を通常の event ログに使用
- 優先度: **Middle**
- 種別: 実装不備
- 対象: `functions/organizations/abolishOrganization/app.py`
- 根拠:
  - 設計書: 特記なし（運用上の品質問題）
  - 実装: `abolishOrganization/app.py` L24 にて `logger.error("event: %s", json.dumps(event))` と、正常ケースのイベントログを ERROR レベルで記録している。他のハンドラー（searchOrganizations 等）は `logger.info` を使用しており不整合。
- 指摘内容:
  - 正常処理の入力ログを ERROR レベルで記録すると、監視アラートや障害調査時に誤検知が発生する。`logger.info` に修正が必要。
- 期待されるテスト:
  - 正常系テストにて logger.error ではなく logger.info が呼ばれることを確認するテスト（または既存テストの修正不要、実装修正で対応）

---

### 11. getTenant/app.py で tenantId が空文字の場合に入力チェックなし
- 優先度: **Middle**
- 種別: 実装不備
- 対象: `functions/tenants/getTenant/app.py`
- 根拠:
  - 設計書: 詳細設計 OpenAPI `spec/詳細設計/openapi/paths/tenants.yaml` の `tenantId` は required パラメータ
  - 実装: `tenantId: str = path_params.get("tenantId") or ""` で空文字がデフォルト設定されており、空文字のまま `get_tenant_by_id(db, tenantId)` が呼ばれる。サービス層・リポジトリ層での空文字チェックが確認できない。
- 指摘内容:
  - tenantId が空文字でリクエストされた場合、不正クエリが発行される可能性がある。最低限、空文字の場合に 400 Bad Request または 404 を返すチェックが必要。
- 期待されるテスト:
  - tenantId 未指定のリクエストで適切なエラーレスポンスが返ることを確認するテスト

---

### 12. search_organizations_for_csv の end_date IS NULL 条件の根拠が不明
- 優先度: **Middle**
- 種別: 設計確認事項
- 対象: `shared/repositories/organizationRepository.py`（search_organizations_for_csv 関数）
- 根拠:
  - 設計書: `spec/基本設計/エンティティ定義/内部設計/エンティティ定義.md` にて組織情報の end_date（有効終了日）の NULL 可否について確認が必要
  - 実装: `or_(Organization.end_date >= search_date, Organization.end_date.is_(None))` という条件があり、コメントには `# is_abolished == False -> end_date == None` と記載されているが、設計書上のエンティティ定義で end_date が NULL になるケースが定義されているか不明。
- 指摘内容:
  - `end_date IS NULL` の条件は検索用（search_organizations）には存在せず、CSV用のみに存在する。設計書に end_date=NULL のデータが存在する旨の定義がなければ、意図しない条件追加である可能性がある。設計書との整合性確認が必要。
- 期待されるテスト:
  - end_date が NULL のデータが存在する場合の動作確認（設計書で定義されているかどうかに依存）

---

### 13. テストコードが HTTP ハンドラー層のみで、サービス層・リポジトリ層のテストが存在しない
- 優先度: **Middle**
- 種別: テスト不足
- 対象: `tests/` 全体（organizationService, organizationRepository のテスト）
- 根拠:
  - 実装: `tests/functions/organizations/test_organizations.py` のみ存在し、`tests/services/` や `tests/repositories/` 配下に organizationService / organizationRepository のテストファイルが存在しない
  - `spec/050.テスト/テスト観点.md` 単体テスト観点「正常系」「分岐条件」「データ入出力」「異常系」
- 指摘内容:
  - サービス層・リポジトリ層の複雑なロジック（廃止条件の OR 条件、廃止対象チェックの境界値、楽観的ロックの動作など）がハンドラー層テストのモックで隠蔽されており、設計仕様との照合ができていない。C1カバレッジ観点でも分岐が見えないため品質保証が不十分。
- 期待されるテスト:
  - organizationService の abolish_organization に対して、廃止対象チェック境界値（start_date = abolishDate - 1日 / start_date = abolishDate）のテスト
  - organizationRepository の search_organizations に対して、includeAbolished の OR 条件が正しく機能することを確認するテスト（DB統合テストまたは SQLAlchemy クエリ生成の単体テスト）

---

### 14. getOrganizationsTreeVersion/app.py で orm_to_dict を二重インポートしている
- 優先度: **Low**
- 種別: 実装不備
- 対象: `functions/organizations/getOrganizationsTreeVersion/app.py`
- 根拠:
  - 設計書: 特記なし（コード品質問題）
  - 実装: `from shared.core.common import (get_current_user, normalize_event, to_response)` と `from shared.core.common import orm_to_dict` の2行に分けてインポートしており、冗長。
- 指摘内容:
  - 同一モジュール `shared.core.common` からの import が2行に分割されており、DRY 原則違反。機能への影響はないが、保守性低下につながる。
- 期待されるテスト:
  - なし（実装修正のみ）

---

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）

### 正常系不足
- AbolishOrganization: 廃止実行後に対象組織・配下組織の is_abolished=True / end_date=廃止日 への更新確認
- ExportOrganizations: CSVのコンテンツ（列名・廃止フラグ・有効終了日）の出力内容確認

### 入力値バリエーション不足
- SearchOrganizations: 各検索条件（tenantOrgId, orgName, orgNameAll, orgNameDisplay）の指定有無パターン
- SearchOrganizations: includeAbolished = true / false の差異
- AbolishOrganization: abolishDate の各パターン（正常値、境界値、不正値）

### 境界値不足
- SearchOrganizations: page / pageSize の下限（1件、0件、最大件数）
- AbolishOrganization: 廃止対象組織の有効開始日 = 廃止日（NG）、廃止日 - 1日（OK）

### 分岐条件不足
- SearchOrganizations: includeAbolished 分岐（OR 条件）
- AbolishOrganization: 楽観的ロック成功 / 失敗の分岐
- AbolishOrganization: 配下組織あり / なし の分岐
- ExportOrganizations: include_abolished の分岐による出力差異

### データ入出力不足
- ExportOrganizations: CSV出力内容（ヘッダー、有効終了日の9999/12/31空白化、廃止フラグ文字列、組織属性表示順）
- AbolishOrganization: 操作ログの登録内容（function, operation 列の値）
- SearchOrganizations: レスポンスボディのページング情報（total_count, page）の確認

### 非更新確認不足
- AbolishOrganization: 廃止対象外組織（配下でない組織）が更新されないことの確認
- AbolishOrganization: is_abolished=True の組織（既廃止）が再廃止されないことの確認

### 異常系不足
- SearchOrganizations: 未認証（401）、権限不足（403）
- AbolishOrganization: 権限不足（401/403）、廃止日未入力、廃止日不正形式、廃止対象チェックエラー（ORG01001）、楽観的ロック失敗（COM01010）
- ExportOrganizations: ダウンロード権限不足（403）、0件エラー（COM01003）
- getTenant: tenantId 未指定、tenantId 不正値

---

## 総評

**High 指摘が5件**あり、うち3件（CSV有効終了日の出力誤り・CSVヘッダーのテナント組織ID列名ハードコード・操作ログへのテナントID誤記録）は設計仕様と明確に異なる実装不備であるため、**現状のままリリース不可**。特にCSV出力とテナント認証なしの問題はユーザー影響・セキュリティリスクが大きい。テスト面では、AbolishOrganization・ExportOrganizations ともにハンドラー正常系・例外のみで設計上の主要なバリデーション・ビジネスロジックが未検証であり、サービス層・リポジトリ層のテストも存在しないため品質保証が不十分。修正優先順位は High 指摘の実装修正 → テスト追加の順とする。

---

## 残留リスク・確認できなかった範囲

- `getTenant/app.py` の `get_current_user` コメントアウトが意図的かどうか（フロント側で認証済み前提の内部 API として設計されている可能性）。設計書に認証要件の明示がないため、設計者への確認が必要。
- `search_organizations_for_csv` の `end_date IS NULL` 条件について、エンティティ定義上 end_date が NULL になりうるかどうかの確認ができなかった。実データで NULL が存在する場合、この条件が意図した挙動かどうかは設計者確認が必要。
- `exportOrganizations/app.py` の baseDate デフォルト値が `""` である一方、`searchOrganizations/app.py` は `"9999-12-31"` であり不整合があるが、リポジトリ層で同じ 9999-12-31 に統一されているため実害は現状なし。ただし明示的なデフォルト値の統一が推奨される。
- テナント管理者・マスターデータ管理者の権限コード定義（`TENANT_ADMIN`, `MASTER_DATA_ADMIN`, `DOWNLOAD`）について、`spec/基本設計/概要資料/権限について.md` の権限種類との突き合わせは実施したが、権限コード文字列の正確な一致まで確認できていない。
