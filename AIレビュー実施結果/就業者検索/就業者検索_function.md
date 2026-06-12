# 就業者検索 レビュー結果

## 指摘事項

### 1. CSV出力仕様の必須項目（退職）未出力
- 優先度: High
- 種別: 実装不備
- 対象: functions/persons/createPersonCsv/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/060.就業者情報/就業者検索.md の「CSV出力ボタン押下」では、出力項目に「就業者情報.退職（trueの場合、"退職"）」を含める仕様。
  - 実装: functions/persons/createPersonCsv/app.py のCSVヘッダー生成（L72-L94）およびデータ行生成（L96-L122）に退職列および退職判定出力処理が存在しない。
- 指摘内容:
  - CSV仕様上の必須出力項目が欠落しており、利用者が退職状態をCSVで判別できない。帳票・連携用途で誤運用につながるため、設計逸脱として重大。
- 期待されるテスト:
  - 退職日ありデータで退職列に「退職」が出力されること。
  - 退職日なしデータで退職列が空になること。
  - ヘッダーに退職列が常に含まれること。

### 2. createPersonCsv APIの単体テスト欠落
- 優先度: High
- 種別: テスト不足
- 対象: functions/persons/createPersonCsv/app.py / tests/functions/persons
- 根拠:
  - 設計書: spec/基本設計/機能設計/060.就業者情報/就業者検索.md の「CSV出力ボタン押下」では、権限チェック、0件時COM01003、属性項目動的出力、日付/日時フォーマット、全件出力など多分岐仕様が定義されている。
  - 実装: functions/persons/createPersonCsv/app.py には複数分岐がある一方、tests/functions/persons/test_persons.py はsearchPersons中心で、createPersonCsvのテストケースが存在しない。
- 指摘内容:
  - 重要処理（権限制御・CSV整形・0件時処理・属性型変換）に対する自動テストがなく、仕様逸脱や回帰を検知できない。
- 期待されるテスト:
  - 権限あり/なし（TENANT_ADMIN, MASTER_DATA_ADMIN, RELATION_SYSTEM_ADMIN, DOWNLOAD, 無権限）の分岐。
  - 検索0件時にCOM01003相当レスポンスで終了すること。
  - 属性が日付/日時/文字列の場合のCSVフォーマット。
  - 並び順（テナント就業者ID昇順）と全件取得（ページング非適用）。

### 3. ダウンロード権限エラー時のメッセージ仕様不一致
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/persons/createPersonCsv/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/060.就業者情報/就業者検索.md のバリデーション定義で「ダウンロード権限チェック」のメッセージIDはCOM01001。
  - 実装: functions/persons/createPersonCsv/app.py（L34-L42）では無権限時に FORBIDDEN / 「ダウンロード権限がありません。」を返しており、COM01001の利用が確認できない。
- 指摘内容:
  - メッセージIDベース運用（画面共通制御・多言語・運用監視）前提と不整合。画面側の期待仕様とずれる可能性がある。
- 期待されるテスト:
  - 無権限時に仕様上のメッセージID（COM01001）が返ること。
  - ステータスコードとメッセージIDの組合せが仕様通りであること。

### 4. searchPersonsの単体テストが主要分岐・境界値を未網羅
- 優先度: Middle
- 種別: テスト不足
- 対象: functions/persons/searchPersons/app.py / tests/functions/persons/test_persons.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/060.就業者情報/就業者検索.md の「検索ボタン押下」では、検索基準日、退職済を含む、組織名称/役職名称などの条件分岐が定義されている。
  - 実装: functions/persons/searchPersons/app.py（L27-L56）で複数条件を受け取り PersonRepository.search_persons に委譲。
  - 実装（テスト）: tests/functions/persons/test_persons.py は test_success / test_no_params / test_exception が中心で、includeRetired、baseDate境界、ページング境界、組織/役職条件の検証が不足。
  - テスト観点: spec/050.テスト/テスト観点.md の「境界値」「分岐条件」「異常系」観点に対し不足。
- 指摘内容:
  - 条件分岐が多いAPIに対し、現状テストでは仕様差異や検索条件不具合を十分に検知できない。
- 期待されるテスト:
  - includeRetired true/false で結果集合が変化すること。
  - baseDate 指定/未指定（未指定時9999-12-31扱い）で抽出条件が適用されること。
  - page/itemsPerPage の境界値（最小値、不正値）時の挙動。
  - orgName/positionName の部分一致条件。

### 5. getTenantのテストでtenantSettings.personColumnName検証が不足
- 優先度: Middle
- 種別: テスト不足
- 対象: functions/tenants/getTenant/app.py / tests/functions/tenants/test_tenants.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/060.就業者情報/就業者検索.md の初期表示仕様では、テナント設定の就業者コードカラム名を画面ラベルに使用。
  - 実装: functions/tenants/getTenant/app.py -> shared/services/tenantService.py で tenantSettings.personColumnName を返却可能な構造。
  - 実装（テスト）: tests/functions/tenants/test_tenants.py の成功系で tenantSettings / personColumnName のアサーションが見当たらない。
- 指摘内容:
  - 画面ラベルの要となる返却項目の契約テストが不足しており、API仕様変更時に検知できない。
- 期待されるテスト:
  - tenantSettings.personColumnName が正常返却されること。
  - tenantSettings が空/欠損の場合のデフォルト挙動（必要に応じた仕様確認含む）。

### 6. 日付変換処理の例外捕捉が広すぎ、異常検知が弱い
- 優先度: Low
- 種別: 実装不備
- 対象: functions/persons/createPersonCsv/app.py
- 根拠:
  - 設計書: spec/基本設計/機能設計/060.就業者情報/就業者検索.md ではCSV日付/日時の形式が明示されており、形式異常時の品質担保が重要。
  - 実装: functions/persons/createPersonCsv/app.py の format_date は bare except で全例外を握りつぶし、入力文字列をそのまま返却する。
- 指摘内容:
  - 想定外データ不整合を検知しにくく、品質問題の早期発見を阻害する。運用上の調査性も低下。
- 期待されるテスト:
  - 正常ISO日付、非ISO日付、None相当値での出力確認。
  - 変換失敗時のログ出力または規定フォールバック挙動の検証。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足
  - createPersonCsvの正常系（権限あり、検索ヒット、CSV生成成功）の検証が不足。
- 入力値バリエーション不足
  - searchPersonsの各検索条件（manNumber, 氏名, かな, orgName, positionName, includeRetired）の単独・組合せ検証が不足。
- 境界値不足
  - page/itemsPerPage の境界値、不正値、baseDate境界日での検証が不足。
- 分岐条件不足
  - includeRetired true/false、baseDate 指定/未指定、CSV権限分岐（管理者権限/ダウンロード権限）検証が不足。
- データ入出力不足
  - CSVのヘッダー列完全性（退職列含む）、属性列順序、日付/日時フォーマット、ソート順の検証が不足。
- 非更新確認不足
  - 検索系APIとして更新副作用がないこと、想定外データが混入しないことの確認が不足。
- 異常系不足
  - createPersonCsv 無権限時、テナント情報未取得時、依存先例外時の期待レスポンス検証が不足。

## 総評
- High指摘として、CSV出力仕様の必須項目欠落と createPersonCsv テスト欠落が確認され、リリース前の是正が必要。
- searchPersons/getTenant もテスト観点ベースで分岐・契約検証が不足しており、回帰検知力に課題がある。
- 優先順位は、1) CSV仕様逸脱修正、2) createPersonCsvテスト整備、3) searchPersons/getTenantの観点拡充の順が妥当。

## 残留リスク・確認できなかった範囲
- spec/050.テスト配下のExcel形式テスト仕様書（単体テスト仕様書_就業者検索.xlsx など）の詳細内容までは本レビューで未確認。
- 実DB接続での結合条件・性能（実データ件数時のSQL実行特性）は静的レビュー範囲外。