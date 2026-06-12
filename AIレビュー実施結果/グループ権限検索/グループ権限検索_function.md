# グループ権限検索 レビュー結果

## 指摘事項

### 1. page/pageSize の不正入力でハンドラが未捕捉例外終了する
- 優先度: Middle
- 種別: 実装不備
- 対象: groups/searchGroupPermissions/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/010.システム共通/ページネーション.md](spec/基本設計/機能設計/010.システム共通/ページネーション.md#L39) ではページサイズ既定値のみ定義されており、[spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md#L16-L26) では入力値バリエーションと異常系の確認が必要とされている。
  - 実装: [functions/groups/searchGroupPermissions/app.py](functions/groups/searchGroupPermissions/app.py#L33-L39) で page と pageSize を try の前に int 変換しており、数値以外・0・負数などの不正値に対するバリデーションや捕捉がない。
- 指摘内容:
  - page や pageSize に数値変換できない値が入ると ValueError が try の外で発生し、401/403/500 の API レスポンス整形に入らずハンドラが異常終了する。ページング入力の異常系が未制御で、利用者には統一されたエラーレスポンスが返らない。
- 期待されるテスト:
  - page に文字列を指定した場合に制御されたエラーレスポンスになること。
  - pageSize に 0、負数、極端に大きい値を指定した場合の挙動が仕様どおりであること。

### 2. 権限表示順序の仕様を API が保証していない
- 優先度: Middle
- 種別: 実装不備
- 対象: グループ権限検索機能
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/グループ権限検索.md](spec/基本設計/機能設計/050.権限管理/グループ権限検索.md#L97-L105) では、検索結果の権限表示順序を テナント管理者 → 連携システム管理者 → 連携システム操作者 → マスターデータ管理者 → 閲覧者 → ダウンロード の順に固定している。
  - 実装: [shared/services/groupAuthorityHelper.py](shared/services/groupAuthorityHelper.py#L1-L20) は権限配列をそのまま GroupPermission に変換しており並び替え処理がない。 [shared/repositories/groupmasterRepository.py](shared/repositories/groupmasterRepository.py#L263-L273) でもその結果をそのままレスポンスへ詰めている。
- 指摘内容:
  - API は権限配列の順序を保証しておらず、DB 保存順に依存する。画面仕様どおりの順で表示する責務を API が担う前提なら設計逸脱であり、フロントエンド側で整列する前提でもその責務分担が設計書上で明確でない。
- 期待されるテスト:
  - 同一グループに複数権限を持つデータを用意し、レスポンスの permissions が設計書の順序で返ること。

### 3. 既存テストがハンドラのステータスコードしか見ておらず、検索仕様の中核ロジックを検証できていない
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/groups/test_searchGroupPermissions.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/グループ権限検索.md](spec/基本設計/機能設計/050.権限管理/グループ権限検索.md#L73-L91) では、グループ名部分一致、検索対象権限、ログインユーザー除外、所属ユーザー取得、0件時の扱いまで規定している。 [spec/050.テスト/テスト観点.md](spec/050.テスト/テスト観点.md#L46-L52) では返却値や表示内容の具体的な期待結果確認を求めている。
  - 実装: [tests/functions/groups/test_searchGroupPermissions.py](tests/functions/groups/test_searchGroupPermissions.py#L10-L58) では GroupmasterRepository.search_group_permissions を丸ごとモックし、主に statusCode のみを検証している。実際の検索ロジックは [shared/repositories/groupmasterRepository.py](shared/repositories/groupmasterRepository.py#L179-L273) にあり、テストでは実行されない。
- 指摘内容:
  - 現状のテストでは、グループ名部分一致、権限絞り込み、ログインユーザー除外、並び順、members と permissions の変換、totalCount と page 情報など、仕様の本体を一切検証できない。ハンドラの例外分岐だけが通っても、検索機能としての回帰を検知できない構成になっている。
- 期待されるテスト:
  - groupName 部分一致で対象グループのみ返ること。
  - permissions 複数指定時に想定どおりの絞り込みになること。
  - ログインユーザー所属グループが除外されること。
  - response body の items、totalCount、page、pageSize が期待値になること。

### 4. ページング境界と空結果の仕様確認がテストに不足している
- 優先度: Middle
- 種別: テスト不足
- 対象: グループ権限検索機能
- 根拠:
  - 設計書: [spec/基本設計/機能設計/010.システム共通/ページネーション.md](spec/基本設計/機能設計/010.システム共通/ページネーション.md#L25-L30) では 20 件以下はページング非表示、[spec/基本設計/機能設計/010.システム共通/ページネーション.md](spec/基本設計/機能設計/010.システム共通/ページネーション.md#L47-L49) では表示範囲計算を定義している。 [spec/基本設計/機能設計/050.権限管理/グループ権限検索.md](spec/基本設計/機能設計/050.権限管理/グループ権限検索.md#L91) では 0 件時の扱いを定義している。
  - 実装: [tests/functions/groups/test_searchGroupPermissions.py](tests/functions/groups/test_searchGroupPermissions.py#L34-L51) にデフォルトパラメータ確認はあるが、20 件境界、21 件目以降、最終ページ、0 件時の body 内容は確認していない。
- 指摘内容:
  - ページングは UI 表示仕様も含むため画面テスト側の責務はあるが、API 側でも totalCount、page、pageSize の整合が崩れると画面表示が壊れる。現状は境界値と 0 件系の API 応答が未確認で、ページング不具合の検知力が不足している。
- 期待されるテスト:
  - totalCount が 0、20、21 の各ケースで page と pageSize を含むレスポンス整合性を検証すること。
  - 最終ページで件数が端数になる場合のレスポンス整合性を検証すること。

### 5. systemName 解決がループ内呼び出しになっており、結果件数に比例して DB アクセスが増える
- 優先度: Low
- 種別: 実装不備
- 対象: shared/repositories/groupmasterRepository.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/グループ権限検索.md](spec/基本設計/機能設計/050.権限管理/グループ権限検索.md#L49-L56) では連携システム権限にシステム名を付与して表示することを求めている。
  - 実装: [shared/repositories/groupmasterRepository.py](shared/repositories/groupmasterRepository.py#L247-L254) では permission ごとのループ内で SystemMasterRepository を生成し、systemName を都度取得している。
- 指摘内容:
  - 検索結果件数や権限数が増えるほど systemName 取得クエリが増え、一覧検索の応答時間が悪化しやすい。現時点で機能停止には直結しないが、一覧 API としては性能劣化の温床になる。
- 期待されるテスト:
  - 追加不要。大量データ条件での性能確認、または systemName 解決をまとめて取得する実装へ寄せる検討が必要。

### 6. 画面仕様にある表示文言の整形責務が API とフロントエンドのどちらか不明確
- 優先度: Low
- 種別: 設計確認事項
- 対象: グループ権限検索機能
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/グループ権限検索.md](spec/基本設計/機能設計/050.権限管理/グループ権限検索.md#L49-L56) では権限表示名や連携システム名の付与、[spec/基本設計/機能設計/050.権限管理/グループ権限検索.md](spec/基本設計/機能設計/050.権限管理/グループ権限検索.md#L85-L86) では所属ユーザー未設定時の表示文言を規定している。
  - 実装: [shared/schemas/groupSchemas.py](shared/schemas/groupSchemas.py#L12-L33) では members と permissions を構造化データで返しており、[functions/groups/searchGroupPermissions/app.py](functions/groups/searchGroupPermissions/app.py#L39-L53) でも表示用文字列への整形は行っていない。
- 指摘内容:
  - 現実装は API としては妥当な構造化レスポンスだが、画面仕様に記載された 表示名、改行区切り、未設定文言 をどの層で担保するかが設計書から読み切れない。責務分担が曖昧なままだと、API 側でもフロントエンド側でも保証されず表示差異が発生しうる。
- 期待されるテスト:
  - 不足というより責務分担の明確化が先。API 契約を構造化データとするなら、フロントエンド側に表示整形テストが必要。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足
  - groupName 部分一致、permissions 指定あり、ログインユーザー除外ありの検索結果が正しく返るケースがない。
  - response body の items、totalCount、page、pageSize を正常系で確認していない。
- 入力値バリエーション不足
  - permissions 複数指定時の絞り込み、groupName 空文字と指定ありの差、page と pageSize の組み合わせが未確認。
- 境界値不足
  - totalCount が 0、20、21 の境界、最終ページ端数、page と pageSize の 0・負数・非数値が未確認。
- 分岐条件不足
  - 権限順序、連携システム権限の systemName 付与、権限未指定時の全件検索、ログインユーザー所属グループ除外の分岐が未確認。
- データ入出力不足
  - members と permissions の構造、systemName の有無、totalCount の正確性など返却値の内容確認が不足している。
- 非更新確認不足
  - 読み取り専用 API として、検索実行時に更新系処理が走らないことを確認する観点がない。
- 異常系不足
  - page と pageSize の不正値、依存先例外発生時のレスポンス body、想定外入力時の挙動確認が不足している。

## 総評
- High 相当の即時マージ停止級の問題は確認できませんでしたが、Middle 指摘の 4 件は検索 API の信頼性と回帰検知力に直接影響します。
- とくに page と pageSize の未捕捉例外、ならびに repository を完全にモックした現行テスト構成は、利用時障害と見逃しの両方につながります。
- 先に入力異常系の制御と、検索仕様を実データで検証するテスト追加を優先すべきです。

## 残留リスク・確認できなかった範囲
- [shared/core/common.py](shared/core/common.py#L10-L35) の get_current_user は現在スタブ実装に見えるため、本番相当の認証情報からの UnauthorizedException、ForbiddenException 経路は今回の対象コードだけでは確認できていません。
- 画面表示文言の整形責務が API 側かフロントエンド側かを規定する API インターフェイス設計書は、今回確認した範囲では見つけ切れていません。
