<link href="https://raw.githubusercontent.com/simonlc/Markdown-CSS/master/markdown.css" rel="stylesheet"></link>

# インターフェイス定義

**K0011-1 電子承認Create!Webフロー**


## 更新履歴

| 更新日     | 更新者 | 更新内容 |
| ---------- | ------ | -------- |
| 2025/12/22 |        | 新規作成 |

## 目次

- [24-1【直接雇用スタッフ】スタッフ_上申書および雇用契約書](#24-1直接雇用スタッフスタッフ_上申書および雇用契約書)
- [24-2【直接雇用スタッフ】シニアスタッフ_上申書および雇用契約書](#24-2直接雇用スタッフシニアスタッフ_上申書および雇用契約書)
- [24-3【直接雇用スタッフ】要員番号・システム利用申請書](#24-3直接雇用スタッフ要員番号システム利用申請書)
- [24-4【派遣社員】要員番号・システム利用申請書](#24-4派遣社員要員番号システム利用申請書)
- [51.【業務委託等協力会社】要員番号・システム利用申請書](#51業務委託等協力会社要員番号システム利用申請書)

## 24-1【直接雇用スタッフ】スタッフ_上申書および雇用契約書

### 連携方法
電子承認Create!Webフローからの連携は下記の手順で受けるものとする。
* 電子承認Create!Webフローで申請承認が行われる
* 承認された申請が所定の申請の場合、電子承認Create!WebフローがWebhookで案件IDをID管理システムにPost
* ID管理システムが受信した案件IDを元に、電子承認Create!Webフローから申請の内容を取得
* 申請の内容から基準日のみ判定


### ファイルフォーマット
| 項目           | 内容             |
| :------------- | :--------------- |
| 種類           | JSON             |
| ファイル名     | ???.json         |
| 文字コード     | ?                |
| 改行コード     | ?                |
| タイトル行     | なし             |
| エスケープ     | なし             |
| 全量／差分     | 差分             |
| 連携タイミング | 電子承認の承認時 |

### 入力項目
|  No. | 項目名                     | JSON項目名                      | 種別 | 項目説明 | 設定先マスターデータ                                      |
| ---: | :------------------------- | :------------------------------ | :--- | :------- | :-------------------------------------------------------- |
|    1 | 日付                       | APL_DATE-A                      |      |          | -                                                         |
|    2 | 起案者名                   | APL_NAME-A                      |      |          | -                                                         |
|    3 | 部署名                     | APL_GRP-A                       |      |          | -                                                         |
|    4 | 部門承認者名               | APR_NAME_2-A                    |      |          | -                                                         |
|    5 | （雇用対象者）氏名         | TXBKOYOUTAISYOUSYA_SEINENGAPPI  |      |          | **TODO**                                                  |
|    6 | （雇用対象者）生年月日     | TXBKOYOUTAISYOUSYA_SIMEI        |      |          | **TODO**                                                  |
|    7 | （業務内容）               | TXBGYOUMUNAIYOU                 |      |          | -                                                         |
|    8 | （雇用形態）               | -                               |      |          | 就業者情報.雇用形態（スタッフ固定）                       |
|    9 | （雇用期間）　開始         | TXBKOYOUKIKAN_KAISI             |      |          | -                                                         |
|   10 | （雇用期間）　開始         | TXBKOYOUKIKAN_OWARI             |      |          | -                                                         |
|   11 | （雇用する理由）           | TXBKOYOUSURURIYUU               |      |          | -                                                         |
|   12 | （対象者の経歴）           | TXBTAISYOUSYANOKEIREKI          |      |          | -                                                         |
|   13 | 申請区分                   | CMBSINSEIKUBUN                  |      |          | **TODO**選択項目の内容                                    |
|   14 | 日付                       | APL_DATE-A                      |      |          | -                                                         |
|   15 | 所属                       | CMBKEIYAKUSYO_SYOZOKU           |      |          | 就業者所属情報.グループコード                             |
|   16 | 従業員番号　No.            | TXBKEIYAKUSYO_MANNANBA          |      |          | 就業者情報.マンナンバー                                   |
|   17 | 氏名（フリガナ）           | TXBKEIYAKUSYO__SIMEI_KANA       |      |          | 就業者情報.姓カナ <br>就業者情報.名カナ                   |
|   18 | 氏名                       | TXBKEIYAKUSYO_SIMEI             |      |          | 就業者情報.姓 <br>就業者情報.名                           |
|   19 | 部署名                     | APR_GRP_2-A                     |      |          | 就業者所属情報.所属コード(部署名から組織情報でコード変換) |
|   20 | 役職名                     | TXBKEIYAKUSYO_YAKUSYOKUMEI      |      |          | -                                                         |
|   21 | 承認者名氏名               | APR_NAME_2-A                    |      |          | -                                                         |
|   22 | 契約期間（開始）           | TXBKEIYAKUSYO_KEIYAKUKIKAN_KAIS |      |          | 就業者情報.入社日／契約開始日                             |
|   23 | 契約期間（終わり）         | TXBKEIYAKUSYO_KEIYAKUKIKAN_OWAR |      |          | 就業者情報.退職日／契約終了日（契約終了時）               |
|   24 | 契約の更新の有無           | CMBKEIYAKUNOKOUSINNOUMU         |      |          | -                                                         |
|   25 | 契約期間　その他           | TXBKEIYAKUKIKAN_SONOTA          |      |          | -                                                         |
|   26 | 勤務場所（雇入れ直後）     | TXBKINMUBASYO_YATOIIRETYOKUGO   |      |          | -                                                         |
|   27 | 勤務場所（変更の範囲）     | TXBKINMUBASYO_HENKOUNOHANI      |      |          | -                                                         |
|   28 | 業務内容（雇入れ直後）     | TXBGYOUMUNAIYOU_YATOIIRETYOKUGO |      |          | -                                                         |
|   29 | 業務内容（変更の範囲）     | TXBGYOUMUNAIYOU_HENKOUNOHANI    |      |          | -                                                         |
|   30 | 勤務管理者　役職           | TXBKINMUKANRISYA_YAKUSYOKU      |      |          | -                                                         |
|   31 | 勤務管理者　氏名           | APR_NAME_1-A                    |      |          | -                                                         |
|   32 | 就業日　毎週               | TXBSYUUGYOUBI_MAISYUU           |      |          | -                                                         |
|   33 | 就業日　週　日             | TXBSYUUGYOUBI_SYUU_NITI         |      |          | -                                                         |
|   34 | 就業日　（週　時間）       | TXBSYUUGYOUBI_SYUU_JI           |      |          | -                                                         |
|   35 | 就業日　備考               | TXBSYUUGYOUBI_BIKOU             |      |          | -                                                         |
|   36 | 就業時間　時　開始         | TXBSYUUGYOUJIKAN_JI_KAISI       |      |          | -                                                         |
|   37 | 就業時間　分　開始         | TXBSYUUGYOUJIKAN_FUN_KAISI      |      |          | -                                                         |
|   38 | 就業時間　時　終わり       | TXBSYUUGYOUJIKAN_JI_OWARI       |      |          | -                                                         |
|   39 | 就業時間　分　終わり       | TXBSYUUGYOUJIKAN_FUN_OWARI      |      |          | -                                                         |
|   40 | 就業時間　実働　時間       | TXBSYUUGYOUJIKAN_JITUDOU_JI     |      |          | -                                                         |
|   41 | 就業時間　実働　分         | TXBSYUUGYOUJIKAN_JITUDOU_FUN    |      |          | -                                                         |
|   42 | 就業時間　休憩　分         | TXBSYUUGYOUJIKAN_KYUUKEI_FUN    |      |          | -                                                         |
|   43 | 時間外勤務                 | CMBJIKANGAIKINMU                |      |          | -                                                         |
|   44 | 勤務日以外の勤務           | CMBKINMUBIIGAINOKINMU           |      |          | -                                                         |
|   45 | 賃金に関する事項　基準賃金 | TXBKIJUNTINGIN                  |      |          | -                                                         |
|   46 | 社会保険適用　健康保険     | CHKSYAKAIHOKEN_KENKOUHOKEN      |      |          | -                                                         |
|   47 | 社会保険適用　厚生年金     | CHKSYAKAIHOKEN_KOUSEINENKIN     |      |          | -                                                         |
|   48 | 社会保険適用　雇用保険     | CHKSYAKAIHOKEN_KOYOUHOKEN       |      |          | -                                                         |
|   49 | 税区分　甲欄               | CHKZEIKUBUN_KOURAN              |      |          | -                                                         |
|   50 | 税区分　乙欄               | CHKZEIKUBUN_OTURAN              |      |          | -                                                         |
|   51 | 〒                         | TXBKEIYAKSYO_YUUBINBANNGOU      |      |          | -                                                         |
|   52 | 住所                       | TXBKEIYAKSYO_JUUSYO             |      |          | -                                                         |
|   53 | 連絡先TEL：                | TXBKEIYAKSYO_DENWABANGOU        |      |          | -                                                         |
|   54 | 生年月日                   | TXBKEIYAKSYO_SEINENGAPPI        |      |          | 就業者情報.生年月日                                       |
|   55 | 年齢                       | TXBKEIYAKSYO_NENREI             |      |          | -                                                         |
| 番外 | -                          | -                               |      |          | 就業者情報.就業者区分に直接雇用スタッフを設定             |

### 確認事項
* 最初から契約更新無しのことがあるか？




## 24-2【直接雇用スタッフ】シニアスタッフ_上申書および雇用契約書
### ファイルフォーマット
| 項目           | 内容     |
| -------------- | -------- |
| 種類           | JSON     |
| ファイル名     | ---.json |
| 文字コード     | ?        |
| 改行コード     | ?        |
| タイトル行     | なし     |
| エスケープ     | なし     |
| 全量／差分     | 両方     |
| 抽出条件       | ？       |
| 連携タイミング | 内示日？ |

### 入力項目
|  No. | 項目名                        | JSON項目名                      | 種別 | 項目説明 | 設定先マスターデータ                                      |
| ---: | :---------------------------- | :------------------------------ | :--- | :------- | :-------------------------------------------------------- |
|    1 | 日付                          | APL_DATE-A                      |      |          | -                                                         |
|    2 | 部署名                        | APL_GRP-A                       |      |          | -                                                         |
|    3 | 起案者名                      | APL_NAME-A                      |      |          | -                                                         |
|    4 | （雇用対象者）氏名            | TXBKOYOUTAISYOUSYA_SEINENGAPPI  |      |          | -                                                         |
|    5 | （雇用対象者）生年月日        | TXBKOYOUTAISYOUSYA_SIMEI        |      |          | **TODO**氏名と生年月日が逆？                              |
|    6 | （業務内容）                  | TXBGYOUMUNAIYOU                 |      |          | **TODO**氏名と生年月日が逆？                              |
|    7 | （雇用期間）　開始            | TXBKOYOUKIKAN_KAISI             |      |          | -                                                         |
|    8 | （雇用期間）　終わり          | TXBKOYOUKIKAN_OWARI             |      |          | 就業者情報.雇用形態（シニアスタッフ固定）                 |
|    9 | （雇用する理由）              | TXBKOYOUSURURIYUU               |      |          | -                                                         |
|   10 | （対象者の経歴）              | TXBTAISYOUSYANOKEIREKI          |      |          | -                                                         |
|   11 | 申請区分                      | CMBSINSEIKUBUN                  |      |          | **TODO**選択項目の内容                                    |
|   12 | 所属                          | CMBKEIYAKUSYO_SYOZOKU           |      |          | -                                                         |
|   13 | 従業員番号　No.               | TXBKEIYAKUSYO_MANNANBA          |      |          | 就業者情報.マンナンバー                                   |
|   14 | 氏名（フリガナ）              | TXBKEIYAKUSYO__SIMEI_KANA       |      |          | 就業者情報.姓カナ,就業者情報.名カナ**TODO**外国人名の場合 |
|   15 | 氏名                          | TXBKEIYAKUSYO_SIMEI             |      |          | 就業者情報.姓,就業者情報.名**TODO**外国人名の場合         |
|   16 | 部署名                        | APR_GRP_2-A                     |      |          | 就業者所属情報.所属コード(部署名から組織情報でコード変換) |
|   17 | 役職名                        | TXBKEIYAKUSYO_YAKUSYOKUMEI      |      |          | -                                                         |
|   18 | 承認者名氏名                  | APR_NAME_2-A                    |      |          | -                                                         |
|   19 | 契約期間（開始）              | TXBKEIYAKUSYO_KEIYAKUKIKAN_KAIS |      |          | 就業者情報.入社日／契約開始日                             |
|   20 | 契約期間（終わり）            | TXBKEIYAKUSYO_KEIYAKUKIKAN_OWAR |      |          | 就業者情報.退職日／契約終了日（契約終了時）               |
|   21 | 契約の更新の有無              | CMBKEIYAKUNOKOUSINNOUMU         |      |          | -                                                         |
|   22 | 契約期間　その他              | TXBKEIYAKUKIKAN_SONOTA          |      |          | -                                                         |
|   23 | 労働条件変更の有無            | CMBROUDOUJOUKENHENKOUNOUMU      |      |          | -                                                         |
|   24 | 勤務場所（雇入れ直後）        | TXBKINMUBASYO_YATOIIRETYOKUGO   |      |          | -                                                         |
|   25 | 勤務場所（変更の範囲）        | TXBKINMUBASYO_HENKOUNOHANI      |      |          | -                                                         |
|   26 | 業務内容（雇入れ直後）        | TXBGYOUMUNAIYOU_YATOIIRETYOKUGO |      |          | -                                                         |
|   27 | 業務内容（変更の範囲）        | TXBGYOUMUNAIYOU_HENKOUNOHANI    |      |          | -                                                         |
|   28 | 勤務管理者　役職              | TXBKINMUKANRISYA_YAKUSYOKU      |      |          | -                                                         |
|   29 | 勤務管理者　氏名              | APR_NAME_1-A                    |      |          | -                                                         |
|   30 | 就業日　毎週                  | TXBSYUUGYOUBI_MAISYUU           |      |          | -                                                         |
|   31 | 就業日　週　日                | TXBSYUUGYOUBI_SYUU_NITI         |      |          | -                                                         |
|   32 | 就業日　（週　時間）          | TXBSYUUGYOUBI_SYUU_JI           |      |          | -                                                         |
|   33 | 就業日　備考                  | TXBSYUUGYOUBI_BIKOU             |      |          | -                                                         |
|   34 | 就業時間　時　開始            | TXBSYUUGYOUJIKAN_JI_KAISI       |      |          | -                                                         |
|   35 | 就業時間　分　開始            | TXBSYUUGYOUJIKAN_FUN_KAISI      |      |          | -                                                         |
|   36 | 就業時間　時　終わり          | TXBSYUUGYOUJIKAN_JI_OWARI       |      |          | -                                                         |
|   37 | 就業時間　分　終わり          | TXBSYUUGYOUJIKAN_FUN_OWARI      |      |          | -                                                         |
|   38 | 就業時間　実働　時間          | TXBSYUUGYOUJIKAN_JITUDOU_JI     |      |          | -                                                         |
|   39 | 就業時間　実働　分            | TXBSYUUGYOUJIKAN_JITUDOU_FUN    |      |          | -                                                         |
|   40 | 就業時間　休憩　分            | TXBSYUUGYOUJIKAN_KYUUKEI_FUN    |      |          | -                                                         |
|   41 | 時間外勤務                    | CMBJIKANGAIKINMU                |      |          | -                                                         |
|   42 | 勤務日以外の勤務              | CMBKINMUBIIGAINOKINMU           |      |          | -                                                         |
|   43 | 賃金に関する事項　1．基準賃金 | TXBKIJUNTINGIN                  |      |          | -                                                         |
|   44 | 社会保険　健康保険            | CHKSYAKAIHOKEN_KENKOUHOKEN      |      |          | -                                                         |
|   45 | 社会保険　厚生年金            | CHKSYAKAIHOKEN_KOUSEINENKIN     |      |          | -                                                         |
|   46 | 社会保険　雇用保険            | CHKSYAKAIHOKEN_KOYOUHOKEN       |      |          | -                                                         |
|   47 | 税区分　甲欄                  | CHKZEIKUBUN_KOURAN              |      |          | -                                                         |
|   48 | 税区分　乙欄                  | CHKZEIKUBUN_OTURAN              |      |          | -                                                         |
|   49 | 〒                            | TXBKEIYAKSYO_YUUBINBANNGOU      |      |          | -                                                         |
|   50 | 住所                          | TXBKEIYAKSYO_JUUSYO             |      |          | -                                                         |
|   51 | 連絡先TEL：                   | TXBKEIYAKSYO_DENWABANGOU        |      |          | -                                                         |
|   52 | 生年月日                      | TXBKEIYAKSYO_SEINENGAPPI        |      |          | -                                                         |
|   53 | 年齢                          | TXBKEIYAKSYO_NENREI             |      |          | -                                                         |
| 番外 | -                             | -                               |      |          | 就業者情報.就業者区分に直接雇用シニアスタッフを設定       |


## 24-3【直接雇用スタッフ】要員番号・システム利用申請書
### ファイルフォーマット
| 項目           | 内容     |
| -------------- | -------- |
| 種類           | JSON     |
| ファイル名     | ---.json |
| 文字コード     | ?        |
| 改行コード     | ?        |
| タイトル行     | なし     |
| エスケープ     | なし     |
| 全量／差分     | 両方     |
| 抽出条件       | ？       |
| 連携タイミング | 内示日？ |

### 入力項目
|  No | 項目名                           | JSON項目名                   | 種別 | 項目説明   | 設定先マスターデータ                              |
| --: | :------------------------------- | :--------------------------- | :--- | :--------- | :------------------------------------------------ |
|   1 | 日付                             | APL_DATE-A                   |      |            |                                                   |
|   2 | 申請部署                         | APL_GRP-A                    |      |            | -                                                 |
|   3 | 社員番号                         | APL_EMP_ID-A                 |      |            | -                                                 |
|   4 | 氏名                             | APL_NAME-A                   |      |            | -                                                 |
|   5 | 連絡先（内線番号）               | TXBRENRAKUSAKI               |      |            | -                                                 |
|   6 | 申請区分                         | CMBSINSEIKUBUN               |      |            |                                                   |
|   7 | 上申書申請日                     | TXBJOUSINSYO_SINSEIBI        |      |            |                                                   |
|   8 | 要員情報　氏名                   | TXBYOUINJOUHOU_SIMEI         | 文字 |            | 就業者情報.姓、就業者情報.名                      |
|   9 | 要員情報　カナ                   | TXBYOUINJOUHOU_KANA          | 文字 |            | 就業者情報.姓カナ、就業者情報.名カナ              |
|  10 | 要員情報　雇い入れ区分           | CMBYATOIIREKUBUN             |      |            |                                                   |
|  11 | 要員情報　所属                   | CMBYOUINJOUHOU_SYOZOKU       | 文字 |            | 就業者所属情報.グループ名称                       |
|  12 | 要員情報　旧マンナンバー         | TXBKYUUMANNANBA              | 数値 |            | 就業者情報.旧マンナンバー                         |
|  13 | 要員情報　生年月日               | TXBYOUINJOUHOU_SEINENGAPPI   | 日付 | yyyy/mm/dd | 就業者情報.生年月日                               |
|  14 | 要員情報　契約期間　開始         | TXBYOUINJOUHOU_KEIYAKU_KAISI |      |            | 就業者情報.入社日/契約開始日                      |
|  15 | 要員情報　契約期間　終了         | TXBYOUINJOUHOU_KEIYAKU_OWARI |      |            | 就業者情報.退職日/契約終了日                      |
|  16 | 要員情報　マンナンバー           | TXBYOUINJOUHOU_MANNANBA      |      |            | 就業者情報.マンナンバー                           |
|  17 | ジョブカン勤怠業務グループ       | CMBJOBUKANKINTAI_GYOUMUGROUP |      |            |                                                   |
|  18 | 利用区分　Microsoft365アカウント | CHKMICROSOFT365AKAUNTO       | bool |            | 就業者情報.システム利用・M365                     |
|  19 | 利用区分　AD登録（NTAD）         | CHKADTOUROKU                 | bool |            | 就業者情報.システム利用・NTAD                     |
|  20 | 利用区分　NTポータル             | CHKNTPORTAL                  | bool |            | 就業者情報.システム利用・NTポータル(スタッフ権限) |
|  21 | 利用区分　EMILY利用              | CHKEMILYRIYOU                | bool |            | 就業者情報.システム利用・EMILY                    |
|  22 | 利用区分　ジョブカン経費精算利用 | CHKJOBUKANKEIHISEISANRIYOU   | bool |            | 就業者情報.システム利用・ジョブカン経費精算       |
|  23 | カード区分　スタッフ証           | CHKSUTAFFSYOU                |      |            |                                                   |
|  24 | カード区分　第三共同ビル入館     | CHKDAISANKYOUDOUBIRUNYUUKAN  |      |            | 就業者情報.カード区分・第三共同ビル入館           |
|  25 | カード区分　第七共同ビル入館     | CHKDAINANAKYOUDOUBIRUNYUUKAN |      |            | 就業者情報.カード区分・第七共同ビル入館           |
|  26 | 申請時連絡事項                   | TXBSINSEIJIRENRAKUJIKOU      |      |            |                                                   |
|  27 | メールアドレス                   | TXBMEIRUADORESU              |      |            | 就業者情報.メールアドレス                         |
|  28 | 備考                             | TXBBIKOU                     |      |            |                                                   |
|  29 | 業務グループ                     | GYOUMUGROUP                  |      | 未使用     |                                                   |
|  30 | 部署                             | BUSYO                        |      | 未使用     |                                                   |

---
## 24-4【派遣社員】要員番号・システム利用申請書
### ファイルフォーマット
| 項目           | 内容     |
| -------------- | -------- |
| 種類           | JSON     |
| ファイル名     | ---.json |
| 文字コード     | ?        |
| 改行コード     | ?        |
| タイトル行     | なし     |
| エスケープ     | なし     |
| 全量／差分     | 両方     |
| 抽出条件       | ？       |
| 連携タイミング | 内示日   |

### 入力項目
|  No | 項目名                                       | JSON項目名                   | 種別 | 項目説明 | 設定先マスターデータ                                 |
| --: | :------------------------------------------- | :--------------------------- | :--- | :------- | :--------------------------------------------------- |
|   1 | 日付                                         | APL_DATE-A                   |      |          |                                                      |
|   2 | 申請部署                                     | APL_GRP-A                    |      |          | 就業者所属情報.所属コード（要員情報 所属部署を優先） |
|   3 | 社員番号                                     | APL_EMP_ID-A                 |      |          | -                                                    |
|   4 | 氏名                                         | APL_NAME-A                   |      |          | -                                                    |
|   5 | 連絡先（内線番号）                           | TXBRENRAKUSAKI               |      |          | -                                                    |
|   6 | 申請区分                                     | CMBSINSEIKUBUN               |      |          |                                                      |
|   7 | 派遣先通知書添付確認                         | CHKHAKENSAKITUUTISYO         |      |          |                                                      |
|   8 | 要員情報　氏名                               | TXBSIMEI                     | 文字 |          | 就業者情報.姓、就業者情報.名                         |
|   9 | 要員情報　カナ                               | TXBKANA                      | 文字 |          | 就業者情報.姓カナ、就業者情報.名カナ                 |
|  10 | 要員情報　所属会社（派遣元）                 | TXBSYOZOKUKAISYA             | 文字 |          | 就業者情報.派遣／所属会社名                          |
|  11 | 要員情報　所属部署                           | CMBYOUINJOUHOU_BUSYO         | 文字 |          | 就業者所属情報.所属コード                            |
|  12 | 要員情報　勤務先                             | TXBKINMUSAKI                 |      |          |                                                      |
|  13 | 要員情報　契約期間　開始                     | TXBKEIYAKUKIKAN_KAISI        |      |          | 就業者情報.入社日/契約開始日                         |
|  14 | 要員情報　契約期間　終了                     | TXBKEIYAKUKIKAN_OWARI        |      |          | 就業者情報.退職日/契約終了日                         |
|  15 | 要員情報　マンナンバー                       | TXBMANNANBA                  |      |          | 就業者情報.マンナンバー                              |
|  16 | 利用システム区分　Microsoft365アカウント     | CHKMICROSOFTAKAUNTO          | bool |          | 就業者情報.システム利用・M365                        |
|  17 | 利用システム区分　AD登録（NTAD）             | CHKADTOUROKU                 | bool |          | 就業者情報.システム利用・NTAD                        |
|  18 | 利用システム区分　NTポータル（スタッフ権限） | CHKNTPORTAL                  | bool |          | 就業者情報.システム利用・NTポータル(スタッフ権限)    |
|  19 | 利用システム区分　EMILY利用（要NTAD）        | CHKEMILYRIYOU                | bool |          | 就業者情報.システム利用・EMILY                       |
|  10 | 利用システム区分　ジョブカン経費精算利用     | CHKJOBUKANKEIHISEISANRIYOU   | bool |          | 就業者情報.システム利用・ジョブカン経費精算          |
|  21 | カード区分　NHK入館証                        | CHKNHKNYUUKANKYOKASYOU       |      |          | 就業者情報.カード区分・NHK入館許可証                 |
|  22 | カード区分　関連団体ADカード                 | CHKKANRENDANTAIADKADO        |      |          | 就業者情報.カード区分・関連団体ADカード              |
|  23 | カード区分　第三共同ビル入館                 | CHKDAISANKYOUDOUBIRUNYUUKAN  |      |          | 就業者情報.カード区分・第三共同ビル入館              |
|  24 | カード区分　第七共同ビル入館                 | CHKDAINANAKYOUDOUBIRUNYUUKAN |      |          | 就業者情報.カード区分・第七共同ビル入館              |
|  25 | 申請時連絡事項                               | TXBSINSEIJIRENRAKUJIKOU      |      |          |                                                      |
|  26 | メールアドレス                               | TXBMERUADORESU               |      |          |                                                      |
|  27 | 備考                                         | TXBBIKOU                     | 文字 |          |                                                      |
|  28 | 未使用                                       | BUSYO                        |      |          |                                                      |

---
## 51.【業務委託等協力会社】要員番号・システム利用申請書
### ファイルフォーマット
| 項目           | 内容     |
| -------------- | -------- |
| 種類           | JSON     |
| ファイル名     | ---.json |
| 文字コード     | ?        |
| 改行コード     | ?        |
| タイトル行     | なし     |
| エスケープ     | なし     |
| 全量／差分     | 両方     |
| 抽出条件       | ？       |
| 連携タイミング | 内示日   |

### 入力項目
|  No | 項目名                                       | JSON項目名                      | 種別 | 項目説明 | 設定先マスターデータ                                   |
| --: | :------------------------------------------- | :------------------------------ | :--- | :------- | :----------------------------------------------------- |
|   1 | 日付                                         | APL_DATE-A                      |      |          |                                                        |
|   2 | 申請部署                                     | APL_GRP-A                       |      |          | 就業者所属情報.グループ名称（要員情報 管理部署を優先） |
|   3 | 社員番号                                     | APL_EMP_ID-A                    |      |          | -                                                      |
|   4 | 氏名                                         | APL_NAME-A                      |      |          | -                                                      |
|   5 | 連絡先（内線番号）                           | TXBRENRAKUSAKI                  |      |          | -                                                      |
|   6 | 申請区分                                     | CMBSINSEIKUBUN                  |      |          |                                                        |
|   7 | 要員情報　氏名                               | TXBYOUINJOUHOU_SIMEI_01         |      |          | 就業者情報.姓、就業者情報.名                           |
|   8 | 要員情報　フリガナ                           | TXBYOUINJOUHOU_FURIGANA_01      | 文字 |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|   9 | 要員情報　所属会社名                         | TXBYOUINJOUHOU_SYOZOKUKAISYA_01 | 文字 |          | 就業者情報.派遣／所属会社名                            |
|  10 | 要員情報　管理部署                           | CMBKANRIBUSYO_01                | 文字 |          |                                                        |
|  11 | 要員情報　契約期間　開始                     | TXBYOUINJOUHOU_KEIYAKU_KAISI_01 | 文字 |          | 就業者情報.入社日/契約開始日                           |
|  12 | 要員情報　契約期間　終了                     | TXBYOUINJOUHOU_KEIYAKU_OWARI_01 |      |          | 就業者情報.退職日/契約終了日                           |
|  13 | 要員情報　マンナンバー                       | TXBYOUINJOUHOU_MANNANBA_01      |      |          | 就業者情報.マンナンバー                                |
|  14 | 利用システム区分　AD登録（NTAD）             | CHKADTOUROKU                    |      |          | 就業者情報.システム利用・NTAD                          |
|  15 | 利用システム区分　EMILY利用                  | CHKEMILYRIYOU                   | bool |          | 就業者情報.システム利用・EMILY                         |
|  16 | 利用システム区分　Microsoft365アカウント     | CHKMICROSOFTAKAUNTO             | bool |          | 就業者情報.システム利用・M365                          |
|  17 | 利用システム区分　EMILY社員マスタ登録        | CHKEMILYSYAINMASUTATOUROKU      | bool |          | 就業者情報.システム利用・EMILY社員マスタ登録           |
|  18 | カード区分　NHK入館証またはNHK統合認証利用有 | CHKNHKNYUUKAN_OR_TOUGOUNINNSYOU | bool |          | 就業者情報.カード区分・NHK入館許可証                   |
|  19 | カード区分　関連団体ADカード（NHK入館なし）  | CHKKANRENDANTAIAKADO            | bool |          | 就業者情報.カード区分・関連団体ADカード                |
|  20 | カード区分　第三共同ビル入館                 | CHKDAISANKYOUDOUBILUNYUUKAN     | bool |          | 就業者情報.カード区分・第三共同ビル入館                |
|  21 | カード区分　第七共同ビル入館                 | CHKDAINANAKYOUDOUBILUNYUUKAN    |      |          | 就業者情報.カード区分・第七共同ビル入館                |
|  22 | 申請連絡事項                                 | TXBSINSEIJIRENRAKUJIKOU         |      |          |                                                        |
|  23 | 総務処理メモ                                 | TXBSOUMUSYORIMEMO               |      |          |                                                        |
|  24 | メールアドレス                               | TXBMELUADRESU                   |      |          |                                                        |
|  25 | IT企画処理メモ                               | TXBITKIKAKUSYORIMEMO            |      |          |                                                        |
|  26 | 要員情報　２　氏名                           | TXBYOUINJOUHOU_SIMEI_02         |      |          | 就業者情報.姓、就業者情報.名                           |
|  27 | 要員情報　２　フリガナ                       | TXBYOUINJOUHOU_FURIGANA_02      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  28 | 要員情報　２　マンナンバー                   | TXBYOUINJOUHOU_MANNANBA_02      |      |          | 就業者情報.マンナンバー                                |
|  29 | 要員情報　３　氏名                           | TXBYOUINJOUHOU_SIMEI_03         |      |          | 就業者情報.姓、就業者情報.名                           |
|  30 | 要員情報　３　フリガナ                       | TXBYOUINJOUHOU_FURIGANA_03      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  31 | 要員情報　３　マンナンバー                   | TXBYOUINJOUHOU_MANNANBA_03      |      |          | 就業者情報.マンナンバー                                |
|  32 | 要員情報　４　氏名                           | TXBYOUINJOUHOU_SIMEI_04         |      |          | 就業者情報.姓、就業者情報.名                           |
|  33 | 要員情報　４　フリガナ                       | TXBYOUINJOUHOU_FURIGANA_04      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  34 | 要員情報　４　マンナンバー                   | TXBYOUINJOUHOU_MANNANBA_04      |      |          | 就業者情報.マンナンバー                                |
|  35 | 要員情報　５　氏名                           | TXBYOUINJOUHOU_SIMEI_05         |      |          | 就業者情報.姓、就業者情報.名                           |
|  36 | 要員情報　５　フリガナ                       | TXBYOUINJOUHOU_FURIGANA_05      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  37 | 要員情報　５　マンナンバー                   | TXBYOUINJOUHOU_MANNANBA_05      |      |          | 就業者情報.マンナンバー                                |
|  38 | 要員情報　６　氏名                           | TXBYOUINJOUHOU_SIMEI_06         |      |          | 就業者情報.姓、就業者情報.名                           |
|  39 | 要員情報　６　フリガナ                       | TXBYOUINJOUHOU_FURIGANA_06      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  40 | 要員情報　６　マンナンバー                   | TXBYOUINJOUHOU_MANNANBA_06      |      |          | 就業者情報.マンナンバー                                |
|  41 | 要員情報　７　氏名                           | TXBYOUINJOUHOU_SIMEI_07         |      |          | 就業者情報.姓、就業者情報.名                           |
|  42 | 要員情報　７　フリガナ                       | TXBYOUINJOUHOU_FURIGANA_07      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  43 | 要員情報　７　マンナンバー                   | TXBYOUINJOUHOU_MANNANBA_07      |      |          | 就業者情報.マンナンバー                                |
|  44 | 要員情報　８　氏名                           | TXBYOUINJOUHOU_SIMEI_08         |      |          | 就業者情報.姓、就業者情報.名                           |
|  45 | 要員情報　８　フリガナ                       | TXBYOUINJOUHOU_FURIGANA_08      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  46 | 要員情報　８　マンナンバー                   | TXBYOUINJOUHOU_MANNANBA_08      |      |          | 就業者情報.マンナンバー                                |
|  47 | 要員情報　９　氏名                           | TXBYOUINJOUHOU_SIMEI_09         |      |          | 就業者情報.姓、就業者情報.名                           |
|  48 | 要員情報　９　フリガナ                       | TXBYOUINJOUHOU_FURIGANA_09      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  49 | 要員情報　９　マンナンバー                   | TXBYOUINJOUHOU_MANNANBA_09      |      |          | 就業者情報.マンナンバー                                |
|  50 | 要員情報　１０　氏名                         | TXBYOUINJOUHOU_SIMEI_10         |      |          | 就業者情報.姓、就業者情報.名                           |
|  51 | 要員情報　１０　フリガナ                     | TXBYOUINJOUHOU_FURIGANA_10      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  52 | 要員情報　１０　マンナンバー                 | TXBYOUINJOUHOU_MANNANBA_10      |      |          | 就業者情報.マンナンバー                                |
|  53 | 要員情報　１１　氏名                         | TXBYOUINJOUHOU_SIMEI_11         |      |          | 就業者情報.姓、就業者情報.名                           |
|  54 | 要員情報　１１　フリガナ                     | TXBYOUINJOUHOU_FURIGANA_11      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  55 | 要員情報　１１　マンナンバー                 | TXBYOUINJOUHOU_MANNANBA_11      |      |          | 就業者情報.マンナンバー                                |
|  56 | 要員情報　１２　氏名                         | TXBYOUINJOUHOU_SIMEI_12         |      |          | 就業者情報.姓、就業者情報.名                           |
|  57 | 要員情報　１２　フリガナ                     | TXBYOUINJOUHOU_FURIGANA_12      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  58 | 要員情報　１２　マンナンバー                 | TXBYOUINJOUHOU_MANNANBA_12      |      |          | 就業者情報.マンナンバー                                |
|  59 | 要員情報　１３　氏名                         | TXBYOUINJOUHOU_SIMEI_13         |      |          | 就業者情報.姓、就業者情報.名                           |
|  60 | 要員情報　１３　フリガナ                     | TXBYOUINJOUHOU_FURIGANA_13      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  61 | 要員情報　１３　マンナンバー                 | TXBYOUINJOUHOU_MANNANBA_13      |      |          | 就業者情報.マンナンバー                                |
|  62 | 要員情報　１４　氏名                         | TXBYOUINJOUHOU_SIMEI_14         |      |          | 就業者情報.姓、就業者情報.名                           |
|  63 | 要員情報　１４　フリガナ                     | TXBYOUINJOUHOU_FURIGANA_14      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  64 | 要員情報　１４　マンナンバー                 | TXBYOUINJOUHOU_MANNANBA_14      |      |          | 就業者情報.マンナンバー                                |
|  65 | 要員情報　１５　氏名                         | TXBYOUINJOUHOU_SIMEI_15         |      |          | 就業者情報.姓、就業者情報.名                           |
|  66 | 要員情報　１５　フリガナ                     | TXBYOUINJOUHOU_FURIGANA_15      |      |          | 就業者情報.姓カナ、就業者情報.名カナ                   |
|  67 | 要員情報　１５　マンナンバー                 | TXBYOUINJOUHOU_MANNANBA_15      |      |          | 就業者情報.マンナンバー                                |

### 確認事項
* NHK総合認証利用とは？
* 要員ごとのメールアドレスがない