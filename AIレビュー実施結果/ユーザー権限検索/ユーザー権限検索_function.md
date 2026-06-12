# ユーザー権限検索 レビュー結果

## 指摘事項

### 1. 認証ユーザー取得がダミー実装のままで、検索対象の除外条件とテナント境界が実行時コンテキストに連動していない
- 優先度: High
- 種別: 実装不備
- 対象: shared/core/common.py / functions/users/searchUsersPermissions/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md) では、ログインユーザー自身を検索結果に含めないこと、およびログインユーザーの文脈で検索することが前提になっている。
  - 実装: [shared/core/common.py#L10-L36](shared/core/common.py#L10-L36) の get_current_user() が固定値のユーザー情報を返すダミー実装になっており、[functions/users/searchUsersPermissions/app.py#L31-L32](functions/users/searchUsersPermissions/app.py#L31-L32) ではその値を tenant_id と login_user_id として検索条件に使用している。
- 指摘内容:
  - 実行中の認証情報ではなく固定のユーザー情報を検索条件へ流し込んでいるため、テナント絞り込み、自分自身の除外、認可関連の前提がリクエスト実体と一致しない。検索結果の妥当性だけでなく、認証・認可境界の観点でもリリース不可レベルの不備である。
- 期待されるテスト:
  - 認証コンテキストから tenant_id と user_id を取得した場合に、そのユーザー自身が検索結果から除外されること。
  - 異なる tenant_id のデータが返らないこと。
  - 認証情報が欠落または不正な場合に 401 / 403 が返ること。

### 2. グループ権限の返却が検索フラグに依存しており、設計どおりに常時表示されないうえ、重複排除と表示順序制御も未実装
- 優先度: Middle
- 種別: 実装不備
- 対象: shared/repositories/usermasterRepository.py / shared/services/userAuthorityHelper.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md#L72-L85) では検索結果の各行にグループ権限を表示するとあり、同ファイルの備考ではグループ権限は重複を 1 つにまとめ、指定順序で表示すると定義されている。
  - 実装: [shared/repositories/usermasterRepository.py#L175-L199](shared/repositories/usermasterRepository.py#L175-L199) では include_group_permissions が true の場合にのみ groupPermissions を構築しており、複数グループから集約した権限の重複排除も行っていない。[shared/services/userAuthorityHelper.py#L4-L18](shared/services/userAuthorityHelper.py#L4-L18) でも権限の順序制御は行っていない。
- 指摘内容:
  - グループ権限を含めるチェックボックスは検索条件への反映を切り替える仕様であり、検索結果のグループ権限表示の有無を切り替える仕様ではない。現状はチェックオフ時に groupPermissions が空になる可能性があり、さらに複数グループ所属時の重複排除と表示順が設計から逸脱している。
- 期待されるテスト:
  - includeGroupPermissions が false でも groupPermissions が結果表示用に返ること。
  - 複数グループ所属で同一権限が重複していても 1 件に集約されること。
  - userPermissions / groupPermissions が設計書の表示順で返ること。

### 3. 検索結果のソート順が設計書のメールアドレス昇順と一致していない
- 優先度: Middle
- 種別: 実装不備
- 対象: shared/repositories/usermasterRepository.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md#L42-L61](spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md#L42-L61) に検索結果のソート順はメールアドレスの昇順と記載されている。
  - 実装: [shared/repositories/usermasterRepository.py#L140](shared/repositories/usermasterRepository.py#L140) では Usermaster.login_id.asc() を使用している。
- 指摘内容:
  - 返却順序が設計と異なるため、UI の表示順が仕様どおりにならない。利用者の認識と実データの並びがずれ、検索結果の確認性を損なう。
- 期待されるテスト:
  - メールアドレスの昇順で結果が返ることを、複数件データで確認すること。
  - loginId と mailAddress の並び順が異なるデータで、仕様どおり mailAddress が優先されることを確認すること。

### 4. page / itemsPerPage の入力検証が共通ページネーション仕様に追従していない
- 優先度: Middle
- 種別: 実装不備
- 対象: functions/users/searchUsersPermissions/app.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/010.システム共通/ページネーション.md](spec/基本設計/機能設計/010.システム共通/ページネーション.md) では page と pageSize は 1 以上を前提とする。
  - 実装: [functions/users/searchUsersPermissions/app.py#L58-L69](functions/users/searchUsersPermissions/app.py#L58-L69) では int() 変換のみで page / itemsPerPage を repository と response へ渡しており、0 や負数を事前に排除していない。
- 指摘内容:
  - 非数値は ValueError で 400 になる一方、0 や負数は repository 実行まで進んでから response 生成時または DB 実行時に不正値として露見する。入力検証の責務が曖昧で、共通仕様に沿った一貫した 400 応答が保証されていない。
- 期待されるテスト:
  - page=0、page=-1、itemsPerPage=0、itemsPerPage=-1 で 400 になること。
  - page や itemsPerPage に非数値を渡した場合に 400 になること。
  - 1 件、20 件、21 件の境界で totalCount / page / pageSize が正しいこと。

### 5. API テストが空結果モック中心で、仕様上重要な検索条件と返却内容を検証できていない
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/users/test_searchUsersPermissions.py
- 根拠:
  - 設計書: [spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md](spec/基本設計/機能設計/050.権限管理/ユーザー権限検索.md#L72-L85) では、ログインID・ユーザー名・かな・メールアドレスの部分一致、権限検索、グループ権限検索、自分自身の除外、グループ権限表示など複数の業務ルールが定義されている。
  - 実装: [tests/functions/users/test_searchUsersPermissions.py#L11-L150](tests/functions/users/test_searchUsersPermissions.py#L11-L150) では search_users_with_Authority の戻り値を主に空配列へ固定し、200/401/403/500 と page / pageSize の最低限の確認に留まっている。
- 指摘内容:
  - 既存テストでは、検索条件が repository に正しく渡るか、返却 item に userPermissions / groupPermissions が正しく含まれるか、自分自身が除外されるか、設計どおりの並びと内容になるかを検証できていない。これでは設計逸脱を見逃す構成である。
- 期待されるテスト:
  - loginId / userName / userNameKana / mailAddress の部分一致条件が repository に正しく伝播すること。
  - permissions の複数指定が分割されて渡されること。
  - includeGroupPermissions の true / false で検索条件への反映が切り替わること。
  - 返却 item に userPermissions / groupPermissions / totalCount が仕様どおり含まれること。
  - ログインユーザー自身が検索結果に含まれないこと。

### 6. 単体テスト観点で要求される入力バリエーション・境界値・異常系のカバーが不足している
- 優先度: Middle
- 種別: テスト不足
- 対象: tests/functions/users/test_searchUsersPermissions.py / spec/050.テスト/テスト観点.md
- 根拠:
  - 設計書: [spec/050.テスト/テスト観点.md#L23-L68](spec/050.テスト/テスト観点.md#L23-L68) では、正常系、入力値バリエーション、境界値、分岐条件、データ入出力、非更新確認、異常系の観点で単体テストを整備することが求められている。
  - 実装: [tests/functions/users/test_searchUsersPermissions.py#L11-L150](tests/functions/users/test_searchUsersPermissions.py#L11-L150) にあるのは正常系の最小ケースと一部例外系のみで、ValueError の 400、ページネーション境界、権限フィルター分岐、0 件時のデータ内容、includeGroupPermissions の偽値系などが確認されていない。
- 指摘内容:
  - 現状のテストでは、条件分岐と境界値の欠落が大きく、API の不正入力・仕様逸脱・分岐漏れを継続的に検出できない。単体テスト観点に照らしても不足が明確である。
- 期待されるテスト:
  - ValueError を返す入力の 400 応答確認。
  - 0 件、1 件、20 件、21 件の検索結果確認。
  - includeGroupPermissions に false / true / 1 / yes / 空文字を与えたときの分岐確認。
  - permissions 未指定、単一指定、複数指定、空文字混在の分岐確認。

## テスト不足の整理（spec/050.テスト/テスト観点.md 準拠）
- 正常系不足
  - 検索結果が 1 件以上返るケースで、items 内の userPermissions / groupPermissions / totalCount / page / pageSize を具体値で検証するテストがない。
- 入力値バリエーション不足
  - permissions の未指定、単一、複数、空文字混在、includeGroupPermissions の true / false / 1 / yes / 空文字を比較するテストがない。
- 境界値不足
  - page / itemsPerPage の 0、負数、20 件、21 件などページネーション境界のテストがない。
- 分岐条件不足
  - includeGroupPermissions の真偽による検索条件分岐、グループ権限を検索対象に含める分岐、0 件結果時の応答を確認するテストがない。
- データ入出力不足
  - mailAddress 昇順ソート、権限の表示順、groupPermissions の重複排除、レスポンス item の各フィールド内容を検証するテストがない。
- 非更新確認不足
  - ログインユーザー自身が検索結果に含まれないこと、条件外ユーザーが含まれないことを確認するテストがない。
- 異常系不足
  - ValueError による 400、page / itemsPerPage の不正値、認証情報欠落時の処理を確認するテストがない。

## 総評
- High 指摘は、認証コンテキスト取得がダミー実装のままである点で、テナント境界と自ユーザー除外の前提を崩している。
- 実装面では、グループ権限表示とソート順が設計とずれており、検索結果の内容自体が仕様どおりとは言えない。
- テスト面では、検索ロジックの本体となる分岐と境界値がほぼ未検証で、設計逸脱を検出しにくい。
- 修正優先順位は、認証コンテキストの是正、検索結果生成ロジックの設計一致、最後に単体テスト拡充の順が妥当である。

## 残留リスク・確認できなかった範囲
- 実運用の認証基盤側で get_current_user() が差し替えられる前提かどうかは、このレビュー範囲だけでは確認できていない。
- repository を直接検証する単体テストや、実 DB を使った検索ロジックの検証コードは確認できなかった。
- フロントエンド側で groupPermissions を別 API から補完している可能性までは確認していない。