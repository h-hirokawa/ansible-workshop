# Ansibleトレーニング

本トレーニングではAnsible EngineとAnsible Towerの実装ハンズオンを通じて、Ansibleの基本機能を学んでいきます。

## 目次

* Ansible Engine編
  AnsibleによるPlaybookの実装とRoleについて学ぶ
  * [演習 1 - Ansibleのインストールとインベントリー定義](./engine/ex1.html)
  * [演習 2 - ad-hocコマンドの実行](./engine/ex2.html)
  * [演習 3  - 初めてのplaybook作成](./engine/ex3.html)
  * [演習 4 - 変数、ループ、ハンドラを使う](./engine/ex4.html)
  * [演習 5 - Role：Playbookを再利用可能する](./engine/ex5.html)
* Ansible Tower編
  TowerのデプロイとTowerからのPlaybook実行、API経由の操作の基本を学ぶ
  * 

## 前提

* RHEL7サーバー × 2台
  * 内訳
    * Ansibleコントロールノード × 1台
    * Ansible管理対象ノード × 1台
  * サブスクリプションがアタッチ済み
  * インターネット接続が可能
  * `sudo`による管理者権限昇格が可能、もしくはログインユーザーが`root`
* Ansible Towerのworkshopライセンス
  * https://www.ansible.com/workshop-license から発行可能
* SSH経由でのLinux操作の基礎知識
* vi を使ったファイル操作の基礎知識


