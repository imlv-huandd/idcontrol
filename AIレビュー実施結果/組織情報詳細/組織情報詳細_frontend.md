# 組織情報詳細 レビュー結果

> レビュー実施日: 2026-06-09  
> 対象ファイル: `OrganizationDetail.jsx` / `AbolishDialog.jsx`  
> 参照設計書: `spec/基本設計/機能設計/070.組織情報/組織情報詳細.md`  
> 備考: `messageJP.js` / `UserAuthority.js` / `InputType.js` / `AuthContext.jsx` / `spec/050.テスト/テスト観点.md` は SubAgent 探索不可のため未参照。未確認範囲は「残留リスク」セクションに記載。

---

## 指摘事項

---

### 1. `setError` 未定義によるランタイムクラッシュ

- 優先度: `High`
- 種別: `実装不備`
- 対象: `OrganizationDetail.jsx`
- 根拠:
  - 設計書: `組織情報詳細.md` — テナント情報取得・組織属性取得のエラー時処理が定義されている
  - 実装: `fetchTenantOrganizationIdColumnName` および `fetchOrganizationAttribute` の catch ブロックで `setError(...)` を呼び出しているが、このコンポーネントに `setError` という state setter は存在しない（`setErrors` が正しい名称であり引数もオブジェクト形式が必要）
- 指摘内容:
  - `setError` は定義されていないため、catch ブロックに入ると `ReferenceError` が発生しアプリがクラッシュする
  - 初期表示時（テナント情報取得・組織属性取得）の通信失敗という高頻度シナリオでクラッシュが発生するリスクがある
- 期待されるテスト:
  - `fetchTenantOrganizationIdColumnName` が HTTP エラーを返した場合にクラッシュしないこと
  - `fetchOrganizationAttribute` が例外をスローした場合にクラッシュしないこと（エラーメッセージが表示されること）

---

### 2. AbolishDialog — 廃止日の `onChange` で state に誤ったオブジェクト形式を設定

- 優先度: `High`
- 種別: `実装不備`
- 対象: `AbolishDialog.jsx`
- 根拠:
  - 設計書: `組織情報詳細.md` — 廃止日は必須・日付形式のバリデーションが必要
  - 実装: `abolishDate` の初期値は `""` (string) だが、`onChange` で `setAbolishDate(p => ({ ...p, baseDate: e.target.value }))` を呼んでおり、最初の入力後に `abolishDate` が `{ baseDate: "..." }` オブジェクトになる
- 指摘内容:
  - 初回入力後、`abolishDate == ""` の判定が常に `false` となり必須チェックが機能しなくなる
  - `handleAbolish(orgId, abolishDate, ...)` に文字列ではなくオブジェクトが渡されるため、後続処理でも誤動作する
  - 正しくは `setAbolishDate(e.target.value)` とすべき
- 期待されるテスト:
  - 廃止日未入力で廃止ボタンを押下した場合に必須エラーメッセージが表示されること
  - 廃止日を入力して廃止ボタンを押下した場合に `handleAbolish` に正しい日付文字列が渡されること

---

### 3. AbolishDialog — 廃止日の日付形式バリデーションが実質機能しない

- 優先度: `High`
- 種別: `実装不備`
- 対象: `AbolishDialog.jsx`
- 根拠:
  - 設計書: `組織情報詳細.md` — 廃止日は必須・日付形式チェックが必要（バリデーション No.1 COM02006）
  - 実装: `const dateRegex = /^\d{4}\/\d{2}\/\d{2}$/;` が定義されているが一度も使われていない。直後の `if (!abolishDate)` は「空文字でない」確認を既に通過しているため常に `false` となり、日付形式チェックが完全にスキップされる
- 指摘内容:
  - 任意の文字列（例: "abc"）を廃止日として入力した場合でも、バリデーションエラーが発生しない
  - `dateRegex.test(abolishDate)` を使った正しい形式チェックが必要
- 期待されるテスト:
  - 不正な形式（例: `"2026/13/99"` `"abc"` ）を廃止日に入力した場合に COM02006 エラーが表示されること
  - 正常な形式を入力した場合にエラーが表示されないこと

---

### 4. 編集ボタンの権限条件が設計と不一致（テナント管理者を誤って含めている）

- 優先度: `High`
- 種別: `実装不備`
- 対象: `OrganizationDetail.jsx`
- 根拠:
  - 設計書: `組織情報詳細.md` — 「マスターデータ管理者権限を持つユーザーのみ編集ボタンが表示され、組織情報の編集が可能となる」および ボタンエリア No.2 「詳細表示モードかつマスターデータ管理者のみ表示」
  - 実装: `(isAuthorityTenantAdmin(loginUser.authorities) || isAuthorityMasterDataAdmin(loginUser.authorities))` — `isAuthorityTenantAdmin` が OR で結合されており、テナント管理者も編集ボタンが表示されてしまう
- 指摘内容:
  - 設計書ではマスターデータ管理者権限のみ対象。テナント管理者が誤って編集権限を持つことになり、認可不備となる
  - `isAuthorityTenantAdmin` の条件を削除し `isAuthorityMasterDataAdmin` のみにすべき
- 期待されるテスト:
  - マスターデータ管理者権限のユーザーで詳細表示モードを開いた場合に編集ボタンが表示されること
  - テナント管理者権限のユーザーで詳細表示モードを開いた場合に編集ボタンが表示されないこと
  - 上記どちらの権限も持たないユーザーで編集ボタンが表示されないこと

---

### 5. `historyCheck()` の演算子優先度バグによる誤った判定

- 優先度: `High`
- 種別: `実装不備`
- 対象: `OrganizationDetail.jsx`
- 根拠:
  - 設計書: `組織情報詳細.md` — バリデーション No.6 「更新前後で有効開始日に差分がある場合、変更前の有効開始日＜有効開始日＜変更前の有効終了日であること」
  - 実装: `if (oldStart.getTime() !== start.getTime() && oldStart >= start || oldEnd < start)` — `&&` が `||` より優先されるため、実際の評価は `(差分あり && oldStart >= start) || (oldEnd < start)` となる
- 指摘内容:
  - `oldEnd < start` の部分が「有効開始日に差分がある場合」という前提条件なしに独立して評価される
  - 例: 有効開始日を変更していない場合でも `oldEnd < start` が true になればエラーとなる（有効なレコードでは通常 `oldEnd >= oldStart` なので実害は限定的だが、ロジックが設計書と不一致）
  - 正しい条件: `oldStart.getTime() !== start.getTime() && (start <= oldStart || start >= oldEnd)`（差分がある場合のみ範囲チェック）
- 期待されるテスト:
  - 有効開始日を変更せずに他の項目のみ変更した場合に履歴追加チェックエラーが発生しないこと
  - 有効開始日を変更前の有効開始日より前に設定した場合にエラーが発生すること
  - 有効開始日を変更前の有効終了日以降に設定した場合にエラーが発生すること
  - 有効開始日を `(変更前有効開始日, 変更前有効終了日)` の範囲内に設定した場合にエラーが発生しないこと

---

### 6. `if (startDate != "")` のブロック欠如による `setDefaultHistory` の無条件実行

- 優先度: `Middle`
- 種別: `実装不備`
- 対象: `OrganizationDetail.jsx`
- 根拠:
  - 設計書: `組織情報詳細.md` — 「パラメータなし（新規登録モード）」と「パラメータあり（詳細表示モード）」で初期表示を分けている
  - 実装: `useEffect(() => { ... if (startDate != "") setHistorySelect(...); setDefaultHistory(...); }, [])` — 波括弧なしの `if` 文のため `setDefaultHistory` は `startDate` が空でも常に実行される（インデントは揃っているが JavaScript 構文上は `if` の外側）
- 指摘内容:
  - 新規登録モード（`startDate == ""`）でも `setDefaultHistory(`~ `)` が設定され、その後の `useEffect([modalMode])` 内で `handleChangeHistorySelect(defaultHistory)` が呼ばれて空文字列の分割処理が実行される
  - `handleChangeHistorySelect("")` の `split("~")` 処理で `list[1].trim()` が存在しない場合にランタイムエラーが発生する可能性がある
- 期待されるテスト:
  - 新規登録モードで開いた場合に `handleChangeHistorySelect` が呼ばれないこと
  - 詳細表示モードで開いた場合に正しい初期履歴が設定されること

---

### 7. `orgNameDisplay` の HTML `maxLength` が設計と不一致（200文字、正しくは100文字）

- 優先度: `Middle`
- 種別: `実装不備`
- 対象: `OrganizationDetail.jsx`
- 根拠:
  - 設計書: `組織情報詳細.md` — 基本情報エリア No.9 `organization.org_name_display` / バリデーション No.1 「組織名称（表示名）:必須、100文字以内」
  - 実装: `<input ... id="orgNameDisplay" ... maxLength="200" />` — HTML の入力制限が200文字になっており、100文字を超えて入力できてしまう（バリデーション側は `> 100` で正しく100文字以内チェック済み）
- 指摘内容:
  - HTML の `maxLength` は表示レベルの入力制限であり、バリデーションと一致させることでUXを改善し、設計との整合性を保つべき
  - `maxLength="100"` に変更すること
- 期待されるテスト:
  - 組織名称（表示名）に101文字入力した場合にバリデーションエラー(COM02002)が表示されること
  - 100文字入力した場合にバリデーションエラーが表示されないこと

---

### 8. `inputChangesCheck()` で `orgAttr`（追加情報）の変更が未考慮

- 優先度: `Middle`
- 種別: `実装不備`
- 対象: `OrganizationDetail.jsx`
- 根拠:
  - 設計書: `組織情報詳細.md` — バリデーション No.5 「更新前後で有効開始日以外の項目に差分があること」
  - 実装: `inputChangesCheck()` は `tenantOrgId` / `orgName` / `orgNameAll` / `orgNameDisplay` の4項目しか比較しておらず、`orgAttr`（追加情報）や `displayOrder` の変更が差分として評価されない
- 指摘内容:
  - 有効開始日を変更し、基本4項目は変更せず追加情報のみ変更した場合、「更新前後で有効開始日以外の項目に差分がない」と誤判定されエラーとなる
  - `orgAttr` および `displayOrder` の差分比較を追加する必要がある
- 期待されるテスト:
  - 有効開始日を変更し追加情報のみ変更した場合に変更有無チェックエラーが発生しないこと
  - 有効開始日を変更し基本情報も追加情報も変更しなかった場合にエラーが発生すること

---

### 9. `handleUpdate()` で `organizationId`（prop）を使用しており、`handleCreate` の `orgId`（state）と非対称

- 優先度: `Middle`
- 種別: `実装不備`
- 対象: `OrganizationDetail.jsx`
- 根拠:
  - 設計書: 特定の記載なし（実装ロジックの問題）
  - 実装: `handleCreate` は `orgId`（state）を URLパラメータとして使用し、登録完了後に `setOrgId(data.organizationId)` でstateを更新する。一方 `handleUpdate` は `organizationId`（prop）を使用しており、新規登録→詳細→編集→更新フローでの整合性が不明確
- 指摘内容:
  - `organizationId` prop は親コンポーネントが最初に渡した値であり、新規登録後に発行された `organizationId` を反映しない可能性がある
  - `handleUpdate` でも `orgId` state を使用するべきか確認が必要
- 期待されるテスト:
  - 新規登録完了後に詳細モードに切り替え、編集→更新した場合に正しい組織IDで更新APIが呼ばれること

---

### 10. `formData` の初期値に `displayOrder` が含まれておらず `undefined` になる

- 優先度: `Middle`
- 種別: `実装不備`
- 対象: `OrganizationDetail.jsx`
- 根拠:
  - 設計書: `組織情報詳細.md` — 基本情報エリア No.16/17 「表示順」が定義されている
  - 実装: `useState({ tenantOrgId: '', orgName: '', ..., times: '' })` に `displayOrder` が含まれていない。JSX では `<input ... value={formData.displayOrder} ... />` で参照されているため初期値は `undefined`
- 指摘内容:
  - `undefined` を controlled input の `value` に設定すると React が uncontrolled から controlled への切り替え警告を出す可能性があり、新規登録時に表示順が送信されない
  - `formData` の初期値に `displayOrder: ''` または `displayOrder: 0` を追加すること
- 期待されるテスト:
  - 新規登録モードで表示順に値を入力した場合に、APIリクエストの body に `displayOrder` が含まれること

---

### 11. `isAbolished` チェックボックスの `onChange` で `e.target.value` を使用（正しくは `e.target.checked`）

- 優先度: `Middle`
- 種別: `実装不備`
- 対象: `OrganizationDetail.jsx`
- 根拠:
  - 設計書: `組織情報詳細.md` — 「廃止」は詳細表示/編集モード共通で参照のみ
  - 実装: `onChange={e => { handleChange('isAbolished', e.target.value); }}` — チェックボックスには `e.target.checked` を使うべきところ `e.target.value` が使われている。現状は `disabled` 属性があるため動作には影響しないが、将来 `disabled` を外した場合に誤動作する
- 指摘内容:
  - `e.target.value` はチェックボックスでは常に `"on"` になるため、boolean 型の `isAbolished` に正しい値が設定されない
  - `e.target.checked` に修正すること
- 期待されるテスト:
  - 廃止チェックボックスが詳細表示モード・編集モードで操作不可であること

---

### 12. `AbolishDialog` が `OrganizationDetail.jsx` から import されているが JSX 内で使用されていない

- 優先度: `Middle`
- 種別: `設計確認事項`
- 対象: `OrganizationDetail.jsx` / `AbolishDialog.jsx`
- 根拠:
  - 設計書: `組織情報詳細.md` — 廃止機能は組織検索画面側のアクションとして設計されている可能性があるが、`OrganizationDetail.jsx` 内に `<AbolishDialog>` のレンダリング記述が存在しない
  - 実装: `import AbolishDialog from './AbolishDialog';` は存在するが JSX 内に `<AbolishDialog ...>` が一切ない
- 指摘内容:
  - 廃止ダイアログを `OrganizationDetail.jsx` から呼び出す設計があるなら実装漏れ
  - 親コンポーネント（組織検索画面）から呼び出す設計であればこの import は不要な dead import となる
  - どちらが正しいか設計を確認の上、import の削除または実装の追加を行うこと
- 期待されるテスト:
  - 廃止ボタンが適切な権限で表示・非表示されること（設計の確認後に定義）

---

### 13. 履歴セレクトボックスの表示形式が設計と異なる可能性

- 優先度: `Low`
- 種別: `設計確認事項`
- 対象: `OrganizationDetail.jsx`
- 根拠:
  - 設計書: `組織情報詳細.md` — 「表示形式は「yyyy/mm/dd ～ yyyy/mm/dd」。有効終了日が9999/12/31の場合は「yyyy/mm/dd ～」」（スラッシュ区切り、全角波ダッシュ）
  - 実装: `` `${item.startDate} ~ ${item.endDate != "9999-12-31" ? item.endDate : ""}` `` — API レスポンスが `yyyy-MM-dd`（ハイフン区切り）の場合、設計の `yyyy/mm/dd` と異なる。また `~` が半角チルダでありデザイン上「～」（全角波ダッシュ）と異なる可能性がある
- 指摘内容:
  - 日付フォーマット変換（ハイフン→スラッシュ）の処理が必要か確認すること
  - `historySelect` と API レスポンスの `startDate/endDate` を照合する際にも `"9999-12-31"` でのハードコード比較があり、APIレスポンスフォーマットとの整合性を確認すること
- 期待されるテスト:
  - 詳細表示モードで履歴セレクトボックスに `yyyy/mm/dd ～ yyyy/mm/dd` 形式で表示されること
  - 有効終了日が `9999/12/31` の場合に `yyyy/mm/dd ～` の形式で表示されること

---

### 14. `historyGroup` の `option` 要素に `key` prop が欠如

- 優先度: `Low`
- 種別: `実装不備`
- 対象: `OrganizationDetail.jsx`
- 根拠:
  - 設計書: 特定の記載なし（React のベストプラクティス）
  - 実装: `historyGroup.map(item => (<option value={item}>{item}</option>))` — `key` prop がない
- 指摘内容:
  - React の key 警告が発生する
  - `<option key={item} value={item}>` に変更すること
- 期待されるテスト:
  - 複数の履歴がある場合にセレクトボックスに正しい件数の選択肢が表示されること

---

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）

> 注意: `spec/050.テスト/テスト観点.md` が参照できないため、本設計書と実装コードから導出した観点で整理する

### 正常系不足

| No. | 観点 | 対象ファイル |
|-----|------|------------|
| T-N-01 | 新規登録モードで全必須項目を正しく入力し「登録」ボタン押下→確認ダイアログ→登録完了メッセージ→詳細表示モードに遷移すること | OrganizationDetail.jsx |
| T-N-02 | 詳細表示モードで組織情報が正しく表示されること（各フィールドのマッピング確認） | OrganizationDetail.jsx |
| T-N-03 | 詳細表示モードで複数履歴がある場合に履歴セレクトボックスに全履歴が降順で表示されること | OrganizationDetail.jsx |
| T-N-04 | 履歴セレクトボックスで別の履歴を選択した場合に対応するデータが各フィールドに表示されること | OrganizationDetail.jsx |
| T-N-05 | 編集モードで有効開始日を変更せず更新→詳細表示モードに戻ること | OrganizationDetail.jsx |
| T-N-06 | 編集モードで有効開始日を変更して更新（履歴追加）→詳細表示モードに戻ること | OrganizationDetail.jsx |
| T-N-07 | マスターデータ管理者権限ユーザーで詳細表示モードを開いた場合に編集ボタンが表示されること | OrganizationDetail.jsx |
| T-N-08 | マスターデータ管理者権限ユーザーが廃止日を正しく入力し廃止ダイアログを確定した場合に `handleAbolish` が呼ばれること | AbolishDialog.jsx |

### 入力値バリエーション不足

| No. | 観点 | 対象ファイル |
|-----|------|------------|
| T-I-01 | テナント組織IDに100文字ちょうど入力→エラーなし | OrganizationDetail.jsx |
| T-I-02 | テナント組織IDに101文字入力→COM02002 エラー表示 | OrganizationDetail.jsx |
| T-I-03 | 組織名称（表示名）に100文字→エラーなし、101文字→COM02002エラー | OrganizationDetail.jsx |
| T-I-04 | 組織名称（全）に200文字→エラーなし、201文字→COM02002エラー | OrganizationDetail.jsx |
| T-I-05 | 追加情報の各 inputType（TEXT_ALPHA / TEXT_NUMERIC / TEXT_ALPHANUM / TEXT_HALF_WIDTH / TEXT_FULL_WIDTH / INTEGER / INTEGER_POSITIVE / FLAG / DATE / DATETIME）に対し、適合する値・不適合な値を入力した場合に正しいエラーメッセージが表示されること | OrganizationDetail.jsx |
| T-I-06 | 追加情報の必須項目（notEmpty=true）が空のまま登録→COM02001エラー表示 | OrganizationDetail.jsx |
| T-I-07 | 廃止日に不正な日付フォーマットを入力→COM02006エラー表示 | AbolishDialog.jsx |

### 境界値不足

| No. | 観点 | 対象ファイル |
|-----|------|------------|
| T-B-01 | 有効開始日を変更前有効開始日と同日に設定→履歴追加チェックエラーが発生しないこと（変更なしとみなされる） | OrganizationDetail.jsx |
| T-B-02 | 有効開始日を変更前有効終了日の前日に設定→エラーなし | OrganizationDetail.jsx |
| T-B-03 | 有効開始日を変更前有効終了日と同日に設定→エラー | OrganizationDetail.jsx |
| T-B-04 | 有効開始日を変更前有効開始日より1日前に設定→エラー | OrganizationDetail.jsx |
| T-B-05 | 追加情報の文字列項目に maxLength ちょうどの文字→エラーなし、maxLength+1文字→エラー | OrganizationDetail.jsx |

### 分岐条件不足

| No. | 観点 | 対象ファイル |
|-----|------|------------|
| T-D-01 | 新規登録モードで有効終了日・廃止チェックボックスが非表示になること | OrganizationDetail.jsx |
| T-D-02 | 詳細表示モードで全入力フィールドが編集不可（disabled）であること | OrganizationDetail.jsx |
| T-D-03 | 編集モードで追加情報の `notChange=true` 項目が編集不可であること | OrganizationDetail.jsx |
| T-D-04 | 編集モードで `notChange=true` の項目以外が編集可能であること | OrganizationDetail.jsx |
| T-D-05 | テナント管理者権限ユーザーで詳細表示モードを開いた場合に編集ボタンが表示されないこと | OrganizationDetail.jsx |
| T-D-06 | 編集モードでキャンセルボタンを押下した場合、変更有の場合は確認ダイアログが表示され詳細表示に戻ること | OrganizationDetail.jsx |
| T-D-07 | 編集モードでキャンセルボタンを押下した場合、変更無の場合は確認ダイアログなしで詳細表示に戻ること | OrganizationDetail.jsx |
| T-D-08 | 廃止日未入力で廃止ボタンを押下→COM02001エラー表示（AbolishDialogの state が正しく string を保持していること） | AbolishDialog.jsx |

### データ入出力不足

| No. | 観点 | 対象ファイル |
|-----|------|------------|
| T-A-01 | 登録APIリクエストボディに全フォームフィールド（displayOrder含む）が正しく含まれること | OrganizationDetail.jsx |
| T-A-02 | 更新APIリクエストが `organizationId` state（prop ではなく）を使用していること | OrganizationDetail.jsx |
| T-A-03 | 詳細表示モードで `orgAttr` が API レスポンスと `defaultOrgAttr` のマージで正しく表示されること | OrganizationDetail.jsx |

### 非更新確認不足

| No. | 観点 | 対象ファイル |
|-----|------|------------|
| T-U-01 | 変更有無チェック: 有効開始日のみ変更し追加情報を変更した場合にエラーが発生しないこと（`orgAttr` の差分が考慮されること） | OrganizationDetail.jsx |
| T-U-02 | 変更有無チェック: 有効開始日・基本情報・追加情報すべて変更した場合にエラーが発生しないこと | OrganizationDetail.jsx |

### 異常系不足

| No. | 観点 | 対象ファイル |
|-----|------|------------|
| T-E-01 | `fetchTenantOrganizationIdColumnName` が HTTP エラーを返した場合に InforDialog が表示されること（setError ではなく正しいエラー処理） | OrganizationDetail.jsx |
| T-E-02 | `fetchOrganizationAttribute` が例外をスローした場合に画面がクラッシュせずエラーが表示されること | OrganizationDetail.jsx |
| T-E-03 | `fetchOrganizationData` が `histories.length == 0` を返した場合に COM01009 エラーが表示されてダイアログが閉じること | OrganizationDetail.jsx |
| T-E-04 | 登録API が HTTP エラーを返した場合に InforDialog が表示されること | OrganizationDetail.jsx |
| T-E-05 | 更新API が HTTP エラーを返した場合に InforDialog が表示されること | OrganizationDetail.jsx |
| T-E-06 | 登録/更新API がネットワークエラーをスローした場合に InforDialog が表示されること | OrganizationDetail.jsx |

---

## 総評

High 指摘が5件あり、このままリリースすると認可不備（テナント管理者が組織編集可能）・ランタイムクラッシュ（通信エラー時に `setError` 未定義でアプリ停止）・廃止ダイアログの完全な機能不全（廃止日が正しく取得できず日付バリデーションも機能しない）が発生する。  
特に **指摘1（`setError` 未定義）** と **指摘4（権限不備）** は初回操作で再現する重大な不具合であり、優先的に修正とテスト追加が必要。  
テストコードが未作成の現状では、上記の High 指摘の大半が発見されないリスクがある。最低限、権限制御・バリデーション全ケース・API 通信異常系のテストを作成することを強く推奨する。

---

## 残留リスク・確認できなかった範囲

- `spec/050.テスト/テスト観点.md` が未参照のため、プロジェクト共通の単体テスト観点との突き合わせができていない。テスト観点ファイルを参照し、本レビューの「テスト不足の整理」を補完すること
- `frontend/src/constants/messageJP.js` が未参照のため、メッセージID（COM01006、COM01008、ORG02002 等）が実際に定義されているか確認できていない
- `frontend/src/common/UserAuthority.js` が未参照のため、`isAuthorityTenantAdmin` / `isAuthorityMasterDataAdmin` の権限判定ロジックが正しく実装されているか確認できていない。指摘4の修正内容がこれらの関数仕様に依存する
- `frontend/src/common/InputType.js` が未参照のため、`InputType` 定数と `renderInputByType` の `switch` 分岐（文字列リテラルとの比較）の整合性が未確認
- `frontend/src/auth/AuthContext.jsx` が未参照のため、`loginUser.authorities` の構造が未確認
- 組織検索画面側（親コンポーネント）のソースが未参照のため、`AbolishDialog` の呼び出し元・廃止機能の実装範囲が確認できていない
- バックエンド API（`getOrganization` / `createOrganization` / `updateOrganization` / `getTenant` / `getOrganizationAttribute`）のレスポンス仕様が未参照のため、日付フォーマット（`yyyy-MM-dd` vs `yyyy/mm/dd`）の整合性が未確認
