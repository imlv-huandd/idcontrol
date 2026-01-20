# RPAによる連携先システムへのデータ連携について

## 主な処理方法：
* BOXへのファイルの配置

### BOXへのファイルの配置

#### 概要
この方法では当システムで作成した連携データファイルを、BOXに保存します。その後、RPA（ロボットによる自動処理プログラム）が自動的にファイルを読み込み、連携先システムへのデータ登録を行います。

#### 処理の流れ
1. **連携先の保存先フォルダを確認**  
   連携先システムの設定から、ファイルを保存するBOX上のフォルダの場所を取得します。

2. **連携ファイルをBOXAPIを使用して指定フォルダに保存**  
   作成した連携データファイルを、BOXのAPIを用いて指定されたフォルダに配置します。この時点で当システム側の連携処理は完了となります。

3. **RPAが自動起動**  
   BOXに新しいファイルが配置されたことを検知して、RPA（ロボット）が自動的に起動します。

4. **RPAによるデータ登録**  
   起動したRPAが、配置されたファイルを読み込み、連携先システムに対して各種データの登録処理を自動実行します。

#### 処理フロー図
```mermaid
sequenceDiagram
    participant System as ID管理システム
    participant SP as Sharepoint<br/>(クラウド共有フォルダ)
    participant RPA as RPA<br/>(自動処理ロボット)
    participant Target as 連携先システム

    Note over System: 連携データファイルを作成
    System->>System: 連携先の保存先フォルダ情報を取得
    System->>SP: Sharepointに接続
    System->>SP: 指定フォルダにファイルを配置
    Note over System: 当システムの連携処理完了

    SP-->>RPA: ファイル配置を検知
    Note over RPA: 自動起動
    RPA->>SP: ファイルを読み込み
    RPA->>Target: データを登録
    RPA->>Target: 各種処理を実行
    Note over RPA,Target: 連携先への登録完了
```
