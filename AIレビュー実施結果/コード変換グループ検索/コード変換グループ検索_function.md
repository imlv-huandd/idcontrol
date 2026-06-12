# コード変換グループ検索・削除API レビュー結果

## 指摘事項

### 1. 操作ログの [コード変換グループ名] がハードコード化されている

- 優先度: **High**
- 種別: 実装不備
- 対象: deleteCodeConversionGroup (app.py-1)
- 根拠:
  - 設計書: 「操作ログ: 削除時に『コード変換グループ削除：[コード変換グループ名]』を記録」
    - 仕様では [コード変換グループ名] はプレースホルダ として実際の グループ名を動的に置換することを想定
  - 実装: `operation="コード変換グループ削除：[コード変換グループ名]"` が固定文字列
    - [コード変換グループ名] がリテラル文字列のまま記録される
    - 実際のグループ名（convertGroupMaster.name など）が operation フィールドに含まれていない
- 指摘内容:
  - 削除対象のコード変換グループ名を operation に動的に含める必要がある
  - 現在の実装では、操作ログから削除対象を特定できない
  - 監査要件違反の可能性がある（操作ログの一意性・追跡性低下）
- 期待されるテスト:
  - deleteCodeConversionGroup 実行後、operationLog の operation フィールドに「コード変換グループ削除：[実際のグループ名]」が記録されることを検証
  - 例: 削除対象グループ名が「税区分」の場合、「コード変換グループ削除：税区分」と記録されることを確認

### 2. times パラメータが None の場合のエラーハンドリング不備

- 優先度: **High**
- 種別: 実装不備
- 対象: deleteCodeConversionGroup (app.py-1)
- 根拠:
  - 設計書: 「楽観ロック: times パラメータが DB 値と一致することを確認」
    - times が NULL/未指定の場合の動作は明示されていないが、楽観ロックの要件から times は必須と推定
  - 実装: `times != str(convertGroupMaster.times)` の文字列比較
    - times が None の場合、str(None) = "None" となり、DB値と無条件に不一致 → 削除失敗
    - しかし、エラーメッセージが不明確である可能性がある
- 指摘内容:
  - times が None/未指定の場合、適切なエラーメッセージ（例: COM01015「楽観ロック失敗」）を返却する必要がある
  - 現在の実装では、原因不明の削除失敗となる
- 期待されるテスト:
  - times = None の場合、適切なエラーステータスと エラーコード が返却されることを検証
  - times が文字列（タイムスタンプ）である場合の正常系を検証
  - times = "" （空文字列）の場合も検証

### 3. RELATION_SYSTEM_ADMIN でユーザー権限情報が存在しない場合の処理が曖昧

- 優先度: **High**
- 種別: 設計確認事項 / 実装不備
- 対象: searchCodeConversionGroups (app.py-0)
- 根拠:
  - 設計書: 「RELATION_SYSTEM_ADMIN: convert_admin_system が未設定、またはユーザーが権限を持つ systemId と一致するグループを取得」
    - ユーザーが権限を持つ systemId がない場合の動作は明示されていない
  - 実装:
    ```python
    user_auth = userAuthorityRepository.get_user_authority_by_id(login_tenant_id, login_user_id)
    
    if user_auth and user_auth.authorities:
        authorities = user_auth.authorities.get("authorities", [])
        for item in authorities:
            if item.get("authority") == "RELATION_SYSTEM_ADMIN":
                list_permission.append(item.get("systemId"))
    # user_auth = None の場合、list_permission は空リスト
    results, total_count = convertGroupMasterRepository.search_convertgroupmasters(
        login_tenant_id, list_permission, code_conversion_group_name, page, page_size
    )
    ```
    - user_auth が None、または authorities が空の場合、list_permission が空リストになり、検索結果が 0 件
    - エラーが返却されず、空結果として処理される
- 指摘内容:
  - RELATION_SYSTEM_ADMIN権限を持つユーザーの権限情報（systemId）がない場合、仕様上は何を返却すべきか不明
    - 「権限を持つ systemId と一致するグループ」が1件もない場合、正常に 0 件返却すべきなのか
    - または、権限不足エラー（COM01001）を返却すべきなのか
  - 現在の実装は 0 件で返却しているが、**仕様との整合性の確認が必要**
  - ユーザー権限情報がDB上に存在しない異常系の場合も、同じ動作を示す
- 期待されるテスト:
  - RELATION_SYSTEM_ADMIN ユーザーで user_auth = None の場合の動作を検証
  - RELATION_SYSTEM_ADMIN ユーザーで authorities が空の場合の動作を検証
  - 両者が区別される場合は、エラーメッセージが異なることを確認

### 4. ページネーション境界値が検証されていない

- 優先度: **High**
- 種別: 実装不備 / テスト不足
- 対象: searchCodeConversionGroups (app.py-0)
- 根拠:
  - 設計書: ページネーション仕様が「spec/基本設計/機能設計/100.コード定義/コード変換グループ検索.md」に記載されているはず
  - 実装: page, page_size が直接 repository に渡されている
    ```python
    results, total_count = convertGroupMasterRepository.search_convertgroupmasters(
        login_tenant_id, list_permission, code_conversion_group_name, page, page_size
    )
    ```
    - page < 1、page_size < 1、page_size が最大値超過した場合の処理が実装されていないように見える
  - テスト: テスト観点「ページネーション: 1ページ目、最終ページ、範囲外」が記載されているが、テストコード（6件）に境界値テストが含まれているか不明
- 指摘内容:
  - page = 0, -1 の場合の処理
  - page_size = 0, -1 の場合の処理
  - page_size が 1000 を超える場合（上限チェック）
  - 最終ページ超過（page > max_page）の場合
  - いずれも適切なエラーまたは空結果を返却する必要があるが、実装が見当たらない
- 期待されるテスト:
  - page = 0 の場合のエラー返却（またはデフォルト 1 に矯正）を検証
  - page = -1 の場合を検証
  - page_size = 0, -1 を検証
  - page_size > 上限値 の場合を検証
  - 最終ページ超過時の空結果を検証

### 5. ConvertMaster 削除失敗時のトランザクション処理が不明確

- 優先度: **Middle**
- 種別: 実装不備
- 対象: deleteCodeConversionGroup (app.py-1)
- 根拠:
  - 実装コード抜粋に、「ConvertMaster 削除後、ConvertGroupMaster 削除前に flush() を呼んでいる」との指摘
    - 詳細な実装を確認する必要があるが、flush() タイミングが db.commit() ではなく中途 flush の可能性あり
  - 設計書: トランザクション仕様が明示されているはず
    - ConvertMaster と ConvertGroupMaster はどちらが削除されるべきか
    - 片方の削除失敗時は？（ロールバック要件）
- 指摳内容:
  - ConvertMaster 削除失敗 → ConvertGroupMaster 削除スキップ（外部キー制約エラー防止）
  - ConvertGroupMaster 削除失敗 → ConvertMaster 削除は既に確定
  - このため、中途 flush が存在する場合、ロールバックできず不整合になる
  - 適切なトランザクション設計が必要
- 期待されるテスト:
  - ConvertMaster 削除成功 → ConvertGroupMaster 削除成功
  - ConvertMaster 削除失敗 → ConvertGroupMaster が削除されないこと、またはロールバックされることを検証
  - 例外発生時のロールバック動作を検証

### 6. convertGroupMaster.times が None の場合の型変換エラー

- 優先度: **Middle**
- 種別: 実装不備
- 対象: deleteCodeConversionGroup (app.py-1)
- 根拠:
  - 実装: `times != str(convertGroupMaster.times)`
    - convertGroupMaster.times が None の場合、str(None) = "None"
    - これはタイムスタンプ値と異なるため、必ず times != str(convertGroupMaster.times) が真
    - 設計上、times は必ず値を持つべきだが、DB異常時（NULL値）に適切に処理されない
  - times が ISO 形式の文字列または datetime オブジェクトである場合、str() 呼び出しの型チェックが不十分
- 指摘内容:
  - convertGroupMaster.times が None の場合、明示的にエラーを返却すべき
  - またはそもそもこの状態がありえないことを保証する必要がある
- 期待されるテスト:
  - convertGroupMaster.times = None の場合のエラーハンドリングを検証
  - 正常なタイムスタンプ値（ISO 形式）との比較が正しく動作することを検証

### 7. 楽観ロック失敗時のエラーメッセージが COM01015 か不明

- 優先度: **Middle**
- 種別: 設計確認事項
- 対象: deleteCodeConversionGroup (app.py-1)
- 根拠:
  - 設計書: バリデーション定義に「楽観ロック: times パラメータが DB 値と一致することを確認」
    - エラーコードが明示されているはず
    - 一般的には COM01015（楽観ロック失敗）
  - 実装: 具体的なエラーコード返却内容が抜粋では不明
- 指摘内容:
  - times 不一致時に、設計で指定されたエラーコード（例: COM01015）が返却されているか確認が必要
- 期待されるテスト:
  - times 不一致時、正しいエラーコード（COM01015 など）が返却されることを検証

### 8. convert_admin_system = None のグループが RELATION_SYSTEM_ADMIN に見えない可能性

- 優先度: **Middle**
- 種別: 設計確認事項 / テスト不足
- 対象: searchCodeConversionGroups (app.py-0)
- 根拠:
  - 設計書: 「RELATION_SYSTEM_ADMIN: convert_admin_system が未設定、またはユーザーが権限を持つ systemId と一致するグループを取得」
    - convert_admin_system = NULL のグループは、RELATION_SYSTEM_ADMIN ユーザーにも見えるべき
  - 実装:
    ```python
    list_permission = []  # RELATION_SYSTEM_ADMIN の場合、systemId リストが入る
    results, total_count = convertGroupMasterRepository.search_convertgroupmasters(
        login_tenant_id, list_permission, code_conversion_group_name, page, page_size
    )
    ```
    - repository の内部実装次第だが、list_permission が指定された場合、convert_admin_system = NULL のグループがフィルタリングされる可能性がある
- 指摘内容:
  - repository の search_convertgroupmasters がどのようにフィルタを適用しているか不明
  - 仕様: convert_admin_system = NULL **OR** convert_admin_system IN (systemId リスト)
  - 実装が systemId リストのみでフィルタしている場合、NULL グループが漏れる
- 期待されるテスト:
  - RELATION_SYSTEM_ADMIN ユーザーで、convert_admin_system = NULL のグループが検索結果に含まれることを検証
  - convert_admin_system = ユーザーの systemId のグループが含まれることを検証
  - convert_admin_system = 別の systemId のグループが含まれないことを検証

### 9. コード変換グループ名が空文字列の場合のフィルタリング仕様が不明確

- 優先度: **Low**
- 種別: 設計確認事項 / テスト不足
- 対象: searchCodeConversionGroups (app.py-0)
- 根拠:
  - 設計書: 検索条件「code_conversion_group_name: 空、部分一致」とテスト観点で記載
  - 実装: `code_conversion_group_name` が repository に直接渡される
    - 空文字列の場合、全件取得か条件なしか、部分一致のロジックが repository に依存
  - テスト観点: 「入力値バリエーション: コード変換グループ名 - 空」が記載
- 指摘内容:
  - code_conversion_group_name = "" の場合、全件取得 か エラー返却 か、仕様が不明確
- 期待されるテスト:
  - code_conversion_group_name = "" の場合の動作を検証
  - code_conversion_group_name = None の場合の動作を検証
  - 部分一致のロジック（LIKE "%...%"）が正しく動作することを検証

### 10. 複数の ConvertMaster を持つグループの完全削除検証不足

- 優先度: **Middle**
- 種別: テスト不足
- 対象: deleteCodeConversionGroup (app.py-1), test_searchCodeConversionGroups.py
- 根拠:
  - テスト観点: 「データ入出力: conversions 配列の集約」
    - 1つの ConvertGroupMaster が複数の ConvertMaster を保有可能であることを示唆
  - テストコード: 6件中に、複数 ConvertMaster のシナリオが含まれているか不明
  - 実装: ConvertMaster をループで削除しているはずだが、例外時の部分削除状態は不明
- 指摘内容:
  - ConvertGroupMaster 削除時、紐付く ConvertMaster がすべて削除されることを検証する テストが不足している
  - 例: 5件の ConvertMaster を持つグループで、3件削除後に例外が発生する場合、トランザクションがロールバックされることを確認する必要がある
- 期待されるテスト:
  - ConvertMaster が 0 件のグループ削除を検証
  - ConvertMaster が 1 件のグループ削除を検証
  - ConvertMaster が複数件（例: 5件）のグループ削除を検証
  - 削除後、紐付く ConvertMaster がすべて削除されていることを DB 確認

### 11. 権限不足エラー（COM01001）の検証不足

- 優先度: **Middle**
- 種別: テスト不足
- 対象: deleteCodeConversionGroup (app.py-1), test_searchCodeConversionGroups.py
- 根拠:
  - 設計書: 削除権限チェック「RELATION_SYSTEM_ADMIN でかつ convert_admin_system が指定: 当該 systemId の権限者のみ削除可」
  - テスト観点: 「分岐条件: TENANT_ADMIN、RELATION_SYSTEM_ADMIN、権限なし」
  - テストコード: test_search_exception_unauthorized が存在（検索用）だが、削除時の権限チェックテストが記載されているか不明
- 指摘内容:
  - deleteCodeConversionGroup で RELATION_SYSTEM_ADMIN ユーザーが権限のない systemId のグループを削除しようとした場合、COM01001 を返却するべき
  - その検証がない場合は、重大な権限漏洩である
- 期待されるテスト:
  - deleteCodeConversionGroup で RELATION_SYSTEM_ADMIN ユーザーが権限外の convert_admin_system を削除しようとした場合、COM01001 を返却することを検証
  - TENANT_ADMIN ユーザーはすべて削除可能を検証

### 12. 削除対象グループが存在しない場合のエラーハンドリング

- 優先度: **Low**
- 種別: 実装不備 / テスト不足
- 対象: deleteCodeConversionGroup (app.py-1)
- 根拠:
  - 実装: `convertGroupMaster = repository.get_by_id()`
    - グループが見つからない場合のエラーハンドリングが抜粋では不明
  - テスト観点: 「データ入出力: データなし」
- 指摘内容:
  - グループが存在しない場合、適切なエラー（例: COM01016「レコードが見つかりません」）を返却するべき
- 期待されるテスト:
  - 存在しない group_id でのdeletを試みた場合、エラーコードが返却されることを検証

---

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）

### 正常系不足
- ConvertMaster が複数件存在するグループの完全削除検証
- convert_admin_system = NULL のグループへのアクセス検証（RELATION_SYSTEM_ADMIN）
- ページネーション: 最終ページ、1ページ目の複数データ取得

### 入力値バリエーション不足
- code_conversion_group_name = "" （空文字列）での全件取得 or エラー返却
- code_conversion_group_name = None での動作
- times = None、times = "" （空文字列）の楽観ロック検証

### 境界値不足
- page = 0, -1 の検証
- page_size = 0, -1 の検証
- page_size > 上限値 の検証
- 最終ページ超過（page > max_page）での空結果

### 分岐条件不足
- RELATION_SYSTEM_ADMIN で user_auth = None の場合
- RELATION_SYSTEM_ADMIN で authorities が空の場合
- deleteCodeConversionGroup での RELATION_SYSTEM_ADMIN の权限外 systemId 削除試行

### データ入出力不足
- DeleteCodeConversionGroup 後、ConvertMaster がすべて削除されていることの検証（DB 確認）
- 操作ログに実際のグループ名が記録されていることの検証
- convertGroupMaster.times の型（datetime vs 文字列）の検証

### 非更新確認不足
- deleteCodeConversionGroup 実行後、他のグループや ConvertMaster が影響されていないことの確認
- deleteCodeConversionGroup でのトランザクション失敗時のロールバック検証

### 異常系不足
- ConvertMaster 削除失敗時（例外発生）のロールバック
- DB接続失敗時のエラーハンドリング
- convertGroupMaster.times が NULL の場合のエラー処理
- 楽観ロック失敗（times 不一致）時の正確なエラーコード（COM01015）返却
- 削除対象グループが見つからない場合（COM01016）
- 権限不足エラー（COM01001）

---

## 総評

**High優先度指摘が4件** あり、いずれもユーザー操作の表面化または監査要件に関わる重大なもの。最初の2件（操作ログのハードコード化と times のエラーハンドリング）は**リリース前に必ず修正すべき**。

設計書と実装の整合性確認（指摘#3,#7,#8）も重要。特に RELATION_SYSTEM_ADMIN の権限フィルタリング仕様が実装と一致しているかの再確認が必須。

テスト観点の約 40% が実装されていない見込み。複数データ、権限分岐、例外系、トランザクション検証を重点的に追加実装が必要。

---

## 残留リスク・確認できなかった範囲

- `convertGroupMasterRepository.search_convertgroupmasters()` の内部実装（convert_admin_system = NULL の フィルタリング方式）
- `userAuthorityRepository.get_user_authority_by_id()` が None を返す条件
- deleteCodeConversionGroup の詳細実装（flush() タイミング、例外処理）
- 操作ログ記録時のエラーハンドリング
- 全テストケースの詳細アサーション内容（6件中の正確な検証内容）
- spec/050.テスト/テスト観点.md の完全な内容（参照設計書がユーザーから提供されていない）
