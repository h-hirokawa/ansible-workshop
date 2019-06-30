# 演習2 - Ansible Towerのコンフィグレーション

[前に戻る](./ex1.html)

------

この演習を実施することにより、Ansible Towerを用いてPlaybookが実行できるようになります。
ここではブラウザの言語設定が日本語に設定されている状態を前提として操作を解説しています。

## Ansible Towerのコンフィグレーション

今回のトレーニングでは、Ansible Towerの下記の基本機能の動きを確認していきます。

* **認証情報(Credentials)**: 対象ノードやSCM（Git, SVN…）などに接続するための認証情報を設定
* **プロジェクト**: playbookが含まれるSCM上のリポジトリを登録
* **インベントリー**: Ansible EngineにおけるInventoryと同様
  * 情報はTower上で直接管理、Gitリポジトリや外部DBとの同期などが可能
* **ジョブテンプレート**: インベントリー、プロジェクト中の使用playbook、認証情報を組み合わせて「どこに何を実行するか」を決めた状態で保存し、Towerから実行可能な状態にしたもの

## Ansible Towerへのログインとライセンスキーのインストール

### Step 1:

以下の認証情報を用いてAnsible Towerへログインします。

ユーザー名:`admin`

パスワード:`ansibleWS`(もしくは演習 1でinventoryファイルへ記入したパスワード)

ログインが完了すると、ライセンスのリクエストか、ライセンスファイルの投入を求められるページが表示されます。

### Step 3:

ライセンスファイルの「参照」をクリックし、コンピュータ上にあるWorkshop用ライセンスファイルを選択します。

### Step 4:

「**使用許諾契約書に同意します。**」の確認へチェックを入れます。

「TRACKING AND ANALYTICS」以下の項目はRed HatへのTower使用状況送信を許可するかどうかの選択です。必要に応じてチェックを外してください。

### Step 5:

「送信」ボタンをクリックし、ライセンス登録は完了です。
登録が完了すると、Ansible Towerのログイン後トップ画面であるダッシュボードに自動遷移します。

## 認証情報の作成

認証情報は、Ansible Towerがジョブなどを実行する際に利用されます。サーバに対するジョブやインベントリー情報の同期、SCMとのプロジェクト同期を実行する際などに利用されます。

[認証情報のタイプ](http://docs.ansible.com/ansible-tower/latest/html/userguide/credentials.html#credential-types) はマシン(サーバログイン情報)や、SCM、AWSなどのクラウドアカウントなど様々がありますが、このワークショップでは操作対象サーバにSSHログインするためのマシン認証情報を利用します。

### Step 1:

画面左の一覧から「認証情報」をクリックします。

### Step 2:

「+」（新規追加ボタン）をクリックします。

### Step 3:

以下の表の通りに入力し、ワークショップの中で利用する認証情報を登録します。
（これ以降、記載のない項目は未定義またはデフォルト値のままでOKです）

項目 | 値
-----|---------------------------
名前 | トレーニング用マシン認証情報   
組織|Default
認証情報タイプ|Machine
ユーザー名| <<サーバーログインユーザー名>> 
パスワード| <<サーバーログインパスワード>> 
権限昇格方法|sudo

### Step 4:

「保存」をクリックします。 

## プロジェクトの作成

プロジェクトは、Ansible Tower内で使用するplaybookを含んだSCM（Git, SVNなど）で管理されたリポジトリへの参照となります。
Tower自体はplaybook作成/編集機能やリポジトリのバージョン管理機能を持ちませんので、Towerはここで登録したリポジトリをコピー(Clone)して使用することになります。

### Step 1:

画面左の一覧から「プロジェクト」をクリックします。

### Step 2:

「+」（新規追加ボタン）をクリックします。

### Step 3:

以下の値を利用して新規プロジェクトを作成します。

項目 | 値
-----|------------------------
名前 |ワークショップ用プロジェクト
組織|Default
SCMタイプ|Git
SCM URL| <<Gitlab上ワークショップ用リポジトリURL>> 
SCM BRANCH| 
SCM UPDATE OPTIONS(SCM更新オプション)

- [x] Clean(クリーニング) 
- [x] Delete on Update(更新時の削除)
- [x] Update on Launch(起動時の更新)

![Defining a Project](at_project_detail.png)

### Step 4:

SAVEをクリックします。 ![Save button](at_save.png)

## Inventory(インベントリ) の作成

インベントリとは、Jobが実行可能なホストのコレクションです。
インベントリはグループごとに分離され、グループ内にJobが実行されるホストが含まれることになります。
グループはAnsible Towerでホスト名を手動で入力したり、Ansible Towerがサポートしているクラウド・プロバイダーから入手します。

Inventoryは`tower-manage`コマンドを使ってAnsible Towerへインポートすることも可能で、今回のワークショップではこの方法でInventoryを追加します。


### Step 1:

INVENTORIESをクリックします。

### Step 2:

＋ADD(＋追加)をクリック、Inventory(インベントリー)を選択します ![Add button](at_add.png)

### Step 3:

以下の値を利用して、新規Inventoryを作成します。

項目 | 値
-----|--------------------------
NAME(名前) |Ansible Workshop Inventory
DESCRIPTION(説明)|workshop hosts
ORGANIZATION(組織)|Default

![Create an Inventory](at_inv_create.png)

### Step 4:

SAVE(保存)をクリックします。 ![Save button](at_save.png)

### Step 5:

SSHを利用し、Ansibleコントロールノードへログインします。

`tower-manage`　コマンドを利用して既存のインベントリファイルをAnsible Towerへインポートします。（以下のコマンドの_<location of you inventory>_をAnsibleEngineの演習で利用していたInventoryファイルのパスへ置き換えてください。)

```
sudo tower-manage inventory_import --source=<location of you inventory> --inventory-name="Ansible Workshop Inventory"
```

以下のような出力になるはずです:

![Importing an inventory with tower-manage](at_tm_stdout.png)

Ansible Towerのインベントリを確認してみてください。
先ほど作成したインベントリ"Ansible Workshop Inventory"内に、グループWebと、その中にノードが登録されていることが確認できるはずです。

![Inventory with Groups](at_inv_group.png)

### 結果

ここまでで、Ansible Towerの基本的な構成を終えることができました。
次の演習ではjob templateの作成と実行に焦点を当て、Ansible Towerがどのように機能するかを実際に見ていきます。

------

[次に進む](./ex3.html)